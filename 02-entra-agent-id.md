# Step 2 — Entra Agent ID とは / 4 オブジェクトの関係 / Agent Registry

[← Step 1：前提](./01-prerequisites.md) ｜ [← 目次](./README.md) ｜ [Step 3：サードパーティ管理 →](./03-third-party-management.md)

[Step 1](./01-prerequisites.md) では「3 レイヤー」と「Blueprint→Instance」を概観しました。
ここでは **そもそも Entra Agent ID とは何か**、**4 つのオブジェクトの関係**、そして **Agent Registry** を掘り下げます。
（`a365` での具体的な作り方は [Step 3：サードパーティ管理](./03-third-party-management.md) で扱います。）

---

## 1. そもそも Entra Agent ID とは？

Microsoft Entra Agent ID は、**AI エージェントを Microsoft Entra 上で管理するための「新しい種類の ID」** です。
**ユーザー ID** とも **アプリケーション ID** とも異なります。

| ID の種類 | 特徴 |
| --- | --- |
| アプリケーション ID | 長期利用が前提。**安定性**が求められる。 |
| ユーザー ID | 資格情報・組織階層・メールなど**ユーザー属性**に紐づく。 |
| **Agent ID** | 自動化プロセスの中で**動的に作成**される ID。ワークフロー内で**1 日に何千回も作成・破棄**されうる。ユーザー認証はしないが、**ユーザーのように振る舞う**（＝OBO）シナリオもある。 |

> [!NOTE]
> Entra Agent ID は、既存の Entra ID 機能を**エージェントにも拡張**したものです。現時点で主に次の 5 つを実現します：
> ① Agent の登録と管理（Agent Registry）／ ② Agent ID の条件付きアクセス／ ③ ガバナンス（Governance Agent ID）／ ④ ID Protection（攻撃・不正エージェントの阻止）／ ⑤ セキュア ネットワーク アクセス（Global Secure Access for Agents）

---

## 2. 4 つのオブジェクトの関係（ここが混乱の山）

| オブジェクト | わかりやすく言うと |
| --- | --- |
| **Agent identity blueprint** | Agent ID を作成する**テンプレート**。認証・アクセス許可・アクティビティログ等の重要情報を含む。 |
| **Agent identity blueprint principal** | blueprint を**テナントに登録した時にできる実体**。実際にトークンを取得し、Agent ID を作成し、blueprint に代わって**監査ログに表示**される。 |
| **Agent Identity** | エージェント**1 体ごとの ID**。この ID で Microsoft サービスにアクセスする。 |
| **Agent's User Account** | Teams や Outlook など「ユーザーでないと動かないシステム」を使うときに必要。**Agent Identity と 1:1**。 |

```
                Agent identity blueprint  （テンプレート：認証/権限/監査の設計図）
                          │
            テナントに登録 │  ← ここで principal が作られる
                          ▼
        Agent identity blueprint principal  （実体：トークン取得・Agent ID 作成・監査の主体）
                          │
        ┌─────────────────┼─────────────────┐   ← 1 つの blueprint から最大 250 体
        ▼                 ▼                 ▼
   Agent Identity A   Agent Identity B   Agent Identity C
        │  （1:1）
        ▼
   Agent's User Account A   （Teams/Outlook 用。AI Teammate で払い出される）
```

> [!IMPORTANT]
> **テナント内のすべての Agent ID は、必ず Agent identity blueprint から作られます。**
> blueprint は「情報を保持するだけ」ではなく、`AgentIdentity.CreateAsManager` という特別な Graph 権限を持ち、**自分自身が Agent ID を作成する**主体でもあります。

---

## 3. Agent ID Blueprint の詳細

建物の設計図が配管・電気・構造まで含むように、**Agent ID Blueprint** は認証・アクセス許可・アクティビティログまで含む「設計図」です。
1 つの blueprint から作られた各エージェントは、**独自の ID・資格情報・権限**を持ちつつ、blueprint で定義された**共通特性を共有**します。

| 区分 | 内容 |
| --- | --- |
| 共有プロパティ | 説明 / アプリロール / 検証済み発行元 / 認証プロトコル設定（`OptionalClaims` 等） |
| **資格情報** | Agent ID が Entra からアクセストークンを要求する際に使う。blueprint に付与した OAuth 権限は、**そこから作られた全 Agent ID に付与**される。 |
| **必要なリソースアクセス** | エージェントが必要とする API/権限の**宣言**。同意レビュー時に管理者へ提示される。 |
| **継承可能なアクセス許可** | 管理者が blueprint principal に許可を付与すると、**組織内の全 Agent ID が自動で継承**する。 |

> [!TIP]
> **セキュリティを大規模にスケールできるのが blueprint の価値です。**
> - 条件付きアクセス ポリシーを **blueprint に対して**適用すると、そこから作られた全 Agent ID が対象になります。
> - **blueprint を無効化**すると、その全 Agent ID が認証できなくなります（一括停止）。

