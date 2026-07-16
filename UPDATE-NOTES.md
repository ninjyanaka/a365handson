# 更新メモ（GitHub 公開前の点検・修正）

このフォルダーは、`a365handson-main` の Markdown 一式を **GitHub 公開向けに点検・修正した版**です。
最新の Microsoft Learn 等の一次情報と照合したうえで、リンク切れ・番号不整合・時制の鮮度を修正しています。
（更新日：2026-07-15）

> 事実関係の確認結果：`Microsoft 365 E7（The Frontier Suite）`＝2026-05-01 GA、`Microsoft Agent 365` GA、
> 「2026年7月1日移行」（Copilot Studio / Foundry のセキュリティ機能が Agent 365 ライセンス必須へ、
> `AIAgentsInfo`→`AgentsInfo` 統合など）は、いずれも現行の公式ドキュメントと一致していました。

---

## 1. 修正した内容（このフォルダーに反映済み）

### A. リンク切れ・番号不整合の修正 — `07-observability.md`
観測ページを 08→07 へ番号変更した際に残っていた、旧番号向けの内部リンクを修正しました。

| 箇所 | 修正前（誤） | 修正後 |
| --- | --- | --- |
| ヘッダ／フッタのナビ | `[← Step 7：ガバナンス](./07-governance.md)`（存在しないファイル） | `[← Step 6：公開](./06-publish.md)` |
| ヘッダ／フッタのナビ | `[次：Step 9 セキュリティ →](./09-security.md)`（Step 8 を飛ばす） | `[次：Step 8 ガバナンス →](./08-governance.md)` |
| 冒頭 [!TIP] | `[Step 8 実習ラボ](./08-observability-lab.md)` | `[Step 7 実習ラボ](./07-observability-lab.md)` |
| AgentsInfo 節 | `[Step 7](./07-governance.md) の Kill Switch` | `[Step 8：ガバナンス](./08-governance.md) の Kill Switch` |

### D. 日付・時制の鮮度更新（本日 2026-07-15 時点で施行済みの事項）
「2026年7月1日以降 … されます／変わります」の**未来形**を、施行済みの**現在・完了形**に更新しました。

| ファイル | 修正前 | 修正後 |
| --- | --- | --- |
| `02-entra-agent-id.md` | 「検出経路が**変わります**」「一本化**されます**」 | 「検出経路が**変わりました**（本資料時点で施行済み）」「一本化**されました**」 |
| `09-security.md`（1.1） | 「提供され**なくなります**」 | 「提供され**なくなりました**」 |
| `09-security.md`（1.6） | 「Agent 365 に一本化**されます**」 | 「Agent 365 に一本化**されました**」 |

### B. 重複ファイルの除外
- **`08-observability.md` / `08-observability-lab.md` はこのフォルダーに含めていません。**
  これらは `07-observability.md` / `07-observability-lab.md` の**旧コピー**（更新日時が古く、README 目次・構成図にも未掲載）で、
  GitHub に上げると番号が重複・混乱するためです。

### C. 不足していた画像のプレースホルダ挿入 — `images/`
`09-security.md` が参照していたのに `images/` に**実体が無かった**画像に、リンク切れを防ぐ**中立的なプレースホルダ画像**を同名で追加しました。
当初 3 枚を追加しましたが、うち 2 枚は**後にユーザー指示で削除**（参照・ファイルとも／→ K）。**現在残っているプレースホルダは 1 枚のみ**です。

| ファイル名 | 状態 | 該当箇所 |
| --- | --- | --- |
| `09-purview-01-ai-observability.png` | **残（要差し替え）** | Purview › DSPM › AI 監視ページ（直近30日のエージェント一覧とリスクレベル） |
| `09-defender-03-advanced-hunting-alerts.png` | 削除済み | （旧）Defender › Advanced hunting |
| `09-entra-01-gsa-policy.png` | 削除済み | （旧）Entra › GSA ポリシー作成画面 |

### E. 認証ページの再構成（2軸化）— `05-authentication.md`
「**認可モデル（委任 OBO / アプリ専用 S2S）**」と「**実行トリガー（対話 / 自律）**」を**直交する2軸**として分離し、
旧・左右対比の1枚表を **2×2マトリクス**に置き換えました。特に「**自律実行だがユーザーの委任(OBO)を使う**」象限を明示。

