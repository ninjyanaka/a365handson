# Microsoft Agent 365 ハンズオン — Step 3〜7

自前ホスト型エージェントを **登録 → 認証 → 公開 → ガバナンス → 観測** まで実機で一気通貫に動かすハンズオンです。
各 Step を独立したページに分割しています。スクリーンショットは `images/` の差し込み枠を各自の環境のものに差し替えてください。

> 題材：**LangChain + Node.js 自前ホスト型 AI Teammate（ユーザー代理 = OBO）** / 3P・カスタムエンジン
> 出典：本人による実機検証（2026-06-29）＋ Microsoft Learn

---

## 目次

| Step | ページ | 内容 |
| --- | --- | --- |
| 前提 | [00-prerequisites.md](./00-prerequisites.md) | 3 レイヤーの理解 / Blueprint→Instance / 事前準備・ポート設計 |
| 深掘り | [01-entra-agent-id-deep-dive.md](./01-entra-agent-id-deep-dive.md) | **Entra Agent ID とは / Blueprint→Instance の中身 / `a365 setup all` の内部 / Agent Registry** |
| Step 3 | [step3-register.md](./step3-register.md) | Agent 365 登録（Entra Agent ID 発行）・管理下・Block |
| Step 4 | [step4-authentication.md](./step4-authentication.md) | 認証 2 パターン：委任(OBO) と 自律(S2S) |
| Step 5 | [step5-publish.md](./step5-publish.md) | Agent 365 への公開（manifest → Registry） |
| Step 6 | [step6-governance.md](./step6-governance.md) | 管理・ガバナンス（ポリシー / CA / ライフサイクル） |
| Step 7 | [step7-observability.md](./step7-observability.md) | 観測（ログ / 実行トレース / ツール呼び出し） |
| 付録 | [99-troubleshooting.md](./99-troubleshooting.md) | トラブルシュート早見表・クイックチェックリスト |

---

## このハンズオンの地図

```
前提 ──▶ Step3 登録 ──▶ Step4 認証 ──▶ Step5 公開 ──▶ Step6 ガバナンス ──▶ Step7 観測
(3層/        (Entra        (OBO /         (manifest /     (ポリシー /        (ログ /
 Blueprint→   Agent ID      S2S)           Registry)       CA / 停止・削除)   トレース /
 Instance)    発行)                                                          ツール呼び出し)
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
> 詳細は [00-prerequisites.md](./00-prerequisites.md) を参照してください。

---

## スクリーンショットの差し込み方

各ページには `![...](./images/stepN-XX-*.png)` の形で画像枠を用意しています。
`images/` フォルダ内の同名 PNG を、各自の環境のスクリーンショットに**上書き**するだけで反映されます（Markdown の編集は不要）。

```
agent365-handson/
├── README.md
├── 00-prerequisites.md
├── 01-entra-agent-id-deep-dive.md
├── step3-register.md
├── step4-authentication.md
├── step5-publish.md
├── step6-governance.md
├── step7-observability.md
├── 99-troubleshooting.md
└── images/            ← ここのプレースホルダを実際のキャプチャに差し替え
```

---

## 参考リンク（Microsoft Learn）

- [Microsoft Agent 365 概要](https://learn.microsoft.com/microsoft-agent-365/)
- [On-behalf-of フロー（Entra Agent ID）](https://learn.microsoft.com/entra/agent-id/agent-on-behalf-of-oauth-flow)
- [エージェント向け条件付きアクセス](https://learn.microsoft.com/entra/identity/conditional-access/agent-id)
- [エージェントの公開（管理センター）](https://learn.microsoft.com/microsoft-agent-365/developer/publish)
- [Observability の概念](https://learn.microsoft.com/microsoft-agent-365/developer/observability-concepts)