**blueprint principal の役割**：blueprint をテナントに追加すると必ず作られ、①トークン発行（トークンの `oid` が principal を指す）②監査ログ（blueprint の操作は principal が実行したものとして記録）③削除（principal を消すとそのテナントから blueprint を消去）を担います。

---

## 4. AI Teammate：instance 作成で Agent ID ＋ User が払い出される

ここが「Blueprint→Instance」の核心です。

- **blueprint はテンプレート**であって実体ではありません。
- 管理センターで **`+ Add instance`**（または Graph で `agentIdentity` を作成）した**瞬間**に、**Agent Identity**（Entra Agent ID）が払い出されます。
- さらに **AI Teammate** の場合は、Teams/Outlook など「ユーザーでないと動かない」面で動作するために、**Agent's User Account** が **Agent Identity と 1:1** で同時に払い出されます。

```
+ Add instance
     │
     ├──▶ Agent Identity（Entra Agent ID：例 12f560ef-…）     ← Microsoft サービスへのアクセス主体
     │
     └──▶ Agent's User Account（例 hana-assistant@tenant…）    ← Teams で @mention できる実体（AI Teammate）
```

instance 作成画面で入力する主な項目：

| 項目 | 内容 |
| --- | --- |
| Instance display name | Teams での表示名（透明性のため「Assistant」等を含めると良い：例 `Hana (Onboarding Assistant)`） |
| Agent Instance alias | agent user の UPN 前半（例 `hana-assistant` → `hana-assistant@tenant.onmicrosoft.com`） |
| Owner / Reports to（Sponsor） | instance の責任者。**Sponsor は必須**で、**User オブジェクト**を指定（SP やグループ不可）。 |
| License | agent user は実ユーザー扱い。ライセンス割り当てが必要になることがある。 |

> [!TIP]
> - instance 作成後、blueprint 詳細の **Entra agent ID が「—」から実値**に変わります。この値が Observability 送信時の `agentId` です。
> - **1 つの blueprint から最大 250 体**の Agent Identity を作成できます。
> - **Sponsor（スポンサー）** は Agent ID のライフサイクル（更新・延長・削除）の判断責任者。スポンサーが組織を離れると**自動的にマネージャーへ変更**されます。

---

## 5. Agent Registry（エージェント レジストリ）

**Agent Registry** は、組織で使えるすべてのエージェントを一元表示するインベントリです。
**M365 管理センター › Agents › All agents › Registry**、または **Entra 管理センター › Agent ID › Microsoft Entra Agent Registry** から確認します。

