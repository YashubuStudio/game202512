# RPi5.md

## メインサーバ（Source of Truth）

### 概要

RPi5は **ゲーム世界における唯一の正（source of truth）** を持つ中央サーバである。
すべての状態確定・合法判定・ターン解決・ログ保存はここで行う。

---

## 主な責務（やること）

### 1. マッチ管理

* マッチ作成（human/human, human/AI, AI/AI）
* プレイヤー割当（side A/B）
* ターン管理（turn番号、deadline）
* 状態遷移管理（waiting / running / finished / aborted）

### 2. 状態管理（State）

* フルワールド状態の保持
* SQLiteへの保存
* seed管理（完全決定性）
* Fog of War 用のマスク生成

### 3. 指令管理（Command）

* `POST /command` の受信
* Commandの **検証・正規化**
* 不正Commandの拒否
* 締切までの上書き管理

### 4. ターン解決（Resolve）

* 同時手番の一括解決
* Resolveの全サブステップ実行
* 勝敗判定
* ログ生成（再現可能性確保）

### 5. AI / NPC 連携の制御

* N100 / RPi4 への **提案要求**
* 提案Commandの検証（AIでも例外なし）
* 採用／破棄の判断

### 6. Web API 提供

* クライアント用 REST API
* 観戦用API
* リプレイ取得API

---

## 提供API（外部）

### Public（インターネット公開）

* `POST /api/v1/matches`
* `GET  /api/v1/matches/{id}`
* `POST /api/v1/matches/{id}/command`
* `POST /api/v1/matches/{id}/resolve`
* `GET  /api/v1/matches/{id}/state`
* `GET  /api/v1/matches/{id}/replay`

### Private（LAN限定）

* `POST /internal/ai/think`
* `POST /internal/npc/act`
* `POST /internal/events/generate`

---

## 非責務（やらないこと）

* AIの探索・推論
* NPCの戦略決定
* 監視（Grafana等）
* 高頻度リアルタイム通信（WebSocket常設）

---

## 技術スタック

* Language: Go
* DB: SQLite（将来Postgres可）
* API: REST (HTTP/JSON)
* Deploy: Docker / docker-compose

---

## セキュリティ方針

* HTTPS前提
* 認証トークン必須（プレイ操作）
* 入力サイズ制限（JSON 64KB）
* レート制限
* AI/NPCエンドポイントは非公開

---
