# Step 6 — Agent 365 への公開（manifest → Registry）

[← 目次](./README.md) ｜ [← Step 5：認証](./05-authentication.md) ｜ [次：Step 7 観測 →](./07-observability.md)

## 目的

**自前ホスト（サードパーティ／カスタム）エージェントを Agent 365 に公開**します。
Copilot Studio 製と違って**自動登録されない**ため、公開には次の **2 ステップ**が必要です。

| ステップ | やること | 成果物 |
| --- | --- | --- |
| **① マニフェスト作成** | `a365 publish` で **Agent Card Manifest** を生成し、公開に必要な項目（短縮名・説明・アイコン等）を整える | `manifest.zip` |
| **② Agent 365 へ登録** | その `manifest.zip` を **M365 管理センターにアップロード**し、公開ウィザードを進めて**レジストリ一覧に表示**させる | Registry 上のエージェント一覧 |

承認後に Instance を作成すると [Step 4](./04-register.md) の Entra Agent ID 付与につながります。

> [!NOTE]
> **この手順は自前ホスト（サードパーティ／カスタム）エージェント向けです。** Copilot Studio / Microsoft Foundry / M365 Agents は**自動登録**されるため、マニフェスト作成・アップロードは不要です（登録経路の違いは → [Step 3：登録経路](./03-third-party-management.md)）。

```
      ① マニフェスト作成                    ② Agent 365 へ登録
┌───────────────────────┐   ┌────────────────────────────────────┐
a365 publish ──▶ manifest 編集  ──▶  Add agent ──▶ 一覧表示 ──▶ 承認 ──▶ + Add instance
(manifest.zip)   (short<30字)        (アップロード)  (Name/Version)         (Agent ID 付与)
```

> ロール：管理センターのアップロードは **Global Administrator**（または Agent ID Administrator / Developer）／ Frontier preview 登録済みが前提。

---

## ① マニフェスト作成（`a365 publish`）

**Agent Card Manifest**（`manifest.zip`）を生成し、公開に必要な項目を整えます。

### 手順

1. **publish 実行**

   ```powershell
   a365 publish
   # manifest.json に blueprint ID を埋め、manifest.zip を生成
   # → 成功すると manifest フォルダに json と zip、
   #   管理センターへのアップロード手順が出力される
   ```

2. **manifest 編集** — `name.short`（**30 文字未満**）／ `description` ／ `icons` を確認・調整。再公開時は `version` を前回より大きく（例 `1.0.1`）。

### 実行ログ（マスク済み）

```console
PS C:\agent365-langchain-nodejs> a365 publish

Authentication context: user admin@<tenant>.onmicrosoft.com  (tenant <tenant-id>)

Extracting manifest templates...
    Extracted to     C:/path/to/agent365-langchain-nodejs/manifest
    Manifest updated: ...\manifest\manifest.json

Customize before packaging:
    version            - 再公開時は前回より大きく（例 1.0.1）
    name.short         - 30 文字以内（現在: "LangChain Teammate Blueprint"）
    name.full / description.* / developer.* / icons

Open manifest in your default editor now? (Y/n): n
Press Enter when you have finished editing the manifest to continue:

Package created:  C:/path/to/agent365-langchain-nodejs/manifest/manifest.zip

To publish:  https://admin.microsoft.com  >  Agents  >  All agents  >  Add agent
```

> [!NOTE]
> **短縮名は 30 文字未満。** マニフェストの `short name` が 30 文字以上だと publish が失敗します。

---

## ② Agent 365 へ登録（アップロード → 一覧表示）

生成した `manifest.zip` を管理センターにアップロードし、公開ウィザードを進めて**レジストリ一覧に表示**させます。

### 手順

1. **アップロード** — M365 管理センター › **Copilot Control System › Agents › All agents** の **… › Add agent** → `manifest.zip` を選択。
2. **公開ウィザードを進める**（**Upload agent → Publish to users → Review & finish**）。
3. **確認** — 名前・アイコン・ホスト製品を確認 → **5〜10 分後**に一覧表示（`Name` / `Version` / `Publisher` / `Availability`）。
4. **Instance 化** — 管理者承認 → **+ Add instance**（Blueprint → Instance。Entra Agent ID 付与は → [Step 4：登録](./04-register.md)）。

### アップロード → 公開（Publish）ウィザード（実機）

![① All agents › … › Add agent](./images/06-publish-01-add-agent.png)
*▲ ① M365 管理センター › Copilot › **Agents › All agents › Registry**。右上の **…** メニューから **Add agent**（＝カスタムエージェントの `manifest.zip` をアップロード）。*

![② manifest.zip をアップロード（Upload agent）](./images/06-publish-02-upload-manifest.png)
*▲ ② **Upload agent to publish** — `manifest.zip` を選択すると **Uploading and validating**（アップロード＆検証）が走る。*

![③ Publish to users（公開先の指定）](./images/06-publish-03-publish-users.png)
*▲ ③ **Publish to users** — Agent（例 `LangChain Teammate Blueprint`）／ Host products（**Copilot**）／ Publish（**All users**）を指定。**Activate（任意：インスタンスを作成できるユーザーの有効化）は None のまま Next**（有効化は → [Step 8：ガバナンス](./08-governance.md) で扱う）。*

![④ Review & finish → Publish](./images/06-publish-04-review-finish.png)
*▲ ④ **Review & finish** — Publish 先（All users）／ Activate（None）／ Policy template（未選択）を確認 → **Publish**。*

![⑤ 公開完了（You uploaded …）](./images/06-publish-05-done.png)
*▲ ⑤ 「You uploaded …」。Registry の一覧に登録される。**Close** で完了。*

> [!TIP]
> アップロード後、管理センターと Teams に表示されるまで **5〜10 分**。承認 → **+ Add instance** で Entra Agent ID が付与され、Blueprint→Instance が完結します。

---

## 参考

- [エージェントの公開（a365 publish）](https://learn.microsoft.com/microsoft-agent-365/developer/publish)
- [カスタムエージェントのアップロード（管理センター）](https://learn.microsoft.com/microsoft-365/copilot/agent-essentials/agent-lifecycle/agent-upload-agents)

[← Step 5：認証](./05-authentication.md) ｜ [次：Step 7 観測 →](./07-observability.md)
