# Step 5 — 認証パターン：委任(OBO) と 自律(S2S)

[← 目次](./README.md) ｜ [← Step 4：登録](./04-register.md) ｜ [次：Step 6 公開 →](./06-publish.md)

Agent 365 のエージェントは **2 つの認証パターン**で動きます。ここでは**概要**を押さえます（実装の詳細は末尾の Learn 記事を参照）。いずれのトークンも主体は **Entra Agent ID（Blueprint → Agent Identity）** です。

## 2 パターンの全体像

| | **委任アクセス（OBO）** | **自律型（S2S / アプリ専用）** |
| --- | --- | --- |
| 動き方 | **サインイン中のユーザーに代わって**実行 | **ユーザー不在で自律的に**実行 |
| 代表例 | AI Teammate（Teams で `@mention`） | バックグラウンド／夜間・イベント駆動 |
| OAuth フロー | On-Behalf-Of（ユーザートークンを交換） | クライアント資格情報（app-only） |
| 権限の源 | **ユーザーの委任権限**（要ユーザー同意） | **エージェント固有のアプリ権限**（テナント管理者が付与） |
| トークン claim | `scp` | `roles` |
| 監査の記録主体 | **ユーザー ID として** | **エージェント ID として** |
| Observability 経路 | `/observability/…`（agentic token cache・自動） | `/observabilityService/…`（手動 token resolver） |

> 共通スコープ例：`api://9b975845-…/Agent365.Observability.OtelWrite`（S2S 用の App role と OBO 用の Delegated scope を**同名で登録**）。

## ① 委任アクセス（On-Behalf-Of）

- サインイン中のユーザーに代わって、**ユーザーの権限・データ範囲**で実行 → 操作は**ユーザー ID として**記録。
- **エージェント（blueprint）は対話型サインイン（`/authorize`）を直接開始できません。** クライアントから**ユーザートークンを受け取り、OBO 交換**を行います。
- **ユーザー同意**が必要。`InheritDelegatedPermissions` で blueprint の委任権限を継承でき、マルチインスタンスの同意を簡素化できます。

```
ユーザーがクライアントでサインイン（ユーザートークン）
        └─▶ エージェントが受け取って OBO 交換 ─▶ ユーザーのコンテキストでアクセス（scp）
```

## ② 自律型（アプリ専用 / S2S）

- **ユーザー不在**で、**クライアント資格情報フロー**により自律実行 → 操作は**エージェント ID として**記録。
- Agent Identity が**自分自身のトークン**を取得（blueprint が子の Agent Identity を偽装）。**必要な権限はテナント管理者が付与**。
- Agent ID は常に**シングルテナント**（1 つの Agent ID は 1 つの blueprint のみが所有）。

```
エージェント（blueprint）が資格情報を提示 ─▶ Agent Identity が交換 ─▶ アプリ専用トークン（roles）
```

## 押さえどころ

- **OBO は自前ホストの SDK でも成立**します。Copilot Studio が楽なのは Microsoft がホストし配線を肩代わりするため。自前でも **Teams で instance を通せば同じ agentic コンテキスト**が得られます（「動かない」の多くは *本物の製品面 vs エミュレータ* の比較）。
- 1 つの agent アプリが **OBO と S2S の両方**に参加可能（昼は AI Teammate、夜は自律バッチ）。
- 実装は **承認済み SDK（Microsoft.Identity.Web / Entra ID Auth SDK）** を推奨。本番の資格情報は **Managed Identity / FIC**（クライアントシークレットは非推奨）。

## 参考

- [On-Behalf-Of フロー（Microsoft Entra Agent ID）](https://learn.microsoft.com/ja-jp/entra/agent-id/agent-on-behalf-of-oauth-flow)
- [自律アプリの OAuth フロー — アプリ専用プロトコル（Microsoft Entra Agent ID）](https://learn.microsoft.com/ja-jp/entra/agent-id/agent-autonomous-app-oauth-flow)

[← Step 4：登録](./04-register.md) ｜ [次：Step 6 公開 →](./06-publish.md)
