# Step 7 実習ラボ — Agent 365 Observability を画面操作だけで体験する

[← 目次](./README.md) ｜ [Step 7：観測（本編）](./07-observability.md) ｜ [次：Step 8 ガバナンス →](./08-governance.md)

> [!NOTE]
> これは [Step 7：観測](./07-observability.md) の**実習用の補足教材**です。span のデータモデルや各画面の見方といった**概念の説明は Step 7 本編に譲り**、ここでは繰り返しません。本編を未読の場合は先にそちらを読んでから、このラボに進んでください。
> 本ラボは **CLI（`a365`）や SDK 設定を一切使わず、Copilot Studio で作成済みのエージェントに対する画面操作だけ**で完結します。自前ホスト型エージェントの Observability 設定（Exporter・ログレベル等）を扱いたい場合は Step 7 本編を参照してください。

---

## 概要

Copilot Studio で作ったエージェントは、公開すると **Observability のテレメトリ送信が自動的に有効**になります（開発者側の SDK 配線は不要）。このラボでは、そのエージェントを実際に1回動かし、生成された **1件の Run** を、

- **Microsoft 365 管理センター**
- **Microsoft Entra admin center**
- **Microsoft Purview**
- **Microsoft Defender**

の4画面で**同じ Run として突き合わせて追跡できる**ことを、自分の手で確認します。Step 7 本編が「各画面に何が表示されるか」の**辞書**だとすれば、このラボは「1つの Run を実際に4画面で追いかける」**実技演習**です。

---

## 学習ゴール

- Agent 365 管理センターの **Observe 面**（ダッシュボード → インベントリ → エージェント詳細 → アクティビティ → Agent Map）を一巡し、組織のエージェントの全体像と、対象エージェントの**ベースライン**を把握できる。
- Copilot Studio エージェントに1回発話し、そこから生まれる **Run の識別子**（`ConversationId` 等）を自分で控えられる。
- 同じ Run を、**M365 管理センター → Entra → Purview → Defender** の順に、画面操作だけで辿れる。
- 各画面で今見ている情報が **ログ／実行トレース／メトリクス** のどれに当たるかを判別できる。
- Run が特定の画面に「見えない」ときに、どこを疑えばよいか自分で仮説を立てられる。

---

## 前提条件

| 項目 | 内容 |
| --- | --- |
| **前提知識** | [Step 7：観測（本編）](./07-observability.md) を読了済み（span / `invoke_agent` / 3方向反映の関係を理解していること） |
| **対象エージェント** | Copilot Studio で作成し、Agent 365 に公開（Publish）済みのエージェントが1つ（[Step 4：登録](./04-register.md) 相当が完了していること） |
| **実行手段** | Teams または Microsoft 365 Copilot で当該エージェントに `@mention` できること |
| **ロール** | AI Reader（M365 管理センター）／Reports Reader（Entra）／Compliance Administrator + Content Viewer 相当（Purview）／Security Reader（Defender） |
| **開発環境** | 不要（CLI・Node.js・devtunnel などは使いません） |

---

## ハンズオンシナリオ概要

0. **Observe 面を一巡する** — 管理センターで ダッシュボード → インベントリ → エージェント詳細 → アクティビティ → Agent Map を確認し、組織のエージェントと対象のベースラインを把握する。
1. Copilot Studio エージェントに1回発話して **Run を1件**発生させ、時刻・質問内容を実習シートに記録する。
2. M365 管理センターの Agent inventory / Activity に反映されるのを確認する。
3. Entra admin center の Sign-in logs でエージェントの認証イベントを探し、識別子を控える。
4. Purview Activity explorer で同じ Run の Prompt/Response・ツール呼び出しを確認する。
5. Defender Advanced Hunting で KQL を実行し、`ConversationId` で同じ Run を横断的に照合する。
6. 4画面で控えた識別子・時刻を突き合わせ、「同一 Run である」ことをワークシートで確認する。
7. わざと条件を崩して（例：直近1日より前の Run を探す）、"見えない" 状態を体験し原因を推測する。

