# Step 8 実習ラボ — ガバナンスをエンジニア視点で検証する（CA / Kill Switch / ライフサイクル）

[← 目次](./README.md) ｜ [Step 8：ガバナンス（本編）](./08-governance.md) ｜ [次：Step 9 セキュリティ →](./09-security.md)

> [!NOTE]
> これは [Step 8：ガバナンス](./08-governance.md) の **実技ラボ（エンジニア向け）** です。ポリシー継承・条件付きアクセス・ライフサイクルの**概念は本編に譲り**、ここでは繰り返しません。本ラボは「**設定したガバナンスが実際に効くか**」を、管理センターの画面だけでなく **Defender の KQL・Microsoft Graph・`a365` / `az` CLI・サインインログ**で検証する、一段エンジニア寄りの構成です。
> - Microsoft Learn のトレーニング「[エージェントを監視および管理する](https://learn.microsoft.com/ja-jp/training/modules/agent-365-monitor-manage/)」が**管理者向けのクリック操作中心**なのに対し、本ラボは**クエリ・API・CLI とログ検証**で「なぜ効くのか／どこで確認できるか」まで踏み込みます。
> - **プレビュー機能を多く含みます**（Agent risk 条件・未使用エージェントの自動失効など）。UI・コマンド・提供条件は変わり得ます。実施前に本編と Microsoft Learn で最新仕様をご確認ください。

---

## 概要

Agent 365 のガバナンスは **「見える化 → ルールで統制 → 効いていることを検証」** のループです。本ラボでは、1体のエージェント（自前ホスト型 or Copilot Studio）に対して次を**手を動かして**行い、**それぞれが実際に効くことをログ／クエリで確認**します。

- **棚卸し**：レジストリを UI ではなく **KQL（`AgentsInfo`）と Graph** で照会し、リスク・共有範囲・ツール保有で機械的に絞る。
- **アクセス制御**：条件付きアクセス（CA）を **Report-only → On** で適用し、**サインインログの Conditional Access 結果**で判定を読む。
- **Kill Switch**：Agent ID を Block して、**実トラフィック（LLM / MCP への egress）が止まる**ことを検証。ID レベルの遮断と実行プロセス停止の違いを切り分ける。
- **ライフサイクル**：Agent management rules による**一括統制**と、`a365 cleanup` による**完全削除＋orphan 検証**。
- **ガードレールの限界**：Agent ID を持たない（レジストリ同期のみ）エンティティには CA が効かないことを体験する。

> 統制の 3 本柱（ポリシー継承 / CA / ライフサイクル）と各ポリシーテンプレートの中身は [Step 8 本編](./08-governance.md) を先に一読してください。

---

## 学習ゴール

- レジストリ／リスクを **`AgentsInfo` の KQL** と **Graph** で照会し、統制対象を機械的に抽出できる。
- 条件付きアクセスを **Report-only で影響評価 → On** の順で適用し、**サインインログ**で許可・ブロックの判定根拠（Conditional Access 結果）を読み解ける。
- **Block（Kill Switch）が「ID としてのアクセス」を止めることと、実行プロセス停止（ACA 操作）が別物**であることを、ログと egress の挙動で説明できる。
- Blueprint 承認時の **`resourceConsents` が全 Instance に継承**される様子を確認できる。
- `a365 cleanup` によるリタイアと、**orphan アプリの残存確認**（`az ad app list`）まで実施できる。

---

## 前提条件

### ロール・ライセンス

| 対象 | 必要権限 / ライセンス |
| --- | --- |
| CA ポリシーの作成・カスタムセキュリティ属性 | **Global Administrator**（AI Administrator ではアクセスパッケージのみ可、CA・属性は権限不足） |
| 承認・公開・Block・Active 化 | **AI Administrator** または Global Administrator |
| Defender Advanced Hunting（`AgentsInfo` 等） | Advanced hunting 実行権限（Security Reader 以上） |
| サインイン／監査ログの閲覧 | 最低 **Reports Reader** |
| CA の対象化・ネットワーク制御 | Microsoft Entra ID **P1 / P2** ＋ ユーザーごとの **Agent 365 ライセンス**（ネットワーク制御は **Entra Internet Access**） |

### ツール（エンジニア向け）

- **`a365` CLI**（DevTools）／ **Azure CLI（`az`）** — 自前ホスト型のリタイア・orphan 確認に使用
- **Microsoft Defender portal** — Advanced Hunting（`AgentsInfo`）
- **Microsoft Entra admin center** — 条件付きアクセス／サインインログ／アクセスレビュー
- **Microsoft Graph（`/beta`）** — エージェント ID のサインイン照会（読み取り）

> [!TIP]
> 破壊的操作（`a365 cleanup`）を含みます。**必ず検証用テナント／サンドボックスのエージェント**で実施してください。本番エージェントでは Step 5 を「デモ提示」に切り替えます。

---

## ハンズオンシナリオ概要

0. **統制対象を棚卸しする** — `AgentsInfo`（KQL）と Graph で、リスク・共有・ツール保有からレビュー対象を抽出。
1. **ポリシー継承を確認する** — Blueprint 承認時の `resourceConsents` が全 Instance に継承されることを確認。
2. **条件付きアクセスで Kill Switch を用意する** — Agent risk = High を Block する CA を Report-only → On で適用。
3. **Kill Switch を検証する** — Block → サインイン Failure と egress 停止を確認。ID 遮断とプロセス停止を切り分ける。
4. **ライフサイクルを一括統制する** — Agent management rules（ownerless 再割当など）で「特定 → レビュー → 一括適用」を体験。
5. **リタイアする** — `a365 cleanup` で完全削除し、`az` で orphan を確認。Entra アクセスレビューも実施。
6. **ガードレールの限界を体験する** — Agent ID を持たない（同期のみ）エンティティに CA が効かないことを確認。

---

## ハンズオン手順

### Step 0: 統制対象を棚卸しする（KQL / Graph で機械的に）

**目的**

- レジストリを UI で眺めるのではなく、**クエリで「レビューすべきエージェント」を機械的に抽出**する（広く共有・機密権限・外部エンドポイント・所有者不在など）。

**操作手順**

1. [Microsoft Defender portal](https://security.microsoft.com/) › **Advanced hunting** を開く。
2. 最新状態のエージェント一覧を取得する（`AgentsInfo` は時系列スナップショットのため `arg_max` で最新化）。

   ```kusto
   AgentsInfo
   | summarize arg_max(Timestamp, *) by AgentId
   | where LifecycleStatus != "Deleted"
   | project AgentName, Platform, Owners, SharedWith, PublishedStatus, LifecycleStatus
   ```

3. **優先レビュー対象**を絞る（広く共有 × ツール保有 × 外部エンドポイント）。

   ```kusto
   // ツールを宣言し、かつ外部公開エンドポイントを持つエージェント
   AgentsInfo
   | summarize arg_max(Timestamp, *) by AgentId
   | where isnotempty(DeclaredTools) and isnotempty(Endpoints)
   | project AgentName, Platform, Owners, DeclaredTools, Permissions, Endpoints
   ```

   ```kusto
   // 所有者不在（ownerless）— Step 4 の一括再割当の候補
   AgentsInfo
   | summarize arg_max(Timestamp, *) by AgentId
   | where array_length(Owners) == 0
   | project AgentName, Platform, PublishedStatus, LastUpdatedDateTime
   ```

4. 管理センター **Agents › Overview › Top actions for you** の **Manage agent risks** と突き合わせ、Entra / Defender / Purview 由来の高リスクエージェントを確認する。

**期待される結果**

- レビュー対象が「名前で探す」ではなく**条件で抽出**できる。ownerless・広域共有・外部エンドポイント保有のリストが得られる。

**検証・確認ポイント**

- `Owners` / `Permissions` / `Endpoints` / `DeclaredTools` は **dynamic（JSON）列**。ダッシュボード化する場合は `mv-expand` / `tostring()` で必要プロパティだけを展開する。
- ここで得た **ownerless リストは Step 4**、**高リスク該当は Step 2 の CA** の入力になる。

> KQL の全文・列の意味は [Step 7：観測 › `AgentsInfo` の実用 KQL](./07-observability.md) を参照（本ラボでは再掲しません）。

---

### Step 1: ポリシー継承を確認する（承認時の resourceConsents）

**目的**

- Blueprint（テンプレート）承認時に同意した **Graph / Observability 権限が、そこから作られる全 Agent Identity（Instance）に継承**される「一度の承認で大規模スケール」を確認する。

**操作手順**

1. 管理センター **Agents › Registry › Requests** で保留中の **blueprint を承認（activate）** する。
2. 承認時に走る **同意フロー（`resourceConsents`）** で、要求される Graph / Observability 権限を確認する。
3. その Blueprint から **複数の Instance を作成**（[Step 4：登録](./04-register.md) の `+ Add instance`）する。
4. 各 Instance の Service Principal に、承認時の権限が**継承されている**ことを確認する（Entra › Enterprise applications、または Graph の `/servicePrincipals/{id}/appRoleAssignments` を読み取り）。

**期待される結果**

- 1 回の Blueprint 承認で、以後に作る Instance すべてが同一の権限セットを継承する。個別同意は不要。

**検証・確認ポイント**

- **ポリシーテンプレートは「新規の Active 化」にのみ適用**され、承認済みエージェントには後付けできない（本編の [!IMPORTANT] 参照）。継承の起点は Blueprint であることを押さえる。
- 継承された権限は Step 3 で Block したときに「何が止まるか」の範囲を決める。

> Blueprint と Instance の関係は [Step 2：Agent Registry / Entra Agent ID](./02-entra-agent-id.md) を参照。

---

### Step 2: 条件付きアクセスで Kill Switch を用意する（Report-only → On）

**目的**

- **Agent risk = High** のエージェントをブロックする CA ポリシーを作り、まず **Report-only** で影響を評価してから有効化する。

**操作手順**

1. [Microsoft Entra admin center](https://entra.microsoft.com/) › **Protection › Conditional Access** で新規ポリシーを作成する。
2. 次のように構成する（本編の例「Block - High Risky Agent」に対応）。

   | 設定 | 値 |
   | --- | --- |
   | Users / Target | **All agent identities**（対象＝エージェント ID） |
   | Target resources | All agent resources |
   | Conditions | **Agent risk (Preview) = High** |
   | Grant | **Block access** |
   | Enable policy | **Report-only** で作成 |

3. **Report-only** のまま、対象エージェントを何回か動かす。
4. **Sign-in logs › Service principal sign-ins** で対象エージェントのサインインを開き、詳細パネルの **Conditional Access** タブで、このポリシーが **「Report-only: Would block」** と評価されていることを確認する。
5. 影響が想定内であることを確認したら、ポリシーを **On** に切り替える。

**期待される結果**

- Report-only で「もし On なら Block されていた」ことがログで確認でき、On 化後は実際に Block される。

**検証・確認ポイント**

- **CA の対象・属性適用には Global Administrator が必要**（AI Administrator では不足）。
- CA が評価するのは **エージェント ID としての認証**。次の Step で「ID の遮断」と「実行プロセス停止」の違いを検証する。

**実習シート①（CA 評価）**

| 項目 | 記入欄 |
| --- | --- |
| ポリシー名 | |
| Report-only 時の評価結果（Would block / Not applied） | |
| On 化後のサインイン結果（Success / Failure） | |

---

### Step 3: Kill Switch を検証する（ID 遮断 vs プロセス停止）

**目的**

- Agent ID を **Block** したときに、**実トラフィック（LLM / MCP への egress）が止まるか**を検証し、**「ID としてのアクセス遮断」と「実行体プロセスの停止」は別物**であることを切り分ける。

**操作手順**

1. 管理センターで対象エージェントを **Block**（Kill Switch）する。または Step 2 の CA を On にする。
2. エージェントを動かし、**サインインログに Failure**（トークン取得／交換の失敗）が記録されることを確認する。
3. **egress の挙動**を確認する。
   - エージェントの出口（LLM / MCP 呼び出し）が **Agent ID（fmi_path）トークン**を使う構成なら、Block で **egress が遮断**され、応答生成が失敗する（＝キルスイッチ成立）。
   - 出口が SAMI / UAMI のままなら、**「ID としてのアクセス」は止まってもプロセスは動き続ける**。この場合の完全停止は **ACA（Container Apps）操作**の役割。
4. 調査後、Block を解除して復帰する（構成・データ接続は保持されている）。

**期待される結果**

- Block 後、対象 ID のサインインが Failure になる。egress が Agent ID トークン依存なら実トラフィックも停止する。

**検証・確認ポイント**

- **停止と削除は別物**：Block（Kill Switch）は構成・データ接続を保持したまま使用を停止し、調査後にそのまま解放できる。完全削除は Step 5 の `a365 cleanup`。
- サインインログ詳細の **Status / Conditional access / Failure reason** を控え、どのポリシーで止まったかを特定する。

**実習シート②（Kill Switch 検証）**

| 項目 | 記入欄 |
| --- | --- |
| Block 後のサインイン結果 | |
| egress（LLM / MCP）は停止したか（Yes/No） | |
| 出口トークンの種類（Agent ID / SAMI・UAMI） | |
| プロセス自体は停止したか（Yes/No＝ACA 操作の要否） | |

> ID 遮断とプロセス停止の関係（fmi_path 出口・キルスイッチ成立）は [Step 5：認証](./05-authentication.md) も参照。

---

### Step 4: ライフサイクルを一括統制する（Agent management rules）

**目的**

- 個別操作ではなく **Agent management rules** で「**条件に合致するエージェントを特定 → 実行前にレビュー → 一括適用**」の統制ループを体験する。

**操作手順**

1. 管理センター **Agents › Settings › Agent management rules** を開く。
2. 次のいずれかを実行する（現在提供のルール）。
   - **Reassign ownerless agents to manager** — Step 0 で抽出した **ownerless**（Agent Builder 製）を特定し、**前所有者のマネージャー**（Entra ID 階層）へ一括で所有権移管。
   - **Install Microsoft (1P) agents** — Microsoft 製エージェントをレビュー後に全ユーザーへ一括インストール。
3. **実行前レビュー**で対象一覧を確認してから適用する。
4. 適用後、Step 0 の ownerless KQL を再実行し、**対象が解消**されていることを確認する。

**期待される結果**

- ownerless エージェントの所有者が一括で埋まり、再クエリで件数が減る。

**検証・確認ポイント**

- **Agents › Settings** には他に **Allowed agent types**（Microsoft／自組織／外部発行元の可否）・**Security templates**・**Sharing**・**User access** もある。棚卸し方針に合わせて確認する。
- 未使用エージェントの退役は、**アクセスレビュー（Entra ID Governance）** で運用する。

---

### Step 5: リタイアする（cleanup ＋ orphan 検証）

**目的**

- 退役時に **完全削除**を行い、**Entra 側に orphan（残存アプリ）が残っていない**ことまで検証する。

> [!WARNING]
> `a365 cleanup` は **破壊的**（全 Agent 365 リソースを削除）です。**必ず検証用フォルダ／サンドボックスのエージェント**で実施してください。フォルダを取り違えると別エージェントを削除します。

**操作手順**

1. 作業ディレクトリを確認し、対象 Blueprint を特定してから削除する。

   ```powershell
   cd C:\path\to\target-folder                                  # ← フォルダを間違えない（最重要）
   Get-Content a365.generated.config.json | ConvertFrom-Json    # 対象 blueprint を確認
   a365 cleanup                                                 # 破壊的：全 Agent 365 リソースを削除
   ```

2. **orphan アプリの残存確認**（`a365 cleanup` 後に残ることがある）。

   ```powershell
   az ad app list --display-name "<blueprint名>" --query "[].{name:displayName, appId:appId}" -o table
   az ad app delete --id <orphanのappId>                        # 残っていれば手動削除
   ```

3. Entra **ID Governance › Access reviews** で、退役対象に紐づくアクセス（アクセスパッケージ等）のレビューを実施する。

**期待される結果**

- Agent 365 リソースが削除され、`az ad app list` で orphan が 0 件になる（残っていれば手動削除で 0 件化）。

**検証・確認ポイント**

- 削除後、Step 0 の `AgentsInfo` KQL で `LifecycleStatus == "Deleted"` に遷移していることを確認する（反映にタイムラグあり）。
- **退役 = 「停止（Block）」ではなく「削除（cleanup）＋アクセスレビュー」**。用途に応じて使い分ける。

---

### Step 6: ガードレールの限界を体験する（Agent ID が無いと CA は効かない）

**目的**

- **Agent ID を主体に持たない**エンティティ（レジストリ同期のみ、または素の Entra アプリ登録）には、CA / Purview / Defender の統制が**効かない**ことを体験し、「見えること」と「統制できること」の差を理解する。

**操作手順**

1. レジストリ同期のみ（Registry Sync）で可視化された **Unmanaged agent**、または素の Entra アプリ登録だけのエンティティを 1 つ用意する。
2. Step 2 の CA ポリシー（Agent risk = High → Block）の対象に含まれるかを確認する。
3. そのエンティティのアクセスを試み、**CA が適用されない／ブロックできない**ことを確認する。

**期待される結果**

- Agent ID を持たないエンティティは「一覧には見える」が、**CA の主体にならず統制できない**。統制には Agent ID の発行（[Step 4：登録](./04-register.md)）が前提だと分かる。

**検証・確認ポイント**

- これは「**まず見える化（Observe）→ Agent ID 発行 → 統制（Govern）**」という順序の必然性を示す。可視化だけでは統制は成立しない。
- サインインログで、そのエンティティが `Is Agent = No`（または Agent 型でない）として扱われていることを確認する。

---

## トラブルシューティング（設定したのに効かない）

| 症状 | まず疑う点 |
| --- | --- |
| CA を作ったのにエージェントに適用されない | 対象が **All agent identities** になっているか／**Agent 365 ライセンス**と **Entra P1/P2** が付与されているか／CA 作成に **Global Administrator** を使ったか |
| Report-only で「Not applied」 | 条件（Agent risk = High 等）に該当していない／プレビュー機能が未登録（Frontier テナント要件） |
| Block したのに応答が返り続ける | egress が **SAMI/UAMI のまま**で、ID 遮断はできてもプロセスが動いている（完全停止は **ACA 操作**） |
| Block したのに Defender の活動ビューに出続ける | Block 前に送信済みの**過去 Run が保持期間内に残っている**だけ（Block 失敗ではない。→ [Step 7 本編](./07-observability.md)） |
| `a365 cleanup` 後もアプリが残る | **orphan アプリ**。`az ad app list` → `az ad app delete` で手動削除 |
| ライフサイクルルールが対象を拾わない | ルールの条件（ownerless / 発行元 / タイプ）と実データの不一致。Step 0 の KQL で条件を再確認 |
| ownerless 再割当が効かない | 対象が **Agent Builder 製**か／前所有者に **Entra 上のマネージャー**が設定されているか |

---

## デモ・講師向けメモ

### 強調すべきポイント

- **「効くことを検証する」がエンジニア向けラボの主眼。** UI で設定して終わりにせず、**サインインログの Conditional Access 結果／`AgentsInfo` の再クエリ／egress の挙動**で必ず裏取りする。
- **ID 遮断 ≠ プロセス停止。** Block（Kill Switch）は「ID としてのアクセス」を止める。実トラフィックまで止めたいなら出口を **Agent ID（fmi_path）トークン**にしておく必要がある（さもなくば ACA 操作）。ここが受講エンジニアの一番の学び。
- **Report-only を必ず経由。** 本番 CA をいきなり On にせず、Report-only で「Would block」をログ確認してから On にする運用を体で覚えさせる。
- **順序の必然性**（Step 6）：Agent ID が無いと統制できない＝「Observe → Govern」の順が必然、という締めが効く。

### 補足説明

- **本ラボは Learn の管理者向けモジュールを補完する。** Learn（クリック操作）で全体像を掴み、本ラボで **KQL・Graph・CLI・ログ検証**まで踏み込む、という順番を案内する。
- **プレビュー機能が多い**（Agent risk 条件・自動失効）。ラベルや提供条件が変わり得る点を最初に明言する。

### デモ時のコツ

- **Step 0 の KQL を先に流しておく。** ownerless / 高リスクの実データが出ると、以降の CA・一括統制・cleanup の対象が具体化してストーリーが繋がる。
- **`a365 cleanup`（Step 5）は必ずサンドボックスで。** 本番デモでは「実行結果のスクショ提示」に切り替え、`cd` でフォルダを確認する所作を強調する。
- **サインインログの Conditional Access タブ**を全員で開く時間を取る。「なぜ止まったか」をログで説明できることが、エンジニア受講者の満足度に直結する。
- **スクリーンショット差し替え枠**：本編同様、公開時は `images/` の該当キャプチャを各自の環境のものに差し替える。

---

## 参考

- [Step 8：ガバナンス（本編・概念とテンプレート）](./08-governance.md)
- [エージェント向け条件付きアクセス](https://learn.microsoft.com/entra/identity/conditional-access/agent-id)
- [エージェントの管理・ライフサイクル アクション（管理センター）](https://learn.microsoft.com/microsoft-365/admin/manage/agent-actions)
- [エージェント テンプレート（ポリシー適用）](https://learn.microsoft.com/microsoft-agent-365/admin/agent-template)
- [組織のエージェント ID 管理（Entra Agent ID）](https://learn.microsoft.com/entra/agent-id/)
- [Defender Advanced Hunting（AgentsInfo：エージェント構成・ライフサイクル）](https://learn.microsoft.com/defender-xdr/advanced-hunting-agentsinfo-table)
- [トレーニング：エージェントを監視および管理する](https://learn.microsoft.com/ja-jp/training/modules/agent-365-monitor-manage/)

[← 目次](./README.md) ｜ [Step 8：ガバナンス（本編）](./08-governance.md) ｜ [次：Step 9 セキュリティ →](./09-security.md)
