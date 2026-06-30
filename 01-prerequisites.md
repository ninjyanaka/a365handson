# Step 1 — Agent 365 を利用する前提条件

[← 00 概要](./00-overview.md) ｜ [← 目次](./README.md) ｜ [Step 2：Agent Registry / Entra Agent ID →](./02-entra-agent-id.md)

Agent 365 を使い始めるための、**組織側の前提（ライセンス・権限・対象 Agent・担当者）** を整理します。
（自前ホストで**開発**するための環境・ツール・3 レイヤーの考え方は [Step 3：サードパーティ管理](./03-third-party-management.md) にまとめています。）

---

## 1. ライセンス

| 区分 | 内容 |
| --- | --- |
| **基盤（必須）** | Microsoft 365 テナント |
| **ライセンス（必須）** | **Microsoft Agent 365**（評価可）／ **Microsoft 365 Copilot** ／ **Microsoft 365 E5**（または E7） |
| ネットワーク（任意） | Entra Internet Access / Global Secure Access（エージェントの通信制御を使う場合） |
| データ準備（推奨） | アクセスデータ・**秘密度ラベル**・**DLP ポリシー**の事前準備 |

> [!NOTE]
> Observability や Defender for AI などの保護機能も、上記ライセンス（Agent 365）が前提です。

---

## 2. 権限（RBAC / ロール）

Agent 365 を中核に、**Entra（ID）・Defender（監視）・Purview（データ保護）** の周辺 RBAC を**最小権限**で役割分担します。

| 担当 | 主な必要 RBAC | 概要 |
| --- | --- | --- |
| **統括管理** | **AI Administrator** | 可視化・承認・公開・ブロック・構成・ガバナンスの**中心ロール** |
| 閲覧・監査 | AI Reader ／ Reports Reader | 設定・利用状況・レポートの参照（読み取り専用） |
| Entra（ID/アクセス） | Conditional Access・Identity Governance・Agent ID・Agent Registry Administrator | 条件付きアクセス、アクセスパッケージ、Agent ID・レジストリ管理 |
| Defender（監視・対応） | Security Administrator ／ Reader | 脅威・アラート・インシデントの閲覧と対応 |
| Purview（データ保護） | Data Security AI Viewer ／ Content Viewer ほか | DSPM・DLP の参照、プロンプト/応答の確認、eDiscovery |

> [!TIP]
> **AI Administrator** ＝ エージェントの承認・公開・ブロック、アクセス制御、テナント全体の同意付与（Graph app 権限を除く）。
> ✕ 対象外：ユーザーライセンス/サインイン管理・Entra 構成・特権ロール割り当て（PIM）・高権限同意。

---

## 3. Agent 365 が対象とする Agent

| 対象 Agent | 扱い |
| --- | --- |
| **Copilot Studio（推奨）** | ネイティブで**自動登録**。PoC の中心に最適。 |
| Microsoft 365 Agents / Foundry | ネイティブ連携（自動登録）。 |
| **自前ホスト / SDK 連携**（本ワークショップ題材） | `a365` SDK で登録・管理（→ [Step 3](./03-third-party-management.md)）。 |
| 他クラウド 3P（Bedrock / Gemini / Agentforce / Databricks） | **Registry Sync** で可視化（プレビュー）。 |
| 端末上のローカル Agent | Defender / Intune / Purview で検出（Shadow AI 対策）。 |

> [!NOTE]
> 深い統制（条件付きアクセス・DLP 等）はネイティブ／SDK 連携が優位。右にいくほど「可視化中心」になります。

---

## 4. Agent 365 を利用する担当者（誰が・どんな役割）

エージェントのライフサイクル（**定義 → 構築 → 承認 → 運用・管理 → リタイア**）を、**開発者・IT 管理者・セキュリティ**の3者でエンドツーエンドに統制します。

![Step1 — 役割 × ライフサイクル](./images/01-roles-lifecycle.png)
*▲ AI エージェントのライフサイクルを、開発者・IT 管理者・セキュリティの3者で統制（従業員は承認済みエージェントを利用）*

| 担当者 | 主なポータル | 役割 |
| --- | --- | --- |
| **開発者**（ナレッジワーカー） | Copilot Studio / Foundry | エージェントを構築・公開。Sandbox / Production を選択。公開で **Agent ID が自動付与**。 |
| **IT 管理者**（AI Administrator） | Agent 365（M365 管理センター） | オンボード・**承認・公開**・ポリシー適用・監視・**無効化**。 |
| **セキュリティ管理者** | Microsoft Defender | 脅威検知・防御・調査（Advanced Hunting）。 |
| **コンプライアンス担当** | Microsoft Purview | DSPM・DLP・監査・eDiscovery。 |
| **ID 管理者** | Microsoft Entra | Agent ID・条件付きアクセス・アクセスレビュー（ID Governance）。 |
| **利用者**（従業員） | Teams / Copilot | 承認されたエージェントを利用。 |

> [!TIP]
> 意思決定層・AI 推進 / Copilot CoE は**概念中心でOK**（実機は任意）。既存の ID／セキュリティ／コンプラ体制に合わせて分掌します。

---

[← 00 概要](./00-overview.md) ｜ [Step 2：Agent Registry / Entra Agent ID →](./02-entra-agent-id.md)
