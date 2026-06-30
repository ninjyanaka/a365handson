# Microsoft Agent 365 ワークショップ

Microsoft Agent 365 の理解を深めるためのワークショップです。Copilot Studio、自前ホスト型エージェントのライフサイクルを準備 → 登録 → 認証 → 公開 → 統制 → 観測 まで体験することが可能です。
各ページは番号順に読めば一直線に進みます。スクリーンショットは `images/` の差し込み枠を各自の環境のものに差し替えてください。

> **はじめての方は [00 概要](./00-overview.md) から読んでください。**

---

## 目次

| # | ページ | 内容 |
| --- | --- | --- |
| 00 | [概要](./00-overview.md) | **Agent 365 とは？**（初心者向け）・3 本柱・用語ミニ辞典 |
| 01 | [Step 1：前提](./01-prerequisites.md) | **利用前提**：ライセンス / 権限(RBAC) / 対象 Agent / 利用担当者（役割） |
| 02 | [Step 2：Agent Registry / Entra Agent ID](./02-entra-agent-id.md) | **Agent Registry**（一覧・各タブ・Copilot Studio 例）/ Entra Agent ID とは / 4 オブジェクトの関係 |
| 03 | [Step 3：サードパーティ管理](./03-third-party-management.md) | **開発の前提（3 レイヤー・環境）**・a365 CLI で自前 / 3P を管理下に・コマンド一式・`setup all` の内部・Registry Sync |
| 04 | [Step 4：登録](./04-register.md) | Agent 365 登録（Entra Agent ID 発行）・管理下・Block |
| 05 | [Step 5：認証](./05-authentication.md) | 認証 2 パターン：委任(OBO) と 自律(S2S) |
| 06 | [Step 6：公開](./06-publish.md) | Agent 365 への公開（manifest → Registry） |
| 07 | [Step 7：ガバナンス](./07-governance.md) | 管理・ガバナンス（ポリシー / CA / ライフサイクル） |
| 08 | [Step 8：観測](./08-observability.md) | 観測（ログ / 実行トレース / ツール呼び出し） |
| 99 | [付録](./99-troubleshooting.md) | トラブルシュート早見表・クイックチェックリスト |

---

## このワークショップの地図

```
00 概要 ─▶ 01 前提 ─▶ 02 Entra Agent ID ─▶ 03 サードパーティ管理 ─▶ 04 登録 ─▶ 05 認証 ─▶ 06 公開 ─▶ 07 ガバナンス ─▶ 08 観測 ─▶ 99 付録
 (とは？)   (利用前提)    (ID/Registry)        (開発前提/a365)        (発行)     (OBO/S2S)  (manifest)  (CA/停止削除)   (ログ/トレース)
```

## 前提条件（ライセンス・ロール・ツール）

| 区分 | 内容 |
| --- | --- |
| **ライセンス** | Microsoft 365 **E7**、または **E5 + Microsoft Agent 365**（Observability 権限の前提でもある） |
| **ロール** | 管理センターでの承認・instance 作成に **Global Administrator**（または Agent ID Administrator） |
| **ツール** | Node.js 18+ / Agent 365 DevTools CLI（`a365`）/ Azure CLI（`az`）/ devtunnel CLI |
| **前提プログラム** | テナントが **Frontier preview** に登録済みであること |

> [!IMPORTANT]
> Agent ID は **instance 化して初めて付与**されます。`blueprint` は「テンプレート」であって実体ではありません。
> 概念は [00 概要](./00-overview.md) と [Step 2](./02-entra-agent-id.md)、a365 での作り方は [Step 3](./03-third-party-management.md) を参照してください。

---

## スクリーンショットの差し込み方

各ページには `![...](./images/*.png)` の形で画像枠を用意しています。
`images/` フォルダ内の同名 PNG を、各自の環境のスクリーンショットに**上書き**するだけで反映されます（Markdown の編集は不要）。

```
agent365-handson/
├── README.md
├── 00-overview.md
├── 01-prerequisites.md
├── 02-entra-agent-id.md
├── 03-third-party-management.md
├── 04-register.md
├── 05-authentication.md
├── 06-publish.md
├── 07-governance.md
├── 08-observability.md
├── 99-troubleshooting.md
└── images/            ← ここのプレースホルダを実際のキャプチャに差し替え
```

---

## 参考リンク（Microsoft Learn）

- [Microsoft Agent 365 概要](https://learn.microsoft.com/microsoft-agent-365/)
- [エージェント ID ブループリント（Entra Agent ID）](https://learn.microsoft.com/entra/agent-id/agent-blueprint)
- [エージェント レジストリ（管理センター）](https://learn.microsoft.com/microsoft-365/admin/manage/agent-registry)
- [On-behalf-of フロー（Entra Agent ID）](https://learn.microsoft.com/entra/agent-id/agent-on-behalf-of-oauth-flow)
- [エージェント向け条件付きアクセス](https://learn.microsoft.com/entra/identity/conditional-access/agent-id)
- [Observability の概念](https://learn.microsoft.com/microsoft-agent-365/developer/observability-concepts)
