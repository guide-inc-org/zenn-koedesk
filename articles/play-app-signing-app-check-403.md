---
title: "Play配布版だけFirebase App Checkが403になる — 署名SHA-256を3点突合で特定する切り分け手順"
emoji: "🔏"
type: "tech"
topics: ["android", "firebase", "appcheck", "playintegrity", "googleplay"]
published: true
---

## はじめに

先に結論を書きます。

Google Play 配布版のアプリで**だけ** Firebase App Check の `getToken()` が失敗する場合、最有力容疑者は「**Firebase に登録した SHA-256 が Upload key のもので、Play App Signing の App signing key が未登録**」です。Play Integrity の verdict は Google が再署名した後の **App signing key 側**の証明書で発行されるため、Upload key しか登録されていないと Firebase の Token Exchange が「unknown app」として 403 を返し続けます。**両方登録**すれば直ります。

我々が担当しているアプリで実際にこの事象が起き、調査でかなりハマりました。結局原因は上の1行なんですが、そこに辿り着くまでに容疑者が複数いて（アプリ未登録・App ID 不一致・プロジェクト番号不一致・verdict 要件）、しかもログは「成功しているように見える」出方をする。検索すると同じところでハマっている人がちらほら見えたので、共有です。「登録しましょう」という解決ガイドは既にあるので（[KINTO Tech Blog の登録手順ガイド](https://blog.kinto-technologies.com/posts/2025-10-15-fixFirebaseFcmInstallationsPlaySigningSha/)、[note の App Check 事例](https://note.com/ar_vr_mr/n/ne42160255d9a)）、この記事はその手前、**「本当にそれが原因なのか」を証拠付きで突き止めるまでの経緯と切り分け手順**を書きます。

## 何が起きたか

事象はこうでした（アプリの詳細は伏せます。値はすべてダミーに置き換えています）。

- 開発ビルド（Upload key 署名のまま直インストール）→ App Check 正常
- **Play ストア経由で入れた同じアプリ** → サーバー側で「App Check ヘッダが無い」エラー。クライアントを見ると `getToken()` が null

最初にやることは当然 logcat です。そして、ここで混乱しました。

```
Integrity key attestation record generated successfully
```

**Play Integrity のトークン取得は成功しているように見える**んです。トークンは取れているのにヘッダに乗らない？

ハマりから抜ける最初の鍵はここでした。App Check (Play Integrity provider) のトークン取得は**2段階**あります。

1. Play Integrity API から integrity token を取得 ← logcat で見えていたのはここ
2. その token を Firebase の Exchange API に渡して App Check token に変換 ← **失敗していたのはここ**

②は SDK の中で静かに失敗するので、①のログだけ見ると「Integrity は通ってるのに何かがおかしい」という見え方になります。ここまでで「疑うべきは② Exchange の 403」と分かりましたが、②が 403 を返す理由はまだ複数あります。

## 切り分け: 容疑者を消してから、3つの SHA-256 を突合する

まず Firebase Console / REST で確認できる系を消し込みました。

| 容疑者 | 結果 |
|---|---|
| そもそもアプリが App Check に未登録 | 登録済み ✓ |
| App ID の不一致 | logcat の値と一致 ✓ |
| Cloud プロジェクト番号の不一致 | logcat の `cloudProjectNumber` と一致 ✓ |
| Play Integrity の verdict 要件で弾かれている | 要件は最緩設定。弾く設定になっていない ✓ |

全部シロ。ここまで消えると、残る容疑者は**署名証明書の不一致**だけです。ただ「多分これだろう」で SHA を足して直った気になるのは気持ちが悪いので、証拠を3点集めて突合することにしました。

### (1) Firebase に登録されている SHA-256

Console でも見られますが、REST で取ると突合が楽です（`firebase-tools` でログイン済みなら access token を使い回せます）。

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://firebase.googleapis.com/v1beta1/projects/<PROJECT>/androidApps/<APP_ID>/sha" | jq .
# → 我々のケースでは shaHash: 1111...aaaa (SHA_256) が1件だけ登録されていた
```

### (2) 配布した AAB の署名（= Upload key）

AAB は Signature Scheme v1 形式なので `META-INF/*.RSA` から証明書を抜けます。

```bash
unzip -j -q app-release.aab "META-INF/*.RSA" -d /tmp/aab-meta
openssl pkcs7 -inform DER -in /tmp/aab-meta/*.RSA -print_certs > /tmp/aab.pem
openssl x509 -noout -fingerprint -sha256 -in /tmp/aab.pem
# → 1111...aaaa（Firebase 登録値と一致 = 登録されているのは Upload key）
```

### (3) 実機に入っている Play 配布版 APK の署名（= 実際にユーザーの手元にある証明書）

```bash
APKPATH=$(adb shell pm path com.example.app | grep base.apk | sed 's/^package://')
adb pull "$APKPATH" /tmp/installed.apk
apksigner verify --print-certs /tmp/installed.apk
# → Signer #1 certificate SHA-256 digest: 2222...bbbb
```

ここで2つ目のハマりがありました。**(2) と同じ `META-INF/*.RSA` ルートは実機 APK には通用しません**。Play 配布版は Signature Scheme v2/v3 で、署名は zip エントリではなく APK 末尾近くの **APK Signing Block** に入っています。`unzip` で覗いても空で、一瞬「署名が無い？」と面食らいます。`apksigner`（Android SDK build-tools 同梱）を使ってください。

:::details 余談: Java も build-tools も無かったので、その場でパーサを書いた
実はこの時、調査していたマシンに Java が入っておらず apksigner が使えませんでした。APK Signing Block は構造が公開されているので、pure Python で直接パースしました。ファイル末尾から magic `APK Sig Block 42` を探索し、ブロック内の ID-value ペアから `0x7109871a`（v2）/ `0xf05368c0`（v3）の signer block を辿り、certificates フィールドの DER 証明書を `hashlib.sha256` にかけるだけ。200行未満で書けます。CI コンテナなど Java を入れたくない環境での最終手段として覚えておくと便利です。
:::

### 突合結果

```
Firebase 登録:        1111...aaaa   ─┐ 一致（Upload key）
AAB (Upload key):     1111...aaaa   ─┘
実機 Play 配布版:      2222...bbbb   ← 別物。これが決定打
```

**(1)=(2)≠(3)**。Firebase が知っているのは Upload key だけで、ユーザーの手元で動いているアプリは Firebase の知らない証明書で署名されている——ここでようやく確定しました。他の容疑者を先に消し込んであったので、原因はこれで一意です。

## なぜこうなるか

Play App Signing（現在は実質必須）では、開発者が持つ **Upload key** は「Play Console に AAB を届けるための鍵」でしかなく、ユーザーに配布される APK は Google が管理する **App signing key** で**再署名**されます。つまり本番のアプリの身分証明書は Google の手元にある方です。

Play Integrity の verdict はこの App signing key の証明書に基づいて発行されるので、Firebase 側が Upload key しか知らなければ、Exchange API から見た本番アプリは「登録のない未知のアプリ」= 403 になります。開発ビルド（Upload key 署名の直インストール）では再署名が起きないため正常に動く——これが「Play 配布版だけ壊れる」の正体です。

## 解決

Play Console → 対象アプリ → **App integrity**（アプリの完全性）→ App signing key certificate の SHA-256 をコピーし、Firebase Console → プロジェクト設定 → マイアプリ → 対象 Android アプリ → 「フィンガープリントを追加」で登録します。**既存の Upload key の SHA-256 は消さずに残します**（開発ビルドが動かなくなるので）。debug key も含め、使う署名は全部登録しておくのが定石です。

REST でやる場合:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"shaHash":"2222...bbbb","certType":"SHA_256"}' \
  "https://firebase.googleapis.com/v1beta1/projects/<PROJECT>/androidApps/<APP_ID>/sha"
```

登録後はアプリを再起動して初回 `getToken()` の成功を logcat で確認、サーバー側のヘッダ欠如エラーが消えれば完了です。我々のケースでは、SHA-256 を追加登録した数分後にサーバー側のエラーが急減し、Cloud Audit Logs の追加オペレーション時刻とエラー急減のタイミングが分単位で一致することまで確認できました。修正効果の裏取りとしてはこれ以上ないきれいさでした。

## 同じ根っこの罠: App Links（assetlinks.json）

この「本番の身分証明書は App signing key の方」問題は App Check 専用ではありません。**Android App Links の `assetlinks.json` に書く SHA-256 も App signing key のもの**です。

つまり、Upload key や debug key で自己署名したビルドをいくらインストールしても App Links の検証は**構造的に通りません**。App Links を検証したければ、Internal testing / Closed testing 経由で **Play ストアから配信されたビルド**を落とすしかない。「App Links がローカルビルドで動かない」と悩んでいる場合、コードでも `assetlinks.json` の書式でもなく、署名がそもそも別物である可能性を先に疑ってください。

逆にこれを利用した小技もあります。対象アプリが App Links 対応済みなら、`assetlinks.json` は**外部公開ファイル**なので、

```bash
curl https://<domain>/.well-known/assetlinks.json
```

で **App signing key の SHA-256 が adb も Play Console 権限も無しで即取れます**。切り分けの「(3) 実機 APK の署名」の代わりの証拠として使える場面があります。

## まとめ

- Play 配布版だけ App Check が死ぬ → まず **App signing key の SHA-256 が Firebase 未登録**を疑う。Play App Signing が一般化した今、確率的にこれが一番多い
- 「Integrity token は取れているのに」は正常な混乱。トークン取得は2段階で、死んでいるのは Firebase Exchange 側
- 当てずっぽうで直すより **3点突合**（Firebase 登録値 / AAB / 実機 APK）で決定打を出す。実機 APK は v2/v3 署名なので `apksigner`（`META-INF` 覗きは通用しない）
- Upload key の登録は残したまま **App signing key を追加**登録
- `assetlinks.json`（App Links）も同じ根っこ。自己署名ビルドでは永遠に検証が通らない

---

koedesk は macOS / Windows / iOS / Android で動く音声入力アプリです。開発現場の裏側の話をたまに書いています → [koedesk.app](https://koedesk.app/ja/)
