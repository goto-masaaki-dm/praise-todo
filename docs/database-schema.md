# データベーススキーマ設計書

## 概要

本ドキュメントでは、Praise Todoアプリケーションで使用するSupabase（PostgreSQL）データベースのスキーマ設計を定義します。Prisma ORMを使用してデータアクセス層を実装します。

## データベース情報

- **データベース**: Supabase（PostgreSQL）
- **ORM**: Prisma
- **接続方式**: Supabase接続文字列を使用

## Prismaスキーマ定義

### スキーマファイル構造

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

## テーブル設計

### 1. users（ユーザー）

ユーザー情報を管理するテーブル。

```prisma
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  avatarUrl String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  tasks       Task[]
  categories  Category[]
  tags        Tag[]
  achievements Achievement[]
  streaks     Streak[]
  settings    UserSettings?

  @@map("users")
}
```

**カラム説明**:
- `id`: ユーザーID（UUID）
- `email`: メールアドレス（一意）
- `name`: 表示名
- `avatarUrl`: アバター画像URL
- `createdAt`: 作成日時
- `updatedAt`: 更新日時

### 2. tasks（タスク）

タスク情報を管理するテーブル。

```prisma
model Task {
  id          String   @id @default(uuid())
  userId      String
  title       String
  description String?
  priority    Priority @default(MEDIUM)
  dueDate     DateTime?
  completed   Boolean  @default(false)
  completedAt DateTime?
  categoryId  String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  user       User        @relation(fields: [userId], references: [id], onDelete: Cascade)
  category   Category?   @relation(fields: [categoryId], references: [id], onDelete: SetNull)
  tags       TaskTag[]
  subtasks   Subtask[]
  notes      TaskNote[]

  @@index([userId])
  @@index([categoryId])
  @@index([completed])
  @@index([dueDate])
  @@map("tasks")
}

enum Priority {
  LOW
  MEDIUM
  HIGH
  URGENT
}
```

**カラム説明**:
- `id`: タスクID（UUID）
- `userId`: ユーザーID（外部キー）
- `title`: タスクタイトル
- `description`: タスクの説明
- `priority`: 優先度（LOW, MEDIUM, HIGH, URGENT）
- `dueDate`: 期限日時
- `completed`: 完了フラグ
- `completedAt`: 完了日時
- `categoryId`: カテゴリID（外部キー、オプション）
- `createdAt`: 作成日時
- `updatedAt`: 更新日時

### 3. categories（カテゴリ）

タスクのカテゴリを管理するテーブル。

```prisma
model Category {
  id          String   @id @default(uuid())
  userId      String
  name        String
  color       String?  // カラーコード（例: #FF5733）
  icon        String?  // アイコン名
  description String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  user  User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  tasks Task[]

  @@unique([userId, name])
  @@index([userId])
  @@map("categories")
}
```

**カラム説明**:
- `id`: カテゴリID（UUID）
- `userId`: ユーザーID（外部キー）
- `name`: カテゴリ名
- `color`: カラーコード
- `icon`: アイコン名
- `description`: 説明
- `createdAt`: 作成日時
- `updatedAt`: 更新日時

### 4. tags（タグ）

タスクのタグを管理するテーブル。

```prisma
model Tag {
  id        String   @id @default(uuid())
  userId    String
  name      String
  color     String?  // カラーコード
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  taskTags  TaskTag[]

  @@unique([userId, name])
  @@index([userId])
  @@map("tags")
}
```

**カラム説明**:
- `id`: タグID（UUID）
- `userId`: ユーザーID（外部キー）
- `name`: タグ名
- `color`: カラーコード
- `createdAt`: 作成日時
- `updatedAt`: 更新日時

### 5. task_tags（タスク-タグ関連）

タスクとタグの多対多の関係を管理するテーブル。

```prisma
model TaskTag {
  id     String @id @default(uuid())
  taskId String
  tagId  String

  task Task @relation(fields: [taskId], references: [id], onDelete: Cascade)
  tag  Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@unique([taskId, tagId])
  @@index([taskId])
  @@index([tagId])
  @@map("task_tags")
}
```

**カラム説明**:
- `id`: 関連ID（UUID）
- `taskId`: タスクID（外部キー）
- `tagId`: タグID（外部キー）

### 6. subtasks（サブタスク）

タスクのサブタスクを管理するテーブル。

