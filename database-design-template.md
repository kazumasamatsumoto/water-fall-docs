# データベース設計書

## 文書管理情報

| 項目 | 内容 |
|------|------|
| 文書名 | 土地管理システムデータベース設計書 |
| 文書番号 | DB-001 |
| 版数 | 1.0 |
| 作成日 | YYYY/MM/DD |
| 最終更新日 | YYYY/MM/DD |
| 作成者 | 〇〇 〇〇 |
| 承認者 | □□ □□ |

## 1. データベース概要

### 1.1 データベース構成
- DBMS: PostgreSQL 15.0
- 文字コード: UTF-8
- 照合順序: C.UTF-8

### 1.2 データベースパラメータ
| パラメータ | 設定値 | 説明 |
|------------|--------|------|
| max_connections | 100 | 最大接続数 |
| shared_buffers | 4GB | 共有メモリバッファ |
| work_mem | 16MB | 作業用メモリ |

## 2. テーブル定義

### 2.1 テーブル一覧
| テーブル物理名 | テーブル論理名 | 概要 | 予想レコード数 |
|----------------|----------------|------|----------------|
| m_land | 土地マスタ | 土地の基本情報 | 100万件 |
| m_owner | 所有者マスタ | 土地所有者情報 | 10万件 |
| t_land_history | 土地履歴 | 土地の異動履歴 | 500万件 |

### 2.2 テーブル定義詳細

#### m_land（土地マスタ）
| カラム物理名 | カラム論理名 | データ型 | 桁数 | NULL | 主キー | 外部キー | 説明 |
|--------------|--------------|----------|------|------|---------|-----------|------|
| land_id | 土地ID | char | 10 | NO | YES | - | 土地を一意に識別するID |
| address | 所在地 | varchar | 200 | NO | - | - | 土地の所在地 |
| area | 面積 | decimal | 10,2 | NO | - | - | 土地面積（m²） |
| land_type | 地目 | char | 2 | NO | - | - | 地目コード |
| registration_date | 登記日 | date | - | NO | - | - | 登記年月日 |
| update_date | 更新日時 | timestamp | - | NO | - | - | レコード更新日時 |

#### m_owner（所有者マスタ）
| カラム物理名 | カラム論理名 | データ型 | 桁数 | NULL | 主キー | 外部キー | 説明 |
|--------------|--------------|----------|------|------|---------|-----------|------|
| owner_id | 所有者ID | char | 10 | NO | YES | - | 所有者を一意に識別するID |
| name | 氏名 | varchar | 100 | NO | - | - | 所有者氏名 |
| address | 住所 | varchar | 200 | NO | - | - | 所有者住所 |
| phone | 電話番号 | varchar | 20 | YES | - | - | 連絡先電話番号 |

## 3. インデックス定義

### 3.1 インデックス一覧
| テーブル物理名 | インデックス名 | カラム | 種類 | 説明 |
|----------------|----------------|---------|------|------|
| m_land | idx_land_01 | address | B-tree | 所在地検索用 |
| m_owner | idx_owner_01 | name | B-tree | 氏名検索用 |

## 4. パーティション定義

### 4.1 パーティション方針
- t_land_history: 年度別パーティション
- 保持期間: 10年分

### 4.2 パーティション定義詳細
```sql
CREATE TABLE t_land_history (
    history_id bigint,
    land_id char(10),
    change_date timestamp,
    change_type char(2),
    before_value text,
    after_value text
) PARTITION BY RANGE (date_trunc('year', change_date));

CREATE TABLE t_land_history_2024 PARTITION OF t_land_history
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

## 5. アクセス権限設定

### 5.1 データベースロール
| ロール名 | 説明 | 権限 |
|----------|------|------|
| land_admin | 管理者 | 全権限 |
| land_user | 一般ユーザー | SELECT, INSERT, UPDATE |
| land_readonly | 参照ユーザー | SELECT のみ |

### 5.2 テーブル権限設定
| テーブル物理名 | ロール | SELECT | INSERT | UPDATE | DELETE |
|----------------|--------|---------|---------|---------|---------|
| m_land | land_admin | ○ | ○ | ○ | ○ |
| m_land | land_user | ○ | ○ | ○ | × |

## 6. バックアップ・リカバリ設計

### 6.1 バックアップ方針
| 種類 | タイミング | 保存期間 | 方式 |
|------|------------|----------|------|
| フルバックアップ | 毎日深夜1時 | 1週間 | pg_dump |
| WALアーカイブ | 常時 | 1週間 | アーカイブモード |

### 6.2 リカバリ手順
1. ベースバックアップのリストア
2. WALアーカイブの適用
3. 整合性チェック

## 7. 性能対策

### 7.1 性能要件
| 処理内容 | 目標応答時間 | 同時アクセス数 |
|----------|--------------|----------------|
| 検索 | 3秒以内 | 100 |
| 更新 | 5秒以内 | 20 |

### 7.2 性能対策一覧
1. インデックス最適化
2. パーティショニング
3. 統計情報の定期更新
4. 自動VACUUMの設定調整