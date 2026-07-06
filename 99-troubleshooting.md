# 付録 — トラブルシュート早見表 & クイックチェックリスト

[← 目次](./README.md) ｜ [← Step 10：ローカルエージェント](./10-local-agent.md)

実機で踏んだ落とし穴を症状ベースでまとめます。**上から順に疑う**と早いです。

---

## トラブルシュート早見表

| 症状 | 原因と対処 |
| --- | --- |
| Teams で送っても本体が完全に無反応 | ① Notification URL が古い devtunnel を指す ② `NODE_ENV≠production` で認証ミドルウェアが弾く ③ devtunnel が匿名不許可/停止。まず `curl.exe .../api/health` と inspect 画面で切り分け。 |
| 応答送信が **502 Bad Gateway** | 本体がクラッシュ/停止していて devtunnel が localhost に繋げない。本体を再起動。元エラーがスタックの上にある。 |
| Observability が送られない（**403**） | blueprint に Observability 権限が未同意。generated config の `resourceConsents` を確認。新 blueprint は同意し直し。 |
| Observability が送られない（**無言で失敗**） | `ENABLE_A365_OBSERVABILITY_EXPORTER` が `false`/タイポ（例 `fals`）。`a365.enabled=true` と両方必要。 |
| `agentId` が空・**AADSTS900023** | Playground（emulator）はダミー tenant を渡す。本物の Teams instance で検証する。 |
| **AADSTS82001**（app-only 不可） | agentic アプリに対し Playground が S2S トークンを要求。Playground の構造的制約。Teams で検証。 |
| `MCP: rawServers.map is not a function` | MCP 未使用なのに登録を試みている。`client.ts` の `addToolServersToAgent` をガード/コメントアウト。Observability には無関係。 |
| スパンがコンソールに出ない | `NODE_ENV=production` だと `enableConsoleExporters=false`。これは正常。送信確認は `A365_OBSERVABILITY_LOG_LEVEL` と管理センターで。 |
| （管理者向け）送信は200成功なのに `CloudAppEvents` に0件 | 2条件のどちらか欠落：①テナントに MDA O365 コネクタ未接続で `CloudAppEvents` テーブル未生成（`KS204 / Failed to resolve table` で判別）②`gen_ai.agent.id` が `AgentsInfo` に未登録の appId（Blueprint と Instance の取り違えが典型）。詳細は [Step 7：観測 › Defender Advanced Hunting](./07-observability.md#defender-advanced-huntingcloudappeventsで横断的にハンティングする) 参照。 |
| （管理者向け）Observability データが見当たらないが、CA ブロックは効いている | 正常な状態。**Identity/ガバナンス（CA・Purview DLP）はコード非依存で常に効く**が、**深い per-span トレースは自前ホスト側の SDK 配線が必須**。テレメトリの有無とエージェントの統制状態は別軸で判断する（[Step 7：観測 › IT管理者・セキュリティ管理者向け](./07-observability.md#it管理者セキュリティ管理者向けobservability-とガバナンスの関係コード依存-vs-コード非依存)参照）。 |

---

## クイックチェックリスト

### 展開フロー（7 ステップ）

1. [ ] `a365.config.json` を AI Teammate 用に用意（`agentUserPrincipalName` あり）
2. [ ] `a365 setup all --aiteammate` を実行
3. [ ] generated config：`completed=True` / `resourceConsents` 全 True / `messagingEndpoint`=devtunnel を確認
4. [ ] `a365 publish` → `manifest.zip` を管理センターにアップロード
5. [ ] 管理センターで blueprint を承認
6. [ ] **+ Add instance** で instance 作成 → Entra Agent ID が付くのを確認
7. [ ] Teams で `@mention` して疎通

### 毎回確認する `.env` / 環境

- [ ] `ENABLE_A365_OBSERVABILITY_EXPORTER=true`（タイポ注意。setup で false に戻されることがある）
- [ ] `NODE_ENV=production`（Teams 検証時。認証を有効化）
- [ ] `PORT` が devtunnel のポートと一致
- [ ] Bot の Notification URL が現在の devtunnel を指す
- [ ] `agent.ts` に `🔎 OBS` / `✅❌` デバッグログが入っている（新フォルダは要再仕込み）

### デバッグ起動コマンド

```powershell
# devtunnel（名前付きを使い回す）
devtunnel host langchain-agent

# 本体（Observability ログ ON）
$env:A365_OBSERVABILITY_LOG_LEVEL="info"
$env:OTEL_LOG_LEVEL="INFO"
npm run dev

# 疎通テスト
curl.exe https://<tunnel>-3979.use2.devtunnels.ms/api/health
```

---

## セキュリティ（毎回のクリーンアップ）

> [!IMPORTANT]
> - **`clientSecret` を平文で扱ったら失効・再発行。** 本番は WIF/FIC または Managed Identity（Azure）を使い、シークレットを置かない。
> - AWS ホスト時は Managed Identity が使えないため、**WIF（Federated Identity Credential）** を推奨。Agent ID 自体は Entra 発行なので**ホスト場所に依存せず付与**される。

---

## 付録 A：クラウド別ホスティングの違い

Agent ID は「どこで動くか」ではなく「Entra のどこで作られたか」で決まります。ホスト場所で変わるのは**資格情報の取り方のみ**。

| ホスト | 認証・注意点 |
| --- | --- |
| ローカル / devtunnel | ClientSecret（暫定）。messaging endpoint は devtunnel URL。Agent ID は付く。 |
| Azure ホスト | **Managed Identity** が使える（推奨・シークレット不要）。 |
| AWS（Elastic Beanstalk 等） | Managed Identity 不可。**WIF/FIC 推奨**（or Secret）。egress で `*.cloud.microsoft` / `login.microsoftonline.com` への 443 開放が必要。Agent ID は付く。 |

## 付録 B：Copilot Studio との違い（なぜあちらは OBO が楽か）

Copilot Studio エージェントは Microsoft がホストし、OBO のトークン交換・同意・配線を**プラットフォームが肩代わり**します。自前ホストの custom engine agent は、その配線（exchangeToken / token cache）を自分で持つ必要があります。
「Copilot Studio では OBO が動くのに自前だと動かない」の比較は、多くの場合 **本物の製品面 vs エミュレータ** を比べているにすぎません。自前エージェントでも、**Teams で instance を通せば同じ agentic identity コンテキスト**が得られ、OBO・Observability は成立します。

---

[← 目次](./README.md)
