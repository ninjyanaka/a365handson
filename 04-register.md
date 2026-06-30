# Step 4 — Agent 365 登録（Entra Agent ID 発行）

[← 目次](./README.md) ｜ [← Step 3：サードパーティ管理](./03-third-party-management.md) ｜ [次：Step 5 認証 →](./05-authentication.md)

## 目的

3P カスタムエージェントを **Agent 365 の管理下**に置き、**Entra Agent ID の付与**と **Block（Kill Switch）** までを確認します。

```
a365 setup all ──▶ a365 publish ──▶ 管理者が承認 ──▶ + Add instance ──▶ 管理下 / Block
 (--aiteammate)     (manifest.zip)   (Graph/Obs 同意)   (Entra Agent ID)   (レジストリ/Kill Switch)
```

> 担当・ポータル：**AI 管理者 ＝ M365 管理センター（Copilot Control System）**

> a365 コマンドの詳しい流れ・リファレンスは [Step 3：サードパーティ管理](./03-third-party-management.md) を参照。

---

## 演習 A：`a365 setup all --aiteammate` を通す

### 手順

1. **config 準備** — `a365.config.json` を AI Teammate 用に作成（`agentUserPrincipalName` / `agentUserDisplayName` の存在が AI Teammate 判定ポイント。通常 agent には無い）。
2. **setup 実行**

   ```powershell
   cd C:\path\to\agent365-langchain-nodejs
   a365 setup all --aiteammate
   ```

3. **generated config 確認**

   ```powershell
   Get-Content a365.generated.config.json | ConvertFrom-Json
   ```

   | 項目 | 確認内容 |
   | --- | --- |
   | `completed` | `True` であること（setup が完走した証拠） |
   | `agentBlueprintId` | blueprint の ID。`.env` の各 clientId/agentId と整合しているか |
   | `messagingEndpoint` | **devtunnel URL を指しているか**（古い/別物だと Teams から届かない＝最頻出の罠） |
   | `resourceConsents` | Microsoft Graph / Agent 365 Tools / Messaging Bot API / **Observability API** がすべて `consentGranted=True` か |

### 実行ログ（抜粋・マスク済み）

> [!IMPORTANT]
> 途中で **2 回、権限の承認**があります：**① アプリ権限（S2S）** の付与 `[y/N]: y`、**② 委任権限の管理者同意**（ブラウザでサインイン → ダイアログで **Allow**）。

```console
PS C:\agent365-langchain-nodejs> a365 setup all --aiteammate

Authentication context: user admin@<tenant>.onmicrosoft.com  (tenant <tenant-id>)

Checking requirements...
    Pass : Azure Authentication
    Warn : Frontier Preview Program — テナント登録は自動確認できません（事前に登録のこと）
    Pass : PowerShell Modules (Microsoft.Graph.Authentication, .Applications)
    Pass : Client App Configuration / 'wids' Optional Claim
    Requirements: 4 passed, 1 warning, 0 failed

... 〔中略：blueprint 作成・ディレクトリ伝播待ち〕 ...

Creating agent blueprint...
    Current user : System Administrator <admin@<tenant>.onmicrosoft.com>
    Display Name : LangChain Teammate Blueprint
    Blueprint application created successfully
        Blueprint ID : e722404b-…
    Creating blueprint client secret...
        Blueprint client secret : ****************************   〔redacted — 一度だけ表示〕
        Copy this value now - it will not be shown again automatically.

Configuring inheritable permissions...
    Microsoft Graph / Agent 365 Tools / Messaging Bot API / Observability API / Power Platform API  (kind=allAllowed)

# ── 権限の承認 ① アプリ権限（S2S） ───────────────────────────
Configuring application permissions...
    The following application permission will be granted to the agent blueprint:
        Observability API : Agent365.Observability.OtelWrite
    Assign this application permission now? [y/N]: y
    ✔ Application permissions granted.

# ── 権限の承認 ② 委任権限（管理者同意：ブラウザ） ─────────────
Configuring delegated permissions...
    Opening browser for admin consent...
    → Sign in and Accept the permission(s).
    ✔ Consent granted (All permissions).

Configuring messaging endpoint...
    Messaging endpoint URL : https://<your-tunnel>-3979.use2.devtunnels.ms/api/messages
    Registered successfully.

Updated: C:/path/to/agent365-langchain-nodejs/.env

... 〔中略〕 ...

Setup Summary
  1. Prerequisites                validated
  2. Azure hosting                skipped
  3. Blueprint                    created      'LangChain Teammate Blueprint' (ID: e722404b-…)
  4. Inheritable Permissions      configured
  5. Blueprint Permission Grants  granted      S2S app roles
  6. Messaging endpoint           registered
  7. Project settings             written

✔ Setup completed successfully
```

