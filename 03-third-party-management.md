# Step 3 — サードパーティ管理：a365 CLI で自前 / 3P エージェントを管理下に

[← Step 2：Agent Registry / Entra Agent ID](./02-entra-agent-id.md) ｜ [← 目次](./README.md) ｜ [Step 4：登録 →](./04-register.md)

Microsoft 以外のプラットフォームや**自前ホストのカスタムエージェント**（本ワークショップの題材＝LangChain + Node.js）を Agent 365 の管理下に置く方法をまとめます。
**a365 コマンドの内容はこのページに集約**しています。以降のワークショップ（Step 4〜8）では、ここを参照しながら各コマンドを実行します。

---

## 0. 開発の前提（自前ホスト編）

自前ホストのエージェントを `a365` で管理下に置くには、まず **3 レイヤーの考え方**と**開発環境**を押さえます。

### 3 つのレイヤーを分けて考える

| レイヤー | 役割 |
| --- | --- |
| **① M365 Agents SDK** | メッセージ送受信を担う「配管」。チャネル抽象化・状態管理。**これ単体では Agent ID は付かない**（ただの bot）。 |
| **② Agent 365 SDK** | ①に **Observability・Notifications・MCP アクセス**を足す拡張。テレメトリは OpenTelemetry でここが送る。 |
| **③ Agent 365 / Entra Agent ID** | `blueprint → instance` で **agentic identity（Entra Agent ID）** を発行（→ [Step 2](./02-entra-agent-id.md)）。これがあって初めて agentic 認証・OBO・Observability が成立。 |

> [!IMPORTANT]
> **Agent ID は instance 化して初めて付く**（blueprint はテンプレート）。詳しくは [Step 2：Agent Registry / Entra Agent ID](./02-entra-agent-id.md)。

### 開発に必要なもの

- [ ] Node.js 18 以降（サンプルは Node.js / TypeScript / ts-node）
- [ ] Agent 365 DevTools CLI（`a365`）
- [ ] Azure CLI（`az`。orphan アプリ削除等）
- [ ] devtunnel CLI（ローカルを外部公開）
- [ ] **ライセンス**：Microsoft 365 E7、または E5 + Microsoft Agent 365
- [ ] **ロール**：Global Administrator（または Agent ID Administrator）

### ポート設計（複数エージェントの並行運用）

エージェントごとにフォルダとポートを分ける（例：agent モード `3978`／AI Teammate `3979`）。

### 主要ファイルと役割

| ファイル | 役割 |
| --- | --- |
| `src/index.ts` | エントリポイント。OpenTelemetry 初期化（**他 import より前**）、Express、認証、`/api/messages`・`/api/health`。 |
| `src/agent.ts` | `AgentApplication` 本体。メッセージ処理、Observability トークン preload、通知。 |
| `src/token-cache.ts` | カスタム token resolver とローカルキャッシュ（`Use_Custom_Resolver=true` のとき）。 |
| `.env` | 全設定。`a365 setup all` が値をスタンプ。 |
| `a365.config.json` | **AI Teammate 用の手動 config**（agent user の UPN 等）。 |
| `a365.generated.config.json` | setup が生成。blueprint ID・同意状況・messaging endpoint。 |

---

## 1. 登録経路 — 作り方で変わる

エージェントの**作られ方**によって、Agent Registry への登録経路が変わります。

| 作られ方 | 登録 |
| --- | --- |
| Copilot Studio / Microsoft 365 Agents / Foundry | **自動登録** |
| 上記以外の MS プラットフォーム / Microsoft 以外 | **Microsoft Graph API でセルフサービス登録** |
| カスタム（自前ホスト）＝本ワークショップ | **`a365` CLI ＋ manifest（.zip）を管理センターにアップロード** |

---

## 2. a365 CLI コマンド リファレンス

| コマンド | 役割 |
| --- | --- |
| `a365 setup all [--aiteammate]` | 要件確認 → Blueprint 作成 → 資格情報 → 継承権限/同意 → **Agent Identity 作成** → 登録 → `.env` スタンプ（冪等＝再実行安全）。`--aiteammate` で AI Teammate 構成。 |
| `a365 publish` | `manifest.json` に blueprint ID を埋め、`manifest.zip` を生成（→ 管理センターにアップロード）。 |
| `a365 develop get-token` | 開発・テスト用のトークン取得。 |
| `a365 query-entra` | Entra 側の登録状態を照会（変更前の確認に）。 |
| `a365 logs export` | 実行ログのエクスポート。 |
| `a365 cleanup` | 作業ディレクトリの config を読み、**全 Agent 365 リソースを削除**（破壊的）。 |

> [!TIP]
> `--dry-run` や `a365 query-entra` で**変更前に状態を検証**するのが運用のセーフガードです。

---