- 根拠（一次情報）：
  - OBOフロー仕様に `refresh_token` grant があり *"asynchronous scenarios and background processes"*（非同期・バックグラウンド処理で利用可）と明記 → **自律×OBO は成立**。
  - *"Interactive flows aren't supported for any agent entity type"* → **どのエージェントも対話的サインイン（/authorize）はしない**。「対話/自律」は認可手法ではなく実行トリガー。
- 既存の正しい記述（`scp`/`roles`、監査記録主体、Observability経路、SDK/資格情報の推奨など）は保持し、2軸に沿って再配置しています。
- Observability経路（`/observability/` vs `/observabilityService/`）は「認可モデルそのもの」ではなく典型ホスティング2パターンの**実装差**である旨を注記しました。

### F. エージェント間（A2A）認証の節を追加 — `05-authentication.md`
「自律 × アプリ専用」の具体例として **A2A（agent-to-agent）** 節を新設。
**トークンはホップごとに取り直す**（Agent A→B は audience=B、B→C は audience=C の新規トークンをそれぞれ自 Agent ID で取得。リレー・書き換えはしない）ことと、
ID 管理者の 4 役割、OBO（委任）A2A との違いを明記。他 Step（02/04/07/08）へ相互リンク。

- 根拠（一次情報）：`agent-oauth-protocols`（Entra Agent ID）— *"All agent entities are confidential clients that can also serve as APIs for On-Behalf-Of scenarios"*、Agent identity は *"client_credentials for app-only autonomous operations"*。呼び出し先を audience とする app-only トークンを各エージェントが直接取得する構成と整合。

### G. 公開ページを2フェーズ構成へ再編 — `06-publish.md`
自前ホスト（サードパーティ／カスタム）エージェントの公開が **2 ステップ**（① マニフェスト作成 ＋ ② Agent 365 へアップロード登録）である点を**冒頭で明示**（表＋図＋Copilot Studio は自動登録で不要という注記）。
本文も、旧「Manifest 登録 → 一覧表示」の 1 節を **「① マニフェスト作成（`a365 publish`）」** と **「② Agent 365 へ登録（アップロード → 一覧表示）」** の 2 セクションに分割。

- 既存のコマンド・実行ログ・ウィザード画像（`06-publish-01〜05`）・注意書き（短縮名30字未満 / 反映5〜10分）はすべて保持し、各フェーズへ再配置しました。

### H. 観測・ガバナンスの削除依頼を反映（07-observability*.md / 08-governance.md）
- **`07-observability-lab.md`**：「**Step 6: 4画面の記録を突き合わせる（統合ワークシート）**」以降を削除（Step 6・Step 7〔意図的に「見えない」状態〕・トラブルシューティング・デモ講師向けメモ）。末尾のナビゲーションは保持。あわせて上部「ハンズオンシナリオ概要」の項目 6・7（削除ステップへの言及）も除去。
  - 「学習ゴール」の「Run が"見えない"ときに仮説を立てられる」（削除した Step 7 由来）も除去済み。
- **`07-observability.md`**：`[!TIP]` の補足ブロックを**全 7 箇所削除**（`[!NOTE]`／`[!IMPORTANT]`／`[!WARNING]` は保持）。冒頭の TIP には「実習ラボへの誘導」も含まれていたため、本編からラボへの案内文も併せて消えています（必要なら通常テキストで復活可能）。
- **`08-governance.md`**：`[!TIP]` の補足ブロックを**全 2 箇所削除**（冒頭の実習ラボ誘導／「停止と削除は別物」の要約。`[!NOTE]`(3)・`[!IMPORTANT]`(1) は保持）。こちらも冒頭 TIP のラボ誘導が併せて消えています。

### I. セキュリティ 1.3「リアルタイム保護（RTP）」を最新仕様に全面修正 — `09-security.md`
Microsoft Learn（`defender-security-for-ai` / `ai-agent-real-time-protection` / `ai-agent-detection-protection`）に基づき、誤りを修正して再構成しました。

