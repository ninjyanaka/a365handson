# 前提 — 3 レイヤーの理解 / Blueprint→Instance / 事前準備

[← 目次に戻る](./README.md)

ハンズオンに入る前に、**混乱の元になりやすい 3 つの考え方**を最初に揃えます。Agent 365 の「動かない」の多くは、ここの混同から生まれます。

---

## 1. 3 つのレイヤーを分けて考える

| レイヤー | 役割 |
| --- | --- |
| **① M365 Agents SDK** | メッセージの送受信を担う「配管」。チャネル抽象化・状態管理を提供。**これ単体では Agent ID は付かない**（ただの bot）。 |
| **② Agent 365 SDK** | ①に **Observability・Notifications・MCP アクセス**を足す拡張。テレメトリはここが OpenTelemetry で送る。 |
| **③ Agent 365 / Entra Agent ID** | `blueprint → instance` で **agentic identity（Entra Agent ID）** を発行する仕組み。これがあって初めて agentic 認証・OBO・Observability が成立する。 |

> [!IMPORTANT]
> **Agent ID は instance 化して初めて付く。** `blueprint` は「テンプレート」であって実体ではありません。
> Teams で `@mention` できる実体（agent identity + agent user）は、blueprint から instance を作って初めて生まれます。

---

## 2. Blueprint → Instance の関係

```
         +--------------------+              +---------------------------------+
         |     Blueprint      |  + Add       |  Instance A  Entra Agent ID      |
         |  IT 承認テンプレート | ──instance─▶ |              12f560ef-...        |
         |  機能/ツール/制約/監査|              +---------------------------------+
         |  ポリシーを全 Instance|              |  Instance B  Entra Agent ID — → 実値 |
         |  に継承             |              +---------------------------------+
         +--------------------+              |  Instance C  Entra Agent ID — → 実値 |
                                             +---------------------------------+
```

instance 作成後、blueprint 詳細の **Entra agent ID が「—」から実値**（例 `12f560ef-...`）に変わります。
この値が **Observability 送信時の `agentId`** になり、`agent user`（例 `hana-assistant@tenant.onmicrosoft.com`）がテナントに現れて Teams で `@mention` 可能になります（同期に数分かかることがあります）。

> [!TIP]
> **「そもそも Entra Agent ID とは？」「`a365 setup all` は内部で何を作るのか」「instance で何が払い出されるのか」** を一段深く知りたい場合は、
> [01-entra-agent-id-deep-dive.md（深掘り）](./01-entra-agent-id-deep-dive.md) を参照してください（Blueprint / Blueprint Principal / Agent Identity / Agent's User Account の関係、Graph 作成フローとの対応を解説）。

---

## 3. 事前準備

### 3.1 必要なもの

- [ ] Node.js 18 以降（サンプルは Node.js / TypeScript / ts-node）
- [ ] Agent 365 DevTools CLI（`a365` コマンド）
- [ ] Azure CLI（`az`。orphan アプリ削除などに使用）
- [ ] devtunnel CLI（ローカルを外部公開する）
- [ ] **ライセンス**：Microsoft 365 E7、または E5 + Microsoft Agent 365
- [ ] **ロール**：Global Administrator（または Agent ID Administrator）

### 3.2 ポート設計（複数エージェントを並行運用するとき）

エージェントごとにフォルダとポートを分けると衝突しません。例：agent モードを `3978`、AI Teammate を `3979` に分離。

> [!WARNING]
> **devtunnel・`.env` の `PORT`・Bot の Notification URL の 3 か所でポートが一致**している必要があります。
> 別エージェントはフォルダごと分けること。ただしサンプルをコピーすると `.env` や `agent.ts` の中身も引き継がれるため、デバッグ用 `console.log` などは新フォルダには無い点に注意。

### 3.3 主要ファイルと役割

| ファイル | 役割 |
| --- | --- |
| `src/index.ts` | エントリポイント。OpenTelemetry distro の初期化（**最重要：他 import より前**）、Express サーバ、認証ミドルウェア、`/api/messages`・`/api/health`。 |
| `src/agent.ts` | `AgentApplication` 本体。メッセージ処理、Observability トークンの preload、通知処理。 |
| `src/token-cache.ts` | カスタム token resolver とローカルキャッシュ（`Use_Custom_Resolver=true` のときのみ）。 |
| `.env` | 全設定。`a365 setup all` がここに値をスタンプする。 |
| `a365.config.json` | **AI Teammate 用の手動作成 config**（agent user の UPN 等）。 |
| `a365.generated.config.json` | setup が生成。blueprint ID・同意状況・messaging endpoint を記録。 |

---

[← 目次](./README.md) ｜ [深掘り：Entra Agent ID / Registry →](./01-entra-agent-id-deep-dive.md) ｜ [Step 3 登録 →](./step3-register.md)