```prisma
model Subtask {
  id          String   @id @default(uuid())
  taskId      String
  title       String
  completed   Boolean  @default(false)
  completedAt DateTime?
  order       Int      @default(0)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  task Task @relation(fields: [taskId], references: [id], onDelete: Cascade)

  @@index([taskId])
  @@map("subtasks")
}
```

**カラム説明**:
- `id`: サブタスクID（UUID）
- `taskId`: 親タスクID（外部キー）
- `title`: サブタスクタイトル
- `completed`: 完了フラグ
- `completedAt`: 完了日時
- `order`: 表示順序
- `createdAt`: 作成日時
- `updatedAt`: 更新日時

### 7. task_notes（タスクメモ）

タスクの詳細メモを管理するテーブル。

```prisma
model TaskNote {
  id        String   @id @default(uuid())
  taskId    String
  content   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  task Task @relation(fields: [taskId], references: [id], onDelete: Cascade)

  @@index([taskId])
  @@map("task_notes")
}
```

**カラム説明**:
- `id`: メモID（UUID）
- `taskId`: タスクID（外部キー）
- `content`: メモ内容
- `createdAt`: 作成日時
- `updatedAt`: 更新日時

### 8. achievements（実績・バッジ）

ユーザーの実績・バッジを管理するテーブル。

```prisma
model Achievement {
  id          String   @id @default(uuid())
  userId      String
  type        String   // 実績タイプ（例: "first_task", "streak_7", "complete_100"）
  title       String
  description String?
  icon        String?
  unlockedAt  DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, type])
  @@index([userId])
  @@map("achievements")
}
```

**カラム説明**:
- `id`: 実績ID（UUID）
- `userId`: ユーザーID（外部キー）
- `type`: 実績タイプ（一意の識別子）
- `title`: 実績タイトル
- `description`: 説明
- `icon`: アイコン名
- `unlockedAt`: 獲得日時

### 9. streaks（連続日数）

ユーザーの連続実行日数を管理するテーブル。

```prisma
model Streak {
  id            String   @id @default(uuid())
  userId        String
  currentStreak Int      @default(0)
  longestStreak Int      @default(0)
  lastActiveDate DateTime?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId])
  @@index([userId])
  @@map("streaks")
}
```

**カラム説明**:
- `id`: ストリークID（UUID）
- `userId`: ユーザーID（外部キー）
- `currentStreak`: 現在の連続日数
- `longestStreak`: 最長連続日数
- `lastActiveDate`: 最後にアクティブだった日付
- `createdAt`: 作成日時
- `updatedAt`: 更新日時

### 10. user_settings（ユーザー設定）

ユーザーの設定情報を管理するテーブル。

```prisma
model UserSettings {
  id                    String   @id @default(uuid())
  userId                String   @unique
  theme                 String   @default("light") // "light" | "dark"
  praiseOnComplete      Boolean  @default(true)
  praiseOnMilestone     Boolean  @default(true)
  praiseOnStreak        Boolean  @default(true)
  praiseDailySummary    Boolean  @default(true)
  praiseFirstTask       Boolean  @default(true)
  praiseAllCompleted    Boolean  @default(true)
  animationEnabled      Boolean  @default(true)
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("user_settings")
}
```

**カラム説明**:
- `id`: 設定ID（UUID）
- `userId`: ユーザーID（外部キー、一意）
- `theme`: テーマ（"light" | "dark"）
- `praiseOnComplete`: タスク完了時の賞賛を有効化
- `praiseOnMilestone`: マイルストーン達成時の賞賛を有効化
- `praiseOnStreak`: 連続日数達成時の賞賛を有効化
- `praiseDailySummary`: 1日の終わりのサマリーを有効化
- `praiseFirstTask`: 1日の最初のタスク完了時の賞賛を有効化
- `praiseAllCompleted`: 全てのタスク完了時の賞賛を有効化
- `animationEnabled`: アニメーションを有効化
- `createdAt`: 作成日時
- `updatedAt`: 更新日時

### 11. points（ポイント履歴）

ユーザーのポイント履歴を管理するテーブル（オプション）。

```prisma
model Point {
  id          String   @id @default(uuid())
  userId      String
  amount      Int      // 獲得または消費したポイント数
  reason      String   // ポイント獲得/消費の理由
  taskId      String?  // 関連するタスクID（オプション）
  createdAt   DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([createdAt])
  @@map("points")
}
```

