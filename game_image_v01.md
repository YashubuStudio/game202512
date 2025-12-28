## 0) 仕様固定の基本方針（v0.1）

* **RPi5が唯一の正（source of truth）**：状態確定、合法手判定、解決、ログ
* N100/RPi4は **提案のみ**：状態は絶対に確定しない
* **試合は完全決定性**：`seed` + `turn` + `consumed_rng` が一致すれば再現できる
* **外部公開前提のAPI**：認証、レート制限、入力サイズ制限、バージョニングを最初から入れる

---

## 1) ネットワーク構成（公開/内部の境界を固定）

### 1.1 公開されるのはRPi5のみ

* Public: **RPi5 API + Webクライアント配信**
* Private(LAN): RPi5 ⇄ N100 / RPi4

### 1.2 重要ルール

* N100/RPi4のエンドポイントは **インターネット非公開**
* RPi5→N100/RPi4の呼び出しは **mTLS or 共有トークン（最初は共有トークンでOK）**
* AI/NPC提案は **試合のseedを直接握らせない**（必要なら turn seed をRPi5が発行する）

---

## 2) APIの設計原則（Web公開のための“縛り”）

### 2.1 バージョニング

* RESTパスに固定：`/api/v1/...`
* JSONに `schema_version` を必須（将来互換）

### 2.2 認証（v0.1で決める）

公開を想定するなら最低限これを固定：

* **匿名観戦はOK**（read-only）
* **プレイはトークン必須**
* 方式：`Authorization: Bearer <token>`

  * v0.1は「ユーザー登録」不要でも良い（マッチ作成時にプレイヤートークン発行でもOK）

### 2.3 レート制限・入力制限（必須）

* `POST /command`：1プレイヤー1ターン1回（上書き可/不可を決める）
* JSON最大サイズ：例 **64KB**
* 1IPあたり：例 **60 req/min**（観戦APIは別枠）

---

## 3) Matchライフサイクル（状態遷移を固定）

`matches.status` はこれだけに固定（v0.1）：

* `waiting`（参加者揃い待ち）
* `running`（進行中）
* `finished`（終了）
* `aborted`（管理側停止、無効）

ターン単位の状態：

* `turn`: 現在ターン番号（0開始推奨）
* `deadline_at`: そのターンの締切（Simultaneous用）
* `submissions`: A/Bの提出状況

---

## 4) 指令（Command）仕様 v0.1 を“確定”

### 4.1 送信形式

* **JSON確定**
* 1ターンに1つ（上書き挙動は次で決める）

### 4.2 上書き挙動（Web公開では重要）

おすすめは **締切まで上書き可**：

* `POST /command` は同一ターン内なら上書きOK
* サーバは **最後に受理した正規化済みCommand** を採用
* 監査用に「提出履歴」は残しても良い（v0.2）

### 4.3 Commandの型（最終）

* `budget`：このターンに使ってよい上限（サーバが丸める）
* `priorities`：最大4
* `rules`：最大3
* `rng_salt`：0..2^31-1（多様性用、ログに保存）

**さらに固定するべき制約：**

* `weight`: 0..100
* `radius`: 1..6（重い探索を防ぐ）
* `target`: enum（`enemy_core`, `nearest_enemy`, `cell(x,y)`は v0.2）

---

## 5) Fog of War（情報設計を固定）

v0.1で「迷子になりやすい」ので、ここを決め打ちします。

### 5.1 視界ルール（MVP）

* 各ユニットは視界半径 `vision`（例 Scout=4 / 他=2）
* 視界計算はマンハッタン距離（軽量・決定的）
* 山は遮蔽しない（v0.2で遮蔽）

### 5.2 クライアントに返す情報（v0.1固定）

* `terrain`: 常時公開（固定地形は戦略の土台）
* `ore/heat/control/signal`: **視界内のみ**、視界外は `null`
* 敵 `units/structures`: **視界内のみ表示**（last_seenなし）
* 自軍の `units/structures`: 全て表示