- **誤りの訂正**：
  - 「Defender は Work IQ MCP と直接統合」→ **Work IQ MCP 統合は「Agent 365 のツール呼び出し」評価に限った話**であり、RTP 全体の仕組みではない点を明確化。RTP は**エージェントループ全体（プロンプト／ツール呼び出し／ツール応答）を検査し実行前にブロック**。
  - 「Copilot Studio は**モデルのプロンプト・応答を評価**して RTP 提供」→ 実際は **Copilot Studio の RTP は「ツール呼び出しの評価」**（Work IQ MCP 非依存）。プロンプト・応答評価は**別レイヤーの検出**（1.4）として切り分け。
  - 独自の 7 項目脅威リスト → 公式の **5 類型**（プロンプトベース攻撃／危険なツール使用／資格情報の露出／データ漏えい／異常な実行パターン）へ。
- **追記した要素**：カバー範囲表（Agent 365／Copilot Studio／ローカル）、**Default（監査）/ Custom（ブロック）ルール**、`BehaviorInfo` 記録、**監査時のみアラート発報／ブロック時は抑制**、**Prompt Shields（Foundry）・M365 Copilot Agent Builder のブロックも behavior 記録（Copilot Studio 未対応）**、ルール設定手順（Settings › Security for AI › Policies & rules › Real-time protection）、**Prompt evidence** 設定。
- 参考リンクに RTP 専用ページを追加。画像 `09-defender-02-security-for-ai-settings.png` は保持（キャプションの導線表記を Settings › Security for AI に調整）。

### J. L200デッキのスライド図を 09-security に追加（3枚）
L200 ピッチデッキ（日本語版）から書き出したスライド画像を、`09-security.md` の該当箇所へ図として追加しました（日本語表示も正常な実スライド）。

| 追加図（images/） | 出典スライド | 挿入位置 |
| --- | --- | --- |
| `09-defender-04-runtime-overview.png` | Defender の実行時 監視・ブロック（Monitor/Enforce・SOC/Sentinel） | **1. Microsoft Defender** 冒頭 |
| `09-purview-03-compliance-overview.png` | Purview のコンプライアンス（監査/DLM/コミュ/eDiscovery→リスクシグナル） | **2. Microsoft Purview** 冒頭 |
| `09-purview-02-data-security-flow.png` | Purview のデータ保護フロー（検知・ブロック・IRM・Defender XDR 相関） | **2.2 DLP** の直後 |
| `09-entra-02-agent-id-secure-access.png` | Entra：Agent ID ＋ 条件付きアクセスによるアクセス保護（リスク評価・許可/ブロック） | **3. Microsoft Entra GSA** 冒頭 |
| `09-defender-05-posture-management.png` | Defender：セキュリティポスチャー管理（露出評価・優先度スコア・修復の焦点化） | **1.6**（新設小節。旧 1.6「移行」は **1.7** へ繰り下げ） |

- ※当環境では日本語スライドを画像化すると文字化けするため、**実スライドは Hiroki さんに PowerPoint から書き出していただいたもの**を使用しています。
- ※Entra Agent ID の全体像スライドも受領・追加済み（セクション3冒頭）。当図は **Agent ID／条件付きアクセス（ID 層）** が主題で、GSA（ネットワーク層）は本節本文が扱う旨をキャプションで補足しています。トピック的に 08-governance の条件付きアクセス節へ移したい場合は対応可能です。
- ※**セキュリティポスチャー管理（#56）** はユーザー選択により追加（Defender 節に新小節 **1.6** を新設し、図・露出評価/優先順位付け/修復の 3 段の表を追加）。一方、**#55 脅威タクソノミー・#39 過剰権限・#79 製品スイート図は「不要」との判断**で見送り。