![Step4 — ② 委任権限の管理者同意ダイアログ（Allow / Deny）](./images/04-setup-consent.png)
*▲ ② 管理者同意。blueprint から作られる全エージェントに付与される権限（Mail / MCP / Telemetry 等）を確認して **Allow**。*

![Step3 — generated config の resourceConsents 確認](./images/step3-02-generated-config.png)
*▲ generated config で `resourceConsents` が全て `True` であることを確認*

> [!TIP]
> **Observability の 403 はここで決まる。** generated config の `resourceConsents` で Observability API（`9b975845-388f-...`）が `consentGranted=True` なら、その blueprint では 403 は起きません。**新しい blueprint を作るたびに同意は付け直し**になる点に注意。

---

## 演習 B：承認 — Requests タブから公開（Publish）する

`a365 publish` で登録すると、エージェントは **管理センター › Agents › All agents › Requests** タブに **`Pending review`** として現れます。IT 管理者がここで承認（＝ユーザーへ公開）して初めて利用可能になります。

![① Requests タブ（承認待ち一覧）](./images/04-approve-01-requests.png)
*▲ ① Agents › All agents › **Requests**。`Pending review`（レビュー待ち）/ `Pending activate`（有効化待ち）を確認。*

![② エージェント詳細（Publish to store / Reject submission）](./images/04-approve-02-detail.png)
*▲ ② エージェントを開く。`Publish to store` で承認、`Reject submission` で却下。この時点では Entra agent ID は「—」。*

承認すると **「Publish new agent」ウィザード（4 ステップ）** が開きます。

![③ Select users](./images/04-approve-03-select-users.png)
*▲ ③ **Select users** — インストールできるユーザー（All users / 特定）と事前インストール先を選択。Host products：Copilot / Teams。*

![④ Apply template（ポリシーテンプレート）](./images/04-approve-04-apply-template.png)
*▲ ④ **Apply template** — ポリシーテンプレートを適用。Custom で **Conditional Access**（例「Block - High Risky Agent」）・Access Package・Custom Security Attribute を選択（→ [Step 7：ガバナンス](./07-governance.md)）。*

![⑤ テンプレート変更の保存](./images/04-approve-05-template-changes.png)
*▲ ⑤ 既定テンプレートを変更した場合：このエージェントだけに適用（Continue without saving）か、新規テンプレートとして保存（Save as new template）。*

![⑥ Review permissions（Accept permissions）](./images/04-approve-06-permissions.png)
*▲ ⑥ **Accept permissions** — エージェントが要求する権限を確認（例では特別な権限なし）。*

![⑦ 公開処理中](./images/04-approve-07-publishing.png)
*▲ ⑦ **Review and finish** → `Publish`。ユーザー／グループへの公開処理が走る。*

![⑧ 公開完了](./images/04-approve-08-done.png)
*▲ ⑧ 「You published …」。公開先（All users）・デプロイ先・適用テンプレートが表示され、承認完了。*

> [!TIP]
> 承認（公開）が完了するとエージェントは `Pending review` から外れ、利用可能になります。続いて **演習 C（+ Add instance）** で Agent ID・User を払い出すと、Teams で使えるようになります。

---

## 演習 C：`+ Add instance` — Agent ID・User を払い出す

ここが **Agent ID 付与の瞬間**です。承認済みの blueprint を開き、**`+ Add instance`** を押して、エージェントを「使える実体」にします。

```
管理センター › Agents › blueprint を開く ─▶ [ + Add instance ] ─▶ 項目を入力 ─▶ 作成
                                                                         │
                          ┌──────────────────────────────────────────────┴───────────────┐
                          ▼                                                                ▼
        Agent Identity（Entra Agent ID）                         Agent's User Account（AI Teammate）
        blueprint 詳細の Entra agent ID が                        例 hana-assistant@tenant.onmicrosoft.com
        「—」→ 実値（例 12f560ef-…）に変化                        → Teams で @mention 可能（同期に数分）
```

### 作成画面で入力する項目

| 項目 | 内容 |
| --- | --- |
| **Instance display name** | Teams での表示名（透明性のため「Assistant」等を含めると良い：例 `Hana (Onboarding Assistant)`） |
| **Agent Instance alias** | agent user の UPN 前半（例 `hana-assistant` → `hana-assistant@tenant.onmicrosoft.com`） |
| **Owner / Reports to（Sponsor）** | instance の責任者。**Sponsor は必須**（User オブジェクト。SP・グループは不可） |
| **License** | agent user は実ユーザー扱い。ライセンス割り当てが必要になることがある |