これで「推定」「既知」などの複雑さは v0.2 へ先送りでき、MVPの実装が締まります。

---

## 6) Resolve（同時解決の衝突解決を固定）

### 6.1 サブステップ（v0.1確定）

1. validate & normalize（両者）
2. signal/jam 適用（命令受理可否）
3. priorities → task割当（決定的）
4. movement（最大1セル）
5. economy（採掘/発電/生産）
6. combat（射程1、同時適用）
7. heat update（行動量に比例）
8. price_index update（微小、上限±0.03/turn）
9. win check
10. turn log write

### 6.2 “同時”の決定性ルール

* 同率・同時衝突は **ユニットID順 + 乱数（seed派生）**で解消
* ただし乱数消費はログに残す（`rng_used_count` など）

---

## 7) 経済（市場・交換）をv0.1でどう固定するか

Web公開だと「経済が抜け道になる」ので、MVPは単純に固定します。

### 7.1 交換（トレード）をv0.1では無効化推奨

* `price_index` は将来のためにStateに持つが
* v0.1は **交換コマンド無し**（抜け道が増えやすい）
* 経済の面白さは「枯渇・熱・供給」で出す

※ 交換まで入れるなら、上限/手数料/回数制限が必須になるので v0.2 で。

---

## 8) ターン期限超過（Web運用のため固定）

v0.1決定案：

* 期限超過：**前ターンの正規化Commandを継続**
* 初手超過：サーバ既定 `default_command`（守備+省エネ）
* 連続超過の敗北は v0.2（荒らし対策）

---

## 9) データモデル（DBの“固定テーブル” v0.1）

ここは後から変更が重いので、最初から最小で確実に。

### 9.1 必須

* `matches(id, status, created_at, turn, deadline_at, seed, ruleset_json)`
* `match_players(match_id, side, player_id, role, token_hash, controller_endpoint)`
* `turns(match_id, turn, state_json, resolved_at, rng_used)`
* `submissions(match_id, turn, side, command_json, normalized_json, submitted_at)`
* `turn_logs(match_id, turn, log_json)`（デバッグ用）

`controller_endpoint` は AI/NPC の場合にのみ使用（humanはnull）

---

## 10) ノード間API（AI/NPC提案）を固定

### 10.1 N100: `POST /think`

入力：

* `match_id, turn, side, masked_state, time_budget_ms, nonce`
  出力：
* `command`（上で確定したCommand形式）
* `meta`（評価値など任意）

※ `masked_state` を渡すことで「AIが本来見えない情報を使う」問題を封じます（Web公開の公平性に重要）。

### 10.2 RPi4: `POST /events/generate`

* 入力：`match_id, turn, public_seed, nonce`
* 出力：`event_json`
* RPi5が event を採用するかはルールで決める（v0.1はイベント無しでもOK）

---

## 11) セキュリティ最低ライン（“公開”の必須項目）

v0.1で決めておくべき実務ルール：

* すべてHTTPS（リバプロ前提でもOK）
* `command_json` は **正規化して保存**（受理した形がログに残る）
* `GET /replay` は観戦用に公開してもいいが、**プレイヤー視界版と完全版を分ける**

  * 完全版（full state）は管理者のみ or 試合終了後のみ

---

# ここまでを v0.1 として「確定」すると、次にやるべき仕様固定はこれ

1. **Command kind の列挙を確定**（defend/mine/explore/attack など）
2. 各kindの `params` を確定（radiusやターゲット形式）
3. ユニット/施設の「能力値テーブル」を確定（HP/vision/cost/attack等）

---

# 全体設計上の重要ルール（共通）

```text
RPi5 = 法律・裁判所・台帳
N100 = 思考する参謀
RPi4 = 雑務・事務・イベント係
```

* **確定するのはRPi5だけ**
* 他ノードは「助言者」
* 助言が不正でも世界は壊れない

---