### K. 09-security 1.5 のサンプル KQL を削除
- **サンプル KQL の削除（1.5）**：「**アラートと構成情報を相関させるサンプル KQL**」（`AlertInfo` を `AgentsInfo` に `Title == AgentId` で join する例）を削除。Advanced Hunting テーブル一覧と直後の `[!NOTE]`（`AlertEvidence` 経由の紐付け／`Queries タブ › AI Agents` の事前構築クエリ推奨）は一般的な注意として保持。
- **プレースホルダ画像の削除**：`09-defender-03-advanced-hunting-alerts.png`（1.5）と `09-entra-01-gsa-policy.png`（3.2）の IMAGE PLACEHOLDER を、**本文参照・ファイルとも削除**。残プレースホルダは `09-purview-01-ai-observability.png`（2.5）の 1 枚のみ。

### L. 02 §2 の Copilot Studio ウォークスルーを 04 へ移動・再構成
02 は概念章のため、Copilot Studio の「ストア公開申請 → 承認 → Registry」実機ウォークスルーを **04 へ移動**。**02 §2 は「Copilot Studio エージェントの Entra Agent ID（実機例）」＝`cs-entra-01` 画面のみ**に縮小しました。

- **`04-register.md`**：先頭に **「Copilot Studio エージェントの登録（自動）」** を新設（`cs-01`〜`cs-07`・`cs-reg-01`〜`08` を移設）。目的を **① Copilot Studio（自動）→ ② 自前ホスト（a365）** の 2 パス構成に。既存の a365 フロー（A→B→C→D）は「② 自前ホスト」として後段に。
- **「公開(Publish)」用語の明確化**：Copilot Studio の「公開」＝ **ストア公開（Show in store / Submit to org catalog）＋管理者承認（Publish to store）** と定義し、**[Step 6] の「Agent 365 への公開（manifest アップロード）」とは別**である旨を `[!IMPORTANT]` で明記。
- **相互リンク整備**：02→04（実機手順）・04→02（Agent ID 画面）・04→06（公開の違い）。README 目次の Step 2/4 説明を振り替え（あわせて Step 5 説明も 2 軸表記へ整合）。
- **承認ウィザードは両方保持**：Copilot Studio 例（`cs-*`）と自前ホスト例（`04-approve-*`）は対象エージェントが異なる別キャプチャのため、それぞれの節に残置。
- ※移設した `cs-*` 画像は**作者の既存スクショ**（Hiroki さんの `images/` フォルダーに実在）で、参照先を 02→04 に付け替えただけです。追加のファイル配置は不要。

### M. 10-local-agent.md を Observe / Secure / Govern 構成へ全面再編（L200 デッキ準拠）
L200 ピッチデッキ（ローカルエージェント章）に沿って、独自の「6層モデル」を **Observe / Secure / Govern** の 3 本柱へ再編しました。

- **Observe**：発見・資産マッピング（デバイス／MCP／ID／クラウド）・リスク可視化（Defender for Endpoint／Purview／Registry）。
- **Secure**：実行時制御 **Inspect → Decide → Enforce**（Defender ＋ Purview のシグナルを統合し 許可／ブロック／監査、送信リクエスト・受信応答に強制、低確信度はクラウド／SOC）。※新規。
- **Govern**：**ブロック連鎖**（Intune 実行制限 → Defender 非準拠マーク → Entra 条件付きアクセスで遮断）と、**ガードレール付き許可**（Intune ＋ **MXC** の隔離コンテナー）。
- **「7 つの問い」**（旧「5 つの問い」を差し替え）と「**標準ガバナンスの外側・ユーザーの信頼境界の内側**」の一文を追加。
- 既存の型一覧（20 種類以上）・製品別対応表・提供状況は保持。

**用語の確定**：デッキの「MSX」は**誤記**。Windows Developer Blog（2026-06-02「Microsoft Execution Containers (MXC) SDK」早期プレビュー）と GitHub `microsoft/mxc` で確認し、**正しい表記「MXC」を維持**。参考リンクに両ソースを追加。

**図の追加**：Secure セクションに、L200 デッキ #67 のスライド図 `10-local-agent-02-secure-runtime.png`（Observe → Inspect → Decide → Enforce の実行時制御フロー）を Hiroki さん書き出しの実スライドとして追加。