## 3. `a365 setup all` は内部で何を作っているか

`a365 setup all` は、本来 **Microsoft Graph で手作業する Agent ID 作成フローを自動化**したものです。手動 Graph 版の流れと対応づけると、中で何が起きているかが見えます。

| Graph 手動フロー | `a365 setup all` の対応出力（例） | 生成物 |
| --- | --- | --- |
| ① Graph に接続 | 要件チェック（Azure 認証 / Graph モジュール） | — |
| ② **Blueprint を作成** | `Found existing blueprint`（無ければ新規作成） | Agent identity blueprint（appId） |
| ③ **資格情報を追加** | `Client secret is valid, skipping creation` | クライアントシークレット（※本番は WIF/Managed ID 推奨） |
| ④ **Blueprint Principal を作成** | 継承権限の構成 / 委任・同意の付与 | blueprint principal（`AgentIdentity.CreateAsManager` 自動付与） |
| ⑤ Blueprint のトークン取得 | S2S app role assignments / delegated consent | `roles` に `AgentIdentity.CreateAsManager` |
| ⑥ **Agent Identity を作成** | `Agent identity created (ID: 097c…)` | Agent Identity |
| ⑦ 登録・設定 | `Agent registered (ID: T_f379…)` / `appsettings.json` にスタンプ | レジストリ登録・設定反映 |

> [!IMPORTANT]
> **blueprint principal は自動では作られません**（Graph 手作業の場合 ④ を飛ばすと `400: The Agent Blueprint Principal ... does not exist`）。principal を作ると `AgentIdentity.CreateAsManager` が**自動付与**され、**この権限は取り消せません**（無効化は principal ごと削除）。`a365 setup all` はこの一連を冪等（`reused` = 既存再利用）に処理します。

---

## 4. （参考）Graph API で Agent ID を手動作成する

中身を理解するための参考です（詳細・サンプルコードは [Zenn / Graph 編](https://zenn.dev/microsoft/articles/91df843374fbde)）。

```text
Step 1: Graph に接続（Scopes: AgentIdentityBlueprint.Create, .AddRemoveCreds.All,
                      AgentIdentityBlueprintPrincipal.Create, User.Read）
Step 2: Agent identity blueprint を作成   POST /v1.0/applications/graph.agentIdentityBlueprint
Step 3: 資格情報（client secret）を追加     ※本番は Managed Identity / FIC 推奨
Step 4: blueprint principal を作成         POST /v1.0/servicePrincipals/graph.agentIdentityBlueprintPrincipal
Step 5: blueprint のトークン取得（client_credentials）→ roles に AgentIdentity.CreateAsManager
Step 6: Agent Identity を作成              POST /beta/servicePrincipals/microsoft.graph.agentIdentity
                                          （displayName / agentIdentityBlueprintId / sponsors 必須）
Step 7: 確認  Entra 管理センター › Agent identities › All agent identities
```

- **必要ライセンス**：Microsoft 365 Copilot ／ Frontier プログラム有効。
- **必要ロール**：Agent ID Developer または Agent ID Administrator（blueprint 作成）、Privileged Role Administrator（Graph 権限付与）。
- **注意**：Agent Identity 作成 API は現在 **beta エンドポイントのみ**。Sponsor は **User オブジェクト必須**。

---

## 5. 他クラウドの取り込み — Registry Sync と統合段階

自前ホスト以外（他クラウド／3P PaaS）のエージェントは、可視化から段階的に統制を強化します。

| 段階 | 内容 |
| --- | --- |
| ① 統合不要（事前統合パートナー） | Genspark・Zendesk・n8n 等。追加作業なしで可視化＋基本管理。 |
| ② Registry Sync（軽量統合） | Amazon Bedrock・Google Gemini・Agentforce・Databricks Genie を横断で可視化・棚卸し（プレビュー）。 |
| ③ SDK オンボード（完全統合） | **Entra Agent ID を付与**し、ポリシー適用・ライフサイクル管理までネイティブ同等に。 |

> [!NOTE]
> 深い統制（条件付きアクセス・DLP 等）はネイティブ／SDK 連携が優位です。可視化だけなら素早く始められます（→ Registry Sync はパブリックプレビュー）。

---

## 参考リンク

- [エージェント レジストリ（管理センター）](https://learn.microsoft.com/microsoft-365/admin/manage/agent-registry)
- [Microsoft Entra Agent ID を作成する方法（Graph 編・サンプルコード付き／Zenn）](https://zenn.dev/microsoft/articles/91df843374fbde)
- [第三者エージェント連携（Entra Agent ID）](https://learn.microsoft.com/entra/agent-id/)

---

[← Step 2：Agent Registry / Entra Agent ID](./02-entra-agent-id.md) ｜ [Step 4：登録 →](./04-register.md)
