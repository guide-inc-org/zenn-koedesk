---
title: "Cloudflare Workersから呼ぶとOpenAI APIだけ403になる件をDurable Objectsで回避した"
emoji: "🌏"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "durableobjects", "openai", "typescript"]
published: true
---

## はじめに

先に結論を書きます。

Cloudflare Workers から OpenAI API を呼ぶと、**Worker がどのエッジで実行されたかによって** `403 unsupported_country_region_territory` が返ることがあります。API キーも実装も正しいのに、です。うちでは **Durable Objects を `locationHint` 付きで北米に置き、OpenAI への fetch だけそこを経由させる41行のプロキシ**で回避しました。本番投入後、約1ヶ月の稼働で再発ゼロです。

このエラーで検索すると「問題の特定まで」で止まっている記事や、Cloudflare Community の「Enterprise 限定機能を使え」という回答が多く、無料〜Paid プランで完結する実装解がまとまっていなかったので書いておきます。

## 何が起きたか

koedesk という音声入力アプリを作っていて、文字起こしのバックエンドが Cloudflare Workers にあります。当時は文字起こしエンジンの一つとして OpenAI（gpt-4o-transcribe 系）を呼んでいました。

ある日、ベトナム（開発拠点がここです）からの文字起こしが失敗するようになりました。OpenAI からのレスポンスはこれです。

```json
{
  "error": {
    "code": "unsupported_country_region_territory",
    "message": "Country, region, or territory not supported",
    "type": "request_forbidden"
  }
}
```

最初に疑うのは「ベトナムからのアクセスだから弾かれた？」ですが、切り分けるとそうではありませんでした。

- ベトナムのローカルマシンから**同じ API キーで直接** OpenAI を叩く → **200 OK**（ベトナムは OpenAI のサポート対象国です）
- 同じリクエストを **Workers 経由**で送る → **403**

つまり弾かれているのはユーザーの国でも API キーでもなく、**Worker が実行されたエッジの共有出口 IP** です。Cloudflare Workers はリクエストを受けたエッジ（の近く）で実行され、外向き fetch はそのエッジの IP から出ます。このエッジが OpenAI の非サポート地域（香港などが有名です）に当たると、リクエストの中身に関係なく 403 になります。

厄介な点が2つあります。

1. **実行エッジは選べない**。ユーザーの位置とネットワーク状況で決まるので、アプリ側では制御できません
2. **Smart Placement では防げない**。Smart Placement は実行位置をバックエンド近くに最適化する機能であって、「この地域で実行しない」という保証を与えるものではありません（Cloudflare Community でも同種報告が多数あり、公式の回答は Regional Services＝Enterprise 限定に誘導されがちです）

