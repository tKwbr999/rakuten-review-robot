# テーブル設計詳細

## 1. daily_rankings テーブル
### 概要
日次のランキングデータを管理するテーブル。1日1レコードを想定。

### カラム詳細
| カラム名 | データ型 | 制約 | 説明 |
|---------|----------|------|------|
| id | SERIAL | PRIMARY KEY | レコードの一意識別子 |
| fetch_date | DATE | NOT NULL, UNIQUE | ランキング取得日 |
| fetched_at | TIMESTAMP | NOT NULL | データ取得時刻 |

### 制約
- fetch_dateの一意性により、同一日付のデータ重複を防止
- fetched_atによる取得時刻の記録で、データの鮮度管理を実現

## 2. items テーブル
### 概要
商品情報を管理するテーブル。daily_rankingsテーブルと紐付け。

### カラム詳細
| カラム名 | データ型 | 制約 | 説明 |
|---------|----------|------|------|
| id | SERIAL | PRIMARY KEY | レコードの一意識別子 |
| daily_ranking_id | INTEGER | FOREIGN KEY | daily_rankingsテーブルの外部キー |
| item_code | VARCHAR(50) | NOT NULL | 商品コード |
| item_name | TEXT | NOT NULL | 商品名 |
| item_price | INTEGER | NOT NULL | 商品価格 |
| item_url | TEXT | NOT NULL | 商品URL |
| shop_name | VARCHAR(255) | NOT NULL | ショップ名 |
| medium_image_url | TEXT | NOT NULL | 商品画像URL |
| rank | INTEGER | NOT NULL | ランキング順位 |
| review_count | INTEGER | NOT NULL | レビュー数 |
| review_average | DECIMAL(3,2) | NOT NULL | 平均評価 |
| created_at | TIMESTAMP | NOT NULL DEFAULT CURRENT_TIMESTAMP | レコード作成日時 |

### 制約
- daily_ranking_idの外部キー制約により、参照整合性を保証
- item_codeは商品の一意識別子として使用

## 3. posting_history テーブル
### 概要
投稿履歴を管理するテーブル。itemsテーブルと紐付け。

### カラム詳細
| カラム名 | データ型 | 制約 | 説明 |
|---------|----------|------|------|
| id | SERIAL | PRIMARY KEY | レコードの一意識別子 |
| item_id | INTEGER | FOREIGN KEY | itemsテーブルの外部キー |
| posted_at | TIMESTAMP | NOT NULL | 投稿日時 |
| status | VARCHAR(20) | NOT NULL | 投稿ステータス |
| response_data | JSONB | NULL | 投稿レスポンスデータ |

### 制約
- item_idの外部キー制約により、参照整合性を保証
- statusは列挙型として扱い、定義された値のみ許可

## 4. データ整合性ルール
1. カスケード削除の制御
   - daily_rankings削除時のitemsテーブルの扱い
   - items削除時のposting_historyテーブルの扱い

2. NULL値の制御
   - 基本的に全てのカラムでNOT NULL制約
   - response_dataのみNULL許可

3. デフォルト値の設定
   - created_atはCURRENT_TIMESTAMPをデフォルト値として設定

## 5. データ型選択の根拠
1. VARCHAR vs TEXT
   - 固定長や制限が明確な場合はVARCHAR
   - 可変長で制限が不明確な場合はTEXT

2. 数値型
   - 金額はINTEGER（銭単位なし）
   - 評価値はDECIMAL(3,2)で小数点以下2桁まで対応