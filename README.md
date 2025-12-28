# Robot Directive Strategy Game Platform
（ロボット指令型・非リアルタイム対戦ゲーム基盤）

## 概要

本プロジェクトは、**非リアルタイム（ターン制 / ステップ制）**で進行する  
**ロボット指令型 1v1 対戦ゲーム基盤**の実装を目的とする。

プレイヤーはロボットを直接操作せず、  
毎ターン「指令セット（ポリシー）」を提出し、  
サーバ側で同時解決・経済更新・戦闘解決が行われる。

AI対戦・人間対戦の両方に対応し、  
**複数PC（ノード）による分散構成**を前提とした設計となっている。

---

## 特徴

- 非リアルタイム（同時手番ターン制）
- ロボット指令型（ポリシー提出）
- 経済 × 戦術 × 情報戦（Fog of War）
- 完全決定性（seed + turn による再現可能性）
- Webブラウザ対応（REST API）
- AI / NPC を別ノードに分離可能
- リプレイ・デバッグ重視設計

---

## システム全体構成

```text
          [ Web Client ]
                |
                v
        ┌─────────────────┐
        │   RPi5 (Main)   │  ← 唯一の正
        │  Game Server    │
        └─────────────────┘
          |             |
          | LAN (REST)  | LAN (REST)
          v             v
 ┌─────────────┐   ┌─────────────┐
 │  N100 (AI)  │   │  RPi4 (NPC) │
 │  Thinking   │   │  Events     │
 └─────────────┘   └─────────────┘
````

---

## 各ノードの役割

### RPi5（メインサーバ / Source of Truth）

* マッチ管理
* 状態（State）の保持・確定
* 指令（Command）の検証・正規化
* ターン解決（Resolve）
* 勝敗判定
* ログ / リプレイ保存
* Web API 提供（公開対象）

詳細：`docs/RPi5.md`

---

### N100（AIノード）

* 指令セット（Command）生成
* 戦略探索・推論
* 難易度別AIの提供
* **状態の確定は行わない**

詳細：`docs/N100.md`

---

### RPi4（NPC / サブシステム）

* 軽量NPC制御
* 中立イベント生成
* 補助的・低負荷処理
* **提案のみ行い、確定しない**

詳細：`docs/RPi4.md`

---

## ゲーム仕様（v0.1 固定）

### 基本

* 対戦形式：1v1
* マップ：17×17 グリッド（ミラー対称）
* ターン制：同時手番
* ターン制限：60
* 勝利条件：

  * Core破壊
  * または最終スコア判定

### リソース

* Energy（行動・稼働）
* Material（建設・修理）
* Data（索敵・妨害）

### ロボット（MVP）

* Harvester（採掘）
* Scout（索敵）
* Striker（戦闘）

### 施設（MVP）

* Core（本拠地）
* Generator（発電）
* Factory（生産/修理）

---

## 指令（Command）モデル

* JSON形式
* 1ターン1指令（締切まで上書き可）
* 内容：

  * 予算（budget）
  * 優先度付きポリシー（priorities）
  * 制約ルール（rules）

すべての指令は **サーバ側で検証・正規化**される。

---

## 決定性とリプレイ

* 全試合は `seed` を持つ
* 乱数はすべて seed 派生
* Resolve は純粋関数的に設計
* 全ターンの state / log を保存
* 完全な再現・デバッグが可能

---

## API（概要）

### 公開API（RPi5）

* `POST /api/v1/matches`
* `POST /api/v1/matches/{id}/command`
* `POST /api/v1/matches/{id}/resolve`
* `GET  /api/v1/matches/{id}/state`
* `GET  /api/v1/matches/{id}/replay`

### 内部API（LAN限定）

* `POST /think`（AI）
* `POST /npc/act`
* `POST /events/generate`

---

## ディレクトリ構成（想定）

```text
.
├─ README.md
├─ docs/
│  ├─ RPi5.md
│  ├─ N100.md
│  └─ RPi4.md
├─ server/        # RPi5 (Go)
│  ├─ api/
│  ├─ core/
│  ├─ world/
│  ├─ resolve/
│  └─ storage/
├─ ai/            # N100
├─ npc/           # RPi4
└─ docker/
```

---

## 実装言語・技術

* メインサーバ：Go
* AI：Python（想定、拘束なし）
* NPC：Go
* DB：SQLite（将来PostgreSQL可）
* 通信：HTTP / REST
* デプロイ：Docker / Docker Compose

---

## 設計思想（重要）

* **確定するのは常に1箇所（RPi5）**
* AI/NPCは「助言者」
* 不正・暴走しても世界は壊れない
* Web公開しても安全な責務分離

---

## 開発ステータス

* [x] 全体アーキテクチャ確定
* [x] ノード責務分離
* [x] v0.1仕様固定
* [ ] Command kind 確定
* [ ] Resolve 実装
* [ ] Web UI 実装

---

## 次のステップ

1. Command kind & params の確定
2. ユニット/施設能力値テーブル作成
3. OpenAPI 定義
4. RPi5 最小ループ実装
5. AI/NPC ダミー接続
6. Web公開テスト

---

## ライセンス

TBD