同じ現象は Zenn でも[報告されています](https://zenn.dev/retore/articles/a04046c8829d8f)が、この記事の時点では「様子見」で終わっています。実際、対策の選択肢が Enterprise 契約か様子見しかないように見えるんですよね。もう一つあります。

## 解決策: Durable Objects の「場所が固定される」性質を使う

Durable Objects（DO）には、この問題にちょうど効く性質があります。**DO インスタンスは最初に作られた場所に留まり続ける**、そして**生成時に `locationHint` で初期配置を誘導できる**、という2点です。

つまり「北米西部に DO を1個置いて、OpenAI への fetch を全部そいつに中継させる」と、OpenAI から見たリクエスト元は常に北米になります。

プロキシ本体は41行です。状態は持ちません。

```typescript:workers/src/openai-proxy.ts
import { DurableObject } from "cloudflare:workers";

/**
 * Stateless proxy DO that relays fetch requests to OpenAI from a fixed location (wnam).
 * This avoids the 403 "unsupported_country_region_territory" error that occurs when
 * Cloudflare Workers execute at edges in geo-restricted regions.
 */
export class OpenAIProxy extends DurableObject {
  async fetch(request: Request): Promise<Response> {
    const targetUrl = request.headers.get("X-Target-URL");
    if (!targetUrl) {
      return new Response("Missing X-Target-URL", { status: 400 });
    }

    const headers = new Headers(request.headers);
    headers.delete("X-Target-URL");

    return fetch(targetUrl, {
      method: request.method,
      headers,
      body: request.body,
    });
  }
}
```

呼び出し側はこうです。`idFromName("singleton")` で常に同じ1インスタンスを掴み、`locationHint: "wnam"`（北米西部）を渡します。

```typescript:workers/src/transcribe.ts
/** Works for any provider (OpenAI, Gemini, etc.) — just pass the target URL and request init. */
async function fetchViaGeoProxy(env: Env, url: string, init: RequestInit): Promise<Response> {
  const id = env.OPENAI_PROXY.idFromName("singleton");
  const stub = env.OPENAI_PROXY.get(id, { locationHint: "wnam" });
  const headers = new Headers(init.headers as HeadersInit);
  headers.set("X-Target-URL", url);
  return stub.fetch(new Request("https://geo-proxy/", {
    method: init.method || "POST",
    headers,
    body: init.body,
  }));
}
```

あとは元の `fetch(openaiUrl, init)` を `fetchViaGeoProxy(env, openaiUrl, init)` に差し替えるだけです。地域制限のないプロバイダ（当時併用していた ElevenLabs / Mistral）は直接 fetch のままにして、余計なホップを増やさないようにしました。

wrangler.toml にはバインディングとマイグレーションを足します。

```toml:wrangler.toml
[[durable_objects.bindings]]
name = "OPENAI_PROXY"
class_name = "OpenAIProxy"

[[migrations]]
tag = "v1"
new_classes = ["OpenAIProxy"]
```

デプロイ後、それまで 403 で落ちていたベトナムエッジ経由の文字起こしが gpt-4o-transcribe / gpt-4o-mini-transcribe とも通るようになりました。

## 正直に書いておくこと

**`locationHint` は保証ではありません。** Cloudflare のドキュメント上、これは初期配置の best-effort なヒントです。「wnam に必ず置かれる」とは書かれていません。ただし DO は一度作られればその場所に留まる性質があるので、実質的には「最初の生成が妥当な場所で行われれば、以降は固定」として動きます。うちの場合、本番で約1ヶ月運用して 403 の再発はゼロでした。厳密な保証が契約上必要なケース（それこそ Enterprise の Regional Services の出番です）とは要件のレイヤーが違う、と理解しておくのがいいと思います。

**コスト**はほぼ気にしなくてよいレベルです。DO を使うには Workers Paid（$5/月）が必要ですが、すでに Paid なら追加固定費はなく、リクエスト課金も無料枠（月100万リクエスト）に収まる規模でした。中継1ホップ分のレイテンシは音声ファイルの文字起こしという用途では誤差でした。

**プロバイダごとの地域制限の実態**も当時調べたので置いておきます（2026年3月時点）。

| プロバイダ | 地域制限 |
|---|---|
| OpenAI | 非サポート地域からのアクセスを 403 で拒否（今回の主役） |
| ElevenLabs | 制裁対象国のみ（通常のエッジでは実質問題なし） |
| Mistral | 制限なし |
| Groq | VPN ブロックあり。Cloudflare IP がブロックされた事例の報告あり |

呼び出し側を `fetchViaGeoProxy(env, url, init)` の形で汎用にしておいたのはこのためで、別のプロバイダが同じ挙動を始めたら fetch を1行差し替えるだけで同じ回避が効きます。

## 後日談

このプロキシは現在の koedesk には残っていません。**回避策が壊れたからではなく、その後のエンジン選定で OpenAI の利用自体をやめたため**、フォールバック経路ごと撤去しました（エンジン選定の話は[STT ベンチマークの記事](https://zenn.dev/koedesk/articles/stt-benchmark-13-engines-12-languages)に書いています）。手法としては Workers から地域制限のある API を呼ぶ場面全般で使えるはずです。

Workers で外部 API を叩いていて「特定の地域のユーザーからだけ 403 の報告が来る」「再現したりしなかったりする」という症状が出たら、まず実行エッジを疑ってみてください。レスポンスヘッダや `request.cf.colo` でエッジのコロケーション（3レターコード）を確認すると、切り分けが一気に進みます。

---

koedesk は macOS / Windows / iOS / Android で動く音声入力アプリです。この記事のような裏側の話をたまに書いています → [koedesk.app](https://koedesk.app/ja/)