---

## ハンズオン手順

### Step 0: Observe 面を一巡して組織のエージェントを俯瞰する（オリエンテーション）

**目的**

- Run を追い始める前に、Agent 365 の Observe 面（見える化）をひと通り開き、「組織にどんなエージェントがいて、標準画面で何が見えるか」を把握する。あわせて、後続 Step で使う対象エージェントの **現在の状態（ベースライン）** を控える。

**操作手順**

1. [Microsoft 365 admin center](https://admin.microsoft.com/) にサインインし、左ナビの **Agents**（Agent 365）エリアを開く。
2. **① ダッシュボード** — 組織のエージェント全体のサマリー（総数・状態）を俯瞰する。
3. **② インベントリ（All agents）** — 一覧で対象の Copilot Studio エージェントを探し、名前・所有者・公開ステータス・ライフサイクル状態を確認する。
4. **③ エージェントの詳細** — 対象エージェントを選び、詳細（所有者・メタデータ・どの Blueprint のインスタンスか等）を開く。
5. **④ アクティビティ（ベースライン）** — 詳細内の **Activity** を開き、**今の時点の履歴件数**を控える（後続 Step の反映確認でベースラインとして使う）。
6. **⑤ Agent Map** — エージェントと ID・データ・ツール（MCP サーバー等）の関係性を可視化するビューを開き、対象エージェントの位置づけを確認する。

**実習シート⓪（オリエンテーション記録）**

| 項目 | 記入欄 |
| --- | --- |
| ダッシュボードのエージェント総数（概数） | |
| 対象エージェントの公開／ライフサイクル状態 | |
| Activity の現在の履歴件数（ベースライン） | |
| Agent Map に対象エージェントが表示されたか（Yes/No） | |

**期待される結果**

- ダッシュボード・インベントリ・詳細・Activity・Agent Map の5ビューをひと通り開けており、対象エージェントが登録済みエージェントとして各ビューに現れる。

**Observability確認ポイント**

- **メトリクス**：① ダッシュボードのサマリーと ⑤ Agent Map は、個々の Run ではなく **組織レベルの俯瞰**（全体像）を見る画面。
- **ログ／メトリクス**：④ Activity は、個々の実行履歴（ログ）と件数（メトリクス）の両面を持つ。ここでは **件数をベースライン** として控えておき、Step 2 で新しい Run が反映されたとき、ここから増えていることを見比べる。

> [!NOTE]
> ダッシュボード・Agent Map・各タブの正確なラベルや配置は **プレビューのため変更される可能性**があります。ナビゲーションが本書と異なる場合は、Observe 面（見える化）に相当するビューを探してください。反映には数分のタイムラグがあるため、待ち時間を有効に使うなら **Step 1 の発話だけ先に送信**しておくとよいですが、その場合でも **④ のベースライン件数は発話の反映が届く前（送信直後）に控えて**ください。スクリーンショットは `images/` の枠を各自の環境のキャプチャに差し替えます。

---

### Step 1: Copilot Studio エージェントに発話して Run を1件作る

**目的**

- 以降のすべての画面で追跡する「元データ」となる Run を、自分の操作で1件作る。

**操作手順**

1. Teams（または Microsoft 365 Copilot）を開き、対象の Copilot Studio エージェントとのチャットを開く。
2. `@エージェント名` でメンションし、ツール呼び出しを伴う質問を送る（例：「今日の予定を教えて」など、対象エージェントの機能に応じたもの）。
3. 応答が返るまで待つ。
4. 下の実習シートに記入する。

**実習シート①（自分で記入）**

| 項目 | 記入欄 |
| --- | --- |
| 発話した時刻（分単位） | |
| 送った質問内容（要約） | |
| エージェントの応答（要約） | |
| 使用したチャネル（Teams / Copilot など） | |

**期待される結果**

- エージェントから応答が返る。

**Observability確認ポイント**

- この時点ではまだどの画面も見ない。次の Step 以降で「今記入した時刻・内容」を手がかりに画面上のデータを探す。

---

### Step 2: M365 管理センターで反映を確認する

**目的**

- Run が管理センターの Agent inventory / Activity に**メトリクスとして**現れることを確認する。

**操作手順**

1. [Microsoft 365 admin center](https://admin.microsoft.com/) にサインインする。
2. **Agents › All agents**（Agent inventory タブ）を開き、対象エージェントを選択する。
3. 詳細パネルの **Activity** タブを開く。
4. Step 1 で記入した時刻に最も近い行を探す。

**期待される結果**

- Step 1 の発話に対応する行が Activity に表示される（反映まで数分かかる場合があるため、出ない場合は 1〜2 分待って再読み込みする）。

**実習シート②**

| 項目 | 記入欄 |
| --- | --- |
| Activity に表示された時刻 | |
| Step 1 の時刻との差（分） | |
| ベースライン（実習シート⓪）から件数が増えたか（Yes/No） | |

**Observability確認ポイント**

- ここで見ているのは「その Run が存在したかどうか」という**メトリクス（件数・状態）**であり、対話の中身（プロンプト内容）はここでは見えない。中身は Step 4 の Purview で確認する。

---

### Step 3: Entra admin center で Sign-in logs を確認する

**目的**

- 「対話の中身」ではなく「エージェントという ID がいつ認証されたか」という**ログ**を確認する。

**操作手順**

1. [Microsoft Entra admin center](https://entra.microsoft.com/) にサインインする。
2. **Agents › Agent identities** から対象エージェントを開く。
3. 左メニュー **Activity › Sign-in logs** を開く。
4. フィルタ **Agent type** を `Agent Identity` に、**Is Agent** を `Yes` に設定する。
5. Step 1 の時刻に最も近いイベントを選択し、詳細パネルを開く。

**期待される結果**

- `Is Agent: Yes` のサインインイベントが、Step 1 の発話時刻の前後数分以内に見つかる。

**実習シート③**

| 項目 | 記入欄 |
| --- | --- |
| Sign-in イベントの時刻 | |
| Correlation ID（詳細パネルより） | |

**Observability確認ポイント**

- ここは**認証イベントのログ**。何を話したかではなく「いつ・何のIDとして認証が起きたか」を確認している点を意識する。

---

### Step 4: Purview Activity explorer で対話の中身を確認する

**目的**

- Step 1 の発話内容・応答そのものを、**Prompt/Response** として実際に読む。

**操作手順**

1. [Microsoft Purview portal](https://purview.microsoft.com/) にサインインする。
2. **Agents**（または DSPM › AI observability）から **Activity explorer** を開く。
3. タブを **AI activities** に切り替え、Timestamp フィルタを Step 1 の時刻付近に絞り込む。
4. 該当する行（Activity type が `Copilot Interaction` または `Invoke Agent` のもの）を選択する。
5. 詳細パネルで **Prompt** と **Response** を開き、Step 1 で送った内容と一致するか確認する。

**期待される結果**

- Step 1 で送った質問がそのまま Prompt として、応答が Response として表示される。

**実習シート④**

| 項目 | 記入欄 |
| --- | --- |
| 表示された Activity type | |
| Prompt が Step 1 の記入内容と一致するか（Yes/No） | |
| Agent participant ID（Invoke Agent の場合） | |

**Observability確認ポイント**

- ここは M365 管理センターとは異なり、**対話内容そのもの（Prompt/Response）**を見ている。Prompt/Response が見えない場合は、閲覧に必要な **Content Viewer** 相当のロールが不足している可能性を疑う。

---

### Step 5: Defender Advanced Hunting で横断的に照合する

**目的**

- ここまで3画面で個別に確認してきた Run が、**同一の `ConversationId`** として Defender 側にも記録されていることを、KQL で照合する。

**操作手順**

1. [Microsoft Defender portal](https://security.microsoft.com/) にサインインする。
2. **Investigation & response › Hunting › Advanced hunting** を開く。
3. 次のクエリを実行し、直近の Agent 365 アクティビティを一覧表示する。

   ```kusto
   CloudAppEvents
   | where Timestamp > ago(1d)
   | where ActionType in ("InvokeAgent", "InferenceCall", "ExecuteToolBySDK", "ExecuteToolByGateway", "ExecuteToolByMCPServer")
   | extend AgentId = tostring(RawEventData.AgentId), ConversationId = tostring(RawEventData.ConversationId)
   | project Timestamp, ActionType, AgentId, ConversationId
   | order by Timestamp desc
   ```

4. Step 1 の時刻に近い行から `ConversationId` を1つ控える。
5. その `ConversationId` だけに絞り込み、実行順序を確認する。

   ```kusto
   CloudAppEvents
   | where tostring(RawEventData.ConversationId) == "<手順4で控えたConversationId>"
   | project Timestamp, ActionType, RawEventData
   | order by Timestamp asc
   ```

**期待される結果**

- 手順5の結果に `InvokeAgent` → `InferenceCall` →（あれば）`ExecuteTool...` の順で行が並ぶ。

**実習シート⑤**

| 項目 | 記入欄 |
| --- | --- |
| 控えた ConversationId | |
| ヒットした ActionType の並び順 | |

**Observability確認ポイント**

- ここでの `Timestamp` の並びが**実行トレース**そのもの（1 Run 内の span の時系列）。`summarize count() by ActionType` に変えれば、期間内の呼び出し回数という**メトリクス**にもなる（余力があれば試す）。

---

### Step 6: 4画面の記録を突き合わせる（統合ワークシート）

**目的**

- Step 2〜5 で個別に記録した内容が、すべて**同一の Run**を指していることを自分の目で確認し、「1つの Run が3系統・4画面にどう分岐して見えるか」を体感する。

**操作手順**

1. 実習シート①〜⑤を並べて見比べる。
2. 下の統合表を埋める。

**統合ワークシート**

| 画面 | 見えていた時刻 | 見えていた識別子 | ログ／トレース／メトリクスのどれか |
| --- | --- | --- | --- |
| M365 管理センター（Activity） | | — | |
| Entra（Sign-in logs） | | Correlation ID: | |
| Purview（Activity explorer） | | Agent participant ID: | |
| Defender（Advanced Hunting） | | ConversationId: | |

**期待される結果**

- 4行すべての時刻が、Step 1 で発話した時刻の前後数分以内に収まっている。

**Observability確認ポイント**

- 同じ Run でも、画面によって「何の識別子で管理されているか」（Correlation ID / Agent participant ID / ConversationId）が異なる点に注目する。Step 7 本編の識別子の対応表と照らし合わせておくと理解が深まる。

---

### Step 7: 意図的に「見えない」状態を体験する

**目的**

- Run が特定の画面に表示されない典型パターンを、実際に条件を崩して体験し、症状から原因を推測する力をつける。

**操作手順**

1. Defender Advanced Hunting で、**Observability の保持期間（30日）より前**の期間（例：`ago(60d)` 〜 `ago(31d)`）を指定してクエリを実行する。

   ```kusto
   CloudAppEvents
   | where Timestamp between (ago(60d) .. ago(31d))
   | where ActionType == "InvokeAgent"
   | count
   ```

2. 件数が 0、またはヒットしない場合、その原因を下の表から推測して選ぶ（自前ホストの CLI/Exporter 由来の原因は対象外）。

**トラブルシューティング演習（原因を推測してみる）**

| 症状 | 考えられる原因（画面操作のみで確認できるもの） |
| --- | --- |
| Defender で 30日以上前の Run がヒットしない | Observability データの**保持期間（30日）**を超えて自動削除されている（エージェントの存在・所有者情報などの棚卸しデータは180日残る点と混同しない） |
| Purview の Prompt/Response が見えない | 自分のロールに **Content Viewer** 相当が付与されていない |
| M365 管理センターの Activity に何も出ない | まだ反映のタイムラグ（数分）が経過していない、または `invoke_agent` に相当するイベントが発生していない（ツール呼び出しのみの発話だった等） |
| Entra Sign-in logs にエージェントのイベントが出ない | フィルタ **Agent type** / **Is Agent** の設定を誤っている |
| （応用）送信自体は成功しているはずなのに Defender の `CloudAppEvents` に出ない | テナントに MDA O365 コネクタが未接続でテーブル未生成、またはエージェントの ID が `AgentsInfo`（レジストリ）に未登録の可能性。詳細は [Step 7 本編 › Defender Advanced Hunting](./07-observability.md) を参照 |

**期待される結果**

- 保持期間を超えたクエリでは意図的に 0 件になることを確認できる（※ラボ用に新規作成したエージェントは、その時期にそもそも活動が無いため 0 件になる点も踏まえて考察する）。

**Observability確認ポイント**

- 「表示されない」＝「送信に失敗した」とは限らず、**保持期間・ロール・フィルタ設定**といった画面側の要因であることが多いと体感する。

---

## トラブルシューティング

| 症状 | 対処 |
| --- | --- |
| Step 2 の Activity に Run が出ない | 数分待って再読み込み。それでも出ない場合は Step 1 の発話が `invoke_agent` に相当するイベントを生成したか（応答が実際に返っているか）を確認する。 |
| Step 3 の Sign-in logs で見つからない | フィルタ **Agent type = Agent Identity**、**Is Agent = Yes** になっているか再確認する。 |
| Step 4 の Prompt/Response が非表示・グレーアウトしている | 自分のロールに Purview **Content Viewer** 相当が付与されているか管理者に確認する。 |
| Step 5 の KQL がヒットしない | `ago(1d)` の範囲を広げる（`ago(7d)` 等）。反映まで数分のタイムラグがあるため直後すぎる場合も外れることがある。 |
| 統合ワークシートの時刻が画面ごとに大きくずれる | 反映タイムラグの範囲内かを確認。数分を超えて大きくずれる場合は、複数回発話した別の Run を見ている可能性があるため、Step 1 からやり直す。 |

---

## デモ・講師向けメモ

**強調すべきポイント**

- Copilot Studio エージェントは**送信側の設定が一切不要**（Step 7 本編で扱う Exporter 設定・ログレベルは自前ホスト型のみに関係する）。参加者が「設定ファイルはどこ？」と迷わないよう、最初に明言しておく。
- 4画面はそれぞれ**別の識別子**（Correlation ID / Agent participant ID / ConversationId）で同じ Run を管理している。統合ワークシートを埋める作業自体が、この対応関係を体で覚える一番の近道。

**補足説明**

- 本ラボは Step 7 本編の「概念」を前提にしているため、概念を飛ばして本ラボだけを進めると識別子の意味で混乱しやすい。未読の参加者がいれば Step 7 本編の「識別子（Run を追跡するためのキー）」の表だけ先に共有する。

**デモ時のコツ**

- 反映タイムラグ（数分）があるため、Step 1 の発話は**演習の冒頭で先に実施**し、他の説明をしている間に反映を待つと時間を無駄にしない。
- Step 7 の「わざと見えなくする」演習は、参加者に強い印象を残しやすいので、時間が押している場合でも省略せず優先する。

---

[← Step 7：観測（本編）](./07-observability.md) ｜ [次：Step 8 ガバナンス →](./08-governance.md) ｜ [目次](./README.md)