### N. 10-local-agent.md：成熟度情報の削除＋Defender ランタイム保護の追記
- **成熟度情報を削除**（ユーザー指示）：早見表の「成熟度」列・凡例（◎/○/△/—）・提供状況／注意書きの成熟度参照を除去。見出しを「製品別の対応（早見表）」に。
- **Defender for Endpoint のランタイム保護を追記**（Learn: `ai-agent-runtime-protection-overview`）：Secure 節に —
  ① エージェントループの**3検査ポイント**（ユーザープロンプト／ツール前呼び出し／ツール後応答）、
  ② **2つの検査方式**（エージェントネイティブ・イベント検査〔Claude Code／Codex CLI／GitHub Copilot CLI・アプリ〕／ネットワーク検査〔証明書ピンニング・HTTP/3 は非対応〕）、
  ③ **適用モード**（ブロック／監査／無効・改ざん防止・まず監査推奨）、
  ④ 「不審な AI プロンプトインジェクション」アラート、
  ⑤ `.env` 流出の攻撃例、を新設。参考リンクに Learn 記事を追加。

### O. ロードマップ／将来計画に関わる記述を全ページから削除
公開資料に将来計画（roadmap）を含めない方針により、以下を除去・言い換えました（全ページで「ロードマップ／今後提供予定」＝0 件）。
- **10-local-agent.md**：冒頭 [!IMPORTANT]「ロードマップの機能」→「プレビュー機能」／Observe 表の「（ロードマップ）」削除／提供状況「ロードマップ／プレビュー」→「プレビュー」／末尾 [!IMPORTANT]「ロードマップの更新が速い／ロードマップで再確認」→「更新が速い／Microsoft Learn で再確認」。
- **08-governance.md**：ライフサイクル欄・[!NOTE] の「未使用エージェントの自動失効は Management Rule として**今後提供予定**」（将来機能）を削除し、**アクセスレビュー（Entra ID Governance）で運用**へ言い換え。
- **08-governance-lab.md**：同種の「今後提供予定」記述を削除し、アクセスレビュー運用へ言い換え。
- **02-entra-agent-id.md**：Registry Sync の「**将来スケジュール自動同期**」「**プレビュー時点ではスケジュール同期は未提供**」を削除（現状＝手動トリガーのみ記載）。
- **09-security.md**：GSA ブロック応答の既知課題に付いていた「**（改善予定）**」を削除。

> ※「プレビュー中／仕様は変わり得る」という一般的な注意書き（README 等）は**将来機能の告知ではない**ため残置。07 実習ラボの「今日の予定を教えて」は例文（スケジュール）で対象外。

---

## 2. あなたに手動で行っていただく作業

1. **Markdown の反映** — このフォルダー内の `*.md` を、リポジトリの同名ファイルへ上書きコピー。
2. **画像の反映** — このフォルダーの `images/` にある PNG を、既存の `images/` フォルダーへコピー（09-security 用に追加した **L200 スライド図 5 枚** ＋ **残プレースホルダ 1 枚**）。
   - 残プレースホルダは **`09-purview-01-ai-observability.png`**（Purview › DSPM › AI 監視ページ）の 1 枚のみ。後日、実際の画面キャプチャを**同じファイル名で上書き**してください（Markdown 編集は不要）。
3. **旧・重複ファイルの削除（要手動）** — リポジトリ／OneDrive 側に以下が残っている場合は削除してください。
   本ツールでは OneDrive/SharePoint 上のファイル削除ができないため、手動対応が必要です。
   - `08-observability.md`
   - `08-observability-lab.md`
4. **（任意）孤立画像の整理** — どのページからも参照されていない旧プレースホルダ画像が `images/` に残っています。
   気になる場合のみ削除してください（残っていても表示上の害はありません）。
   - 例：`06-publish-1.png`、`step3-*.png`、`step4-*.png`、`step5-*.png`、`step6-02-*.png`、`step6-03-*.png`、`step7-01/02/03-*.png` など

---

## 3. 参考（照合に使用した一次情報）

- Microsoft 365 E7 / Agent 365 GA（2026-05-01）… Microsoft 365 Blog「Microsoft 365 E7 and Agent 365 are now generally available」
- 2026年7月1日移行 … Microsoft Learn「Transition Microsoft Copilot Studio and Microsoft Foundry agent security capabilities to Microsoft Agent 365」