**カラム説明**:
- `id`: ポイント履歴ID（UUID）
- `userId`: ユーザーID（外部キー）
- `amount`: ポイント数（正の値は獲得、負の値は消費）
- `reason`: 理由（例: "task_completed", "milestone_achieved"）
- `taskId`: 関連するタスクID（オプション）
- `createdAt`: 作成日時

## リレーション概要

### エンティティ関係図

```
User
├── tasks (1:N)
├── categories (1:N)
├── tags (1:N)
├── achievements (1:N)
├── streaks (1:1)
└── settings (1:1)

Task
├── user (N:1)
├── category (N:1, optional)
├── tags (N:M via TaskTag)
├── subtasks (1:N)
└── notes (1:N)

Category
├── user (N:1)
└── tasks (1:N)

Tag
├── user (N:1)
└── tasks (N:M via TaskTag)
```

## インデックス設計

### 主要インデックス

1. **users**
   - `email`: 一意インデックス（ログイン時の検索）

2. **tasks**
   - `userId`: ユーザーごとのタスク検索
   - `categoryId`: カテゴリ別のタスク検索
   - `completed`: 完了状態でのフィルタリング
   - `dueDate`: 期限順のソート

3. **categories**
   - `userId`: ユーザーごとのカテゴリ検索
   - `(userId, name)`: ユニーク制約

4. **tags**
   - `userId`: ユーザーごとのタグ検索
   - `(userId, name)`: ユニーク制約

5. **task_tags**
   - `taskId`: タスクに紐づくタグ検索
   - `tagId`: タグに紐づくタスク検索

6. **achievements**
   - `userId`: ユーザーごとの実績検索
   - `(userId, type)`: ユニーク制約

7. **streaks**
   - `userId`: ユーザーごとのストリーク検索

8. **points**
   - `userId`: ユーザーごとのポイント履歴検索
   - `createdAt`: 時系列でのソート

## データ整合性制約

### 外部キー制約

- すべての外部キーに`onDelete: Cascade`または`onDelete: SetNull`を設定
- ユーザー削除時は関連データも削除（Cascade）
- カテゴリ削除時はタスクのカテゴリをNULLに設定（SetNull）

### ユニーク制約

- `users.email`: メールアドレスの一意性
- `categories(userId, name)`: ユーザー内でのカテゴリ名の一意性
- `tags(userId, name)`: ユーザー内でのタグ名の一意性
- `achievements(userId, type)`: ユーザー内での実績タイプの一意性
- `user_settings.userId`: ユーザーごとに1つの設定のみ
- `streaks.userId`: ユーザーごとに1つのストリーク記録のみ

## マイグレーション戦略

### Prismaマイグレーション

1. スキーマファイル（`prisma/schema.prisma`）を編集
2. `pnpm prisma migrate dev --name <migration_name>`でマイグレーションを作成
3. マイグレーションファイルが生成され、データベースに適用される

### 初期マイグレーション

```bash
pnpm prisma migrate dev --name init
```

### 本番環境への適用

```bash
pnpm prisma migrate deploy
```

## ローカルストレージとの連携

### 同期戦略

- オフライン時のデータはローカルストレージに保存
- オンライン復帰時にSupabaseと同期
- 競合解決は「最後の書き込みが優先」または「タイムスタンプベース」で実装

### ローカルストレージの構造

```typescript
interface LocalStorageData {
  tasks: Task[];
  categories: Category[];
  tags: Tag[];
  lastSyncAt: string; // ISO 8601形式
}
```

## セキュリティ考慮事項

### Row Level Security (RLS)

SupabaseのRLSポリシーを設定して、ユーザーが自分のデータのみアクセスできるようにする：

- 各テーブルにRLSを有効化
- ユーザーIDベースのポリシーを設定
- 読み取り・書き込み・削除の権限を適切に設定

### データバリデーション

- Prismaスキーマレベルでのバリデーション
- アプリケーションレベルでの追加バリデーション
- 入力データのサニタイゼーション

## パフォーマンス最適化

### クエリ最適化

- 必要なカラムのみを取得（SELECTの最適化）
- JOINの適切な使用
- N+1問題の回避（Prismaの`include`や`select`を活用）

### キャッシュ戦略

- 頻繁にアクセスされるデータのキャッシュ
- React QueryやSWRなどのキャッシュライブラリの活用

## 今後の拡張予定

- チーム機能（複数ユーザーでの共有）
- タスクのアーカイブ機能
- データのエクスポート/インポート機能
- バックアップ・復元機能