### 画面の流れ（実機）

**① 管理センターから作成（メイン）**

![① Blueprint 詳細（作成前）— Entra agent ID は「—」](./images/04-add-instance-01-blueprint.png)
*▲ ① blueprint 詳細。右上の `+ Add instance` から作成。この時点では Overview の **Entra agent ID は「—」**。*

![② Add instance フォーム](./images/04-add-instance-02-form.png)
*▲ ② Instance display name（例 `Max (Onboarding Assistant)`）・Agent Instance alias・Owner/Reports to（Sponsor）・Instance location を入力。*

![③ Instance added（作成完了）](./images/04-add-instance-03-added.png)
*▲ ③ 「created successfully and is ready to use」。ライセンス（Microsoft 365 Frontier for AI teammates）が 1 つ消費される（例 17 of 25 available）。*

![④ Instance 詳細 — Entra agent ID が実値化](./images/04-add-instance-04-agentid.png)
*▲ ④ 作成された instance の詳細。**Entra agent ID が「—」から実値の GUID** に変わり、Owner も設定済み。*

**② 別ルート：エージェントストア（Built by your org）から**

![⑤ ストアから Create instance](./images/04-add-instance-05-store.png)
*▲ ⑤ Copilot のエージェントストア「Built by your org」→ blueprint の **Create instance**。*

![⑥ Create agent ダイアログ](./images/04-add-instance-06-create-dialog.png)
*▲ ⑥ Agent name・Alias・Managed by を入力して **Create**。*

![⑦ 作成中](./images/04-add-instance-07-creating.png)
*▲ ⑦ 「Your agent is being created」。完了すると通知が届く。*

**③ 使えることの確認（Teams）**

![⑧ Teams のチャット一覧に表示](./images/04-add-instance-08-teams-list.png)
*▲ ⑧ 作成した agent user（例 `Tsunoda / Max / Hana (Onboarding Assistant)`）が Teams の Chats に出現。*

![⑨ Teams で会話](./images/04-add-instance-09-teams-chat.png)
*▲ ⑨ `@mention` して会話できる（本物の agentic コンテキスト ＝ OBO・Observability が成立）。*

> [!TIP]
> 作成後、blueprint 詳細の **Entra agent ID が「—」から実値**に変わり、その値が Observability の `agentId` になります。
> あわせて **agent user** がテナントに現れ、**Teams で `@mention` して使える**ようになります（同期に数分かかることがあります）。

---

## 演習 D：管理下に置く → Block 確認

### 手順

1. **レジストリ確認** — M365 管理センター › **Copilot Control System › Agents › All agents**（発行元＝あなたの組織）。
2. **詳細を確認** — instance を選択 → 構成・権限・所有者（Owner / Reports to）・**Entra Agent ID の実値**を確認。
3. **Block 実行** — **Kill Switch** で使用停止。構成・データ接続は保持されたまま（協議ブロック／緊急ブロック）。
4. **解放で復帰** — クリア後に再有効化。必要なら権限是正 → 再承認。

![Step3 — All agents 一覧](./images/step3-03-all-agents.png)
*▲ 管理センター › Agents › All agents（レジストリ）*

![Step3 — Block / Kill Switch](./images/step3-04-block.png)
*▲ Block（Kill Switch）— 構成を保持したまま使用停止*

> [!TIP]
> **Block はレジストリからの即時停止（Kill Switch）。** 構成・データ接続を保持するため、調査後にそのまま解放できます。
> 完全削除（リタイア）は [Step 7：ガバナンス](./07-governance.md) の `a365 cleanup` で扱います。

---

## 確認チェックリスト

- [ ] `a365.generated.config.json` の `completed=True`
- [ ] `resourceConsents` が全て `consentGranted=True`（特に Observability API）
- [ ] `messagingEndpoint` が現在の devtunnel を指している
- [ ] 管理センター All agents にエージェントが表示される
- [ ] **Requests** タブで承認（Publish）した
- [ ] instance の **Entra Agent ID が実値**になっている
- [ ] **agent user** が作成され、Teams で `@mention` できる
- [ ] Block → 解放が想定どおり動く

---

## 参考

- [Microsoft Entra Agent ID とは](https://learn.microsoft.com/entra/agent-id/)
- [エージェントの公開（管理センター）](https://learn.microsoft.com/microsoft-agent-365/developer/publish)

[← Step 3：サードパーティ管理](./03-third-party-management.md) ｜ [次：Step 5 認証 →](./05-authentication.md)