> 実機の画面は、下の **[Copilot Studio エージェントを例に：公開 → Registry → Entra](#copilot-studio-エージェントを例に公開--registry--entra)** で確認できます。

### 4 つのエージェント種別

| 種別 | 説明 |
| --- | --- |
| Microsoft エージェント | Microsoft が構築・保守。 |
| 外部パートナー構築エージェント | 信頼された外部開発元が公開。 |
| 組織によって発行された | 承認済みカスタム（LOB / 基幹業務）エージェント。 |
| 作成者が共有する | 個々のユーザー/開発者が作成・共有（Shared エージェント）。 |

### レジストリ概要・リスク

- **概要カウント**：エージェント合計数 / 所有者がいないエージェント / アンマネージド（A365 外で作成・管理）エージェント。
- **リスク列**：Entra・Defender・Purview の**高重大度リスクを集約**表示（シャドウエージェント、所有者未割当、過剰権限、プロンプトインジェクション、機密データアクセス、CA 違反、承認待ち など）。`レビュー` リンクから各セキュリティポータルへ。
- **フィルタ／アクション**：Status / Publisher / Channel（Copilot/Teams/Outlook/M365 アプリ/SharePoint）/ Platform でフィルタ。Refresh / Export（CSV・30 項目超）/ Add agent（manifest .zip アップロード）/ ピン留め / 列カスタマイズ / Graph API（プレビュー）。

> レジストリに登録するには各エージェントについて **エージェント インスタンス**（実行・管理に必要な操作情報）と **エージェント カード マニフェスト**（他のエージェント/アプリが発見・操作するための検出メタデータ）の 2 種類が必要です。登録経路は [Step 3：サードパーティ管理](./03-third-party-management.md) を参照。

---

## Copilot Studio エージェントを例に：公開 → Registry → Entra

**Copilot Studio で作ったエージェントは `a365`/manifest なしで自動的に Registry／Entra Agent ID に載ります**（自前ホストの題材との対比）。公開からレジストリ確認、Entra までを実機で追います。

### A. Copilot Studio から公開する

![① Show in the store](./images/cs-01-show-in-store.png)
*▲ ① Copilot Studio › Channels › Microsoft 365 and Teams › **Show in the store** →「Show to everyone in my org」（*Built by your org* に表示、Waiting for approval）。*

![② Submit to org catalog](./images/cs-02-submit-catalog.png)
*▲ ② Show in Teams app store for org → **Submit to org catalog**（組織カタログへ申請）。*

![③ Agent 365 の Requests に Pending review](./images/cs-03-requests.png)
*▲ ③ M365 管理センター › Agents › All agents › **Requests** に「Agent 365 Workshop Assistant」（Platform: Copilot Studio）が **Pending review** で出現。*

![④ エージェント詳細を確認](./images/cs-04-detail.png)
*▲ ④ 詳細（About / Overview）を確認。右上に Publish to store / Reject submission。*

![⑤ Select users（特定ユーザーに限定）](./images/cs-05-select-users.png)
*▲ ⑤ 承認ウィザード **Select users**。Specific users/groups で特定ユーザー（ハンズオン2/3/6/7 等）に限定して公開。*

> [!TIP]
> ⑥ **Apply template（ポリシー適用）** — ポリシーテンプレートで **Entra Agent ID Protection のリスクが High のとき自動ブロック**する条件付きアクセス（例「Block - High Risky Agent」）を有効化（→ [Step 7：ガバナンス](./07-governance.md)）。

![⑦ Review and finish → Publish](./images/cs-06-review-finish.png)
*▲ ⑦ **Review and finish**（Publish 先・Deploy・Policy template を確認）→ **Publish**。*

![⑧ Copilot から Add](./images/cs-07-copilot-add.png)
*▲ ⑧ ユーザーは Copilot の「Built by your org」でエージェントを開き、**Add** で Teams / Copilot に追加。*

### B. Agent Registry で確認する（タブ別）

![R1 詳細（Registry）](./images/cs-reg-01-detail.png)
*▲ Registry に登録（**合計 278 エージェント**）。詳細：Publisher type = **Platform · Copilot Studio** ／ Owner ／ Entra agent ID ／ Channel。タブ：Details / Users / Data & tools / Security / Permissions / Activity。*

![R2 Instructions](./images/cs-reg-02-instructions.png)
*▲ **Instructions** — エージェントの指示（システムプロンプト相当）を可視化。*

![R3 Data & tools](./images/cs-reg-03-data-tools.png)
*▲ **Data & tools** — Capabilities / Knowledge / **Tools**（Work IQ・Microsoft Learn Docs MCP 等）。*

![R4 Security](./images/cs-reg-04-security.png)
*▲ **Security** — Microsoft Purview（活動監視・機密データ保護・コンプラ評価）＋ Microsoft Entra（identity 保護・Agent ID）。右上に **Block**。*

![R5 Permissions](./images/cs-reg-05-permissions.png)
*▲ **Permissions** — 付与権限（Azure API Connections・Power Platform コネクタ等／Granted・Delegated）。*

![R6 Activity](./images/cs-reg-06-activity.png)
*▲ **Activity** — 利用インサイト（Active users 3 / Sessions 27 / Exceptions 0 / run-time）と時系列グラフ。*

![R7 Agent Map](./images/cs-reg-07-map.png)
*▲ **Agent Map** — 全エージェントをプラットフォーム別クラスタで可視化（Copilot Studio 14・Third-party・Foundry・Bedrock・Agentforce 等）。*

![R8 Map：ツール呼び出し](./images/cs-reg-08-tool-calls.png)
*▲ **Map ドリルダウン** — エージェント → ツール（MCP）の呼び出しを可視化（例：Microsoft Docs Search へ **67 tool calls**）。→ [Step 8：観測](./08-observability.md)*

### C. Entra での Agent ID

![Entra の Agent identity](./images/cs-entra-01-agent-identity.png)
*▲ Microsoft Entra 管理センター（[entra.microsoft.com](https://entra.microsoft.com/)）› Agents › Agent blueprints › 該当の **Agent identity**。Status: Active ／ Sponsors ／ Blueprint App ID ／ Object ID ／ Permissions・Entra roles ／ CA policies・Access packages。**Entra 側でガバナンスされる**。*

> [!NOTE]
> Copilot Studio 製は **Microsoft が自動で Registry／Entra Agent ID に載せる**ため、開発者は `a365` を使わずに「見える化・統制・保護」の対象になります（自前ホストとの対比 → [Step 3：サードパーティ管理](./03-third-party-management.md)）。

---

## 参考リンク

- [エージェント ID ブループリント（Microsoft Learn）](https://learn.microsoft.com/entra/agent-id/agent-blueprint)
- [Microsoft 365 管理センターのエージェント レジストリ（Microsoft Learn）](https://learn.microsoft.com/microsoft-365/admin/manage/agent-registry)
- [Microsoft Entra Agent ID 入門編（Zenn）](https://zenn.dev/microsoft/articles/a52eae77302ce7)

---

[← Step 1：前提](./01-prerequisites.md) ｜ [Step 3：サードパーティ管理 →](./03-third-party-management.md)
