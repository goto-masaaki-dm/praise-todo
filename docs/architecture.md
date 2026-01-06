# アーキテクチャ設計書

## 概要

本ドキュメントでは、Praise Todoアプリケーションのクリーンアーキテクチャに基づく設計を詳細に説明します。

## アーキテクチャの原則

### クリーンアーキテクチャ

本プロジェクトは、Robert C. Martin（Uncle Bob）が提唱するクリーンアーキテクチャの原則に基づいて設計されています。

### 主な原則

1. **依存性逆転の原則（Dependency Inversion Principle）**
   - 外側の層は内側の層に依存する
   - 内側の層は外側の層に依存しない
   - 依存関係はインターフェースを通じて定義される

2. **関心の分離（Separation of Concerns）**
   - 各層は明確な責務を持つ
   - ビジネスロジックと技術的な詳細を分離

3. **単一責任の原則（Single Responsibility Principle）**
   - 各クラス・モジュールは1つの責任のみを持つ

4. **開放閉鎖の原則（Open-Closed Principle）**
   - 拡張に対して開いている
   - 修正に対して閉じている

## レイヤー構成

### レイヤー構造図

```
┌─────────────────────────────────────┐
│     Presentation Layer              │  ← UI, ページ, 状態管理
│  (React Components, Next.js Pages)   │
└──────────────┬──────────────────────┘
               │ 依存
┌──────────────▼──────────────────────┐
│     Application Layer               │  ← ユースケース, ビジネスロジック
│  (Use Cases, Interfaces, DTOs)      │
└──────────────┬──────────────────────┘
               │ 依存
┌──────────────▼──────────────────────┐
│     Domain Layer                     │  ← エンティティ, ドメインモデル
│  (Entities, Value Objects, Rules)    │
└──────────────┬──────────────────────┘
               │ 依存（インターフェース経由）
┌──────────────▼──────────────────────┐
│     Infrastructure Layer            │  ← データベース, 外部サービス
│  (Prisma, Supabase, Repositories)   │
└─────────────────────────────────────┘
```

## 各レイヤーの詳細

### 1. Domain層（ドメイン層）

#### 責務

- ビジネスルールの定義
- エンティティと値オブジェクトの定義
- ドメインモデルの実装

#### ディレクトリ構造

```
src/domain/
├── entities/           # エンティティ
│   ├── Task.ts
│   ├── User.ts
│   ├── Category.ts
│   └── Achievement.ts
├── value-objects/      # 値オブジェクト
│   ├── Priority.ts
│   ├── TaskId.ts
│   └── UserId.ts
├── models/            # ドメインモデル
│   ├── TaskProgress.ts
│   └── Streak.ts
└── rules/            # ビジネスルール
    ├── TaskCompletionRule.ts
    └── AchievementUnlockRule.ts
```

#### 実装例

**エンティティ（Task.ts）**

```typescript
export class Task {
  constructor(
    public readonly id: TaskId,
    public readonly userId: UserId,
    public title: string,
    public description: string | null,
    public priority: Priority,
    public dueDate: Date | null,
    public completed: boolean,
    public completedAt: Date | null,
    public readonly createdAt: Date,
    public updatedAt: Date
  ) {}

  complete(): void {
    if (this.completed) {
      throw new Error('Task is already completed');
    }
    this.completed = true;
    this.completedAt = new Date();
    this.updatedAt = new Date();
  }

  updateTitle(newTitle: string): void {
    if (newTitle.length === 0) {
      throw new Error('Title cannot be empty');
    }
    if (newTitle.length > 200) {
      throw new Error('Title cannot exceed 200 characters');
    }
    this.title = newTitle;
    this.updatedAt = new Date();
  }
}
```

**値オブジェクト（Priority.ts）**

```typescript
export enum Priority {
  LOW = 'LOW',
  MEDIUM = 'MEDIUM',
  HIGH = 'HIGH',
  URGENT = 'URGENT'
}

export class PriorityValue {
  constructor(private readonly value: Priority) {
    if (!Object.values(Priority).includes(value)) {
      throw new Error(`Invalid priority: ${value}`);
    }
  }

  getValue(): Priority {
    return this.value;
  }

  getPoints(): number {
    switch (this.value) {
      case Priority.LOW:
        return 10;
      case Priority.MEDIUM:
        return 20;
      case Priority.HIGH:
        return 30;
      case Priority.URGENT:
        return 50;
    }
  }
}
```

### 2. Application層（アプリケーション層）

#### 責務

- ユースケースの実装
- ビジネスロジックの調整
- インターフェースの定義
- DTOの定義

#### ディレクトリ構造

```
src/application/
├── use-cases/         # ユースケース
│   ├── tasks/
│   │   ├── CreateTaskUseCase.ts
│   │   ├── UpdateTaskUseCase.ts
│   │   ├── DeleteTaskUseCase.ts
│   │   └── CompleteTaskUseCase.ts
│   ├── achievements/
│   │   └── CheckAchievementsUseCase.ts
│   └── progress/
│       └── CalculateProgressUseCase.ts
├── interfaces/        # インターフェース（リポジトリなど）
│   ├── repositories/
│   │   ├── ITaskRepository.ts
│   │   ├── IUserRepository.ts
│   │   └── IAchievementRepository.ts
│   └── services/
│       └── IPraiseService.ts
└── dto/              # データ転送オブジェクト
    ├── CreateTaskDTO.ts
    ├── UpdateTaskDTO.ts
    └── TaskResponseDTO.ts
```

#### 実装例

**ユースケース（CreateTaskUseCase.ts）**

```typescript
import { ITaskRepository } from '../interfaces/repositories/ITaskRepository';
import { Task } from '../../domain/entities/Task';
import { TaskId } from '../../domain/value-objects/TaskId';
import { UserId } from '../../domain/value-objects/UserId';
import { Priority } from '../../domain/value-objects/Priority';
import { CreateTaskDTO } from '../dto/CreateTaskDTO';

export class CreateTaskUseCase {
  constructor(private readonly taskRepository: ITaskRepository) {}

  async execute(dto: CreateTaskDTO): Promise<Task> {
    // バリデーション
    if (!dto.title || dto.title.trim().length === 0) {
      throw new Error('Title is required');
    }

    // エンティティの作成
    const task = new Task(
      TaskId.generate(),
      new UserId(dto.userId),
      dto.title,
      dto.description || null,
      new Priority(dto.priority || Priority.MEDIUM),
      dto.dueDate || null,
      false,
      null,
      new Date(),
      new Date()
    );

    // リポジトリに保存
    return await this.taskRepository.save(task);
  }
}
```

**インターフェース（ITaskRepository.ts）**

```typescript
import { Task } from '../../domain/entities/Task';
import { TaskId } from '../../domain/value-objects/TaskId';
import { UserId } from '../../domain/value-objects/UserId';

export interface ITaskRepository {
  findById(id: TaskId): Promise<Task | null>;
  findByUserId(userId: UserId): Promise<Task[]>;
  save(task: Task): Promise<Task>;
  update(task: Task): Promise<Task>;
  delete(id: TaskId): Promise<void>;
}
```

### 3. Infrastructure層（インフラストラクチャ層）

#### 責務

- データベースアクセスの実装
- 外部サービスとの連携
- リポジトリの実装
- Prismaクライアントの管理

#### ディレクトリ構造

```
src/infrastructure/
├── database/          # データベース関連
│   ├── prisma/
│   │   └── client.ts  # Prismaクライアントの初期化
│   └── migrations/    # マイグレーション（Prisma管理）
├── repositories/      # リポジトリ実装
│   ├── TaskRepository.ts
│   ├── UserRepository.ts
│   └── AchievementRepository.ts
├── services/          # 外部サービス連携
│   ├── SupabaseService.ts
│   └── LocalStorageService.ts
└── mappers/           # エンティティとDBモデルの変換
    ├── TaskMapper.ts
    └── UserMapper.ts
```

#### 実装例

**リポジトリ実装（TaskRepository.ts）**

```typescript
import { ITaskRepository } from '../../application/interfaces/repositories/ITaskRepository';
import { Task } from '../../domain/entities/Task';
import { TaskId } from '../../domain/value-objects/TaskId';
import { UserId } from '../../domain/value-objects/UserId';
import { prisma } from '../database/prisma/client';
import { TaskMapper } from '../mappers/TaskMapper';

export class TaskRepository implements ITaskRepository {
  async findById(id: TaskId): Promise<Task | null> {
    const taskData = await prisma.task.findUnique({
      where: { id: id.getValue() },
      include: {
        category: true,
        tags: { include: { tag: true } },
        subtasks: true,
        notes: true
      }
    });

    if (!taskData) {
      return null;
    }

    return TaskMapper.toDomain(taskData);
  }

  async findByUserId(userId: UserId): Promise<Task[]> {
    const tasksData = await prisma.task.findMany({
      where: { userId: userId.getValue() },
      include: {
        category: true,
        tags: { include: { tag: true } },
        subtasks: true,
        notes: true
      },
      orderBy: { createdAt: 'desc' }
    });

    return tasksData.map(TaskMapper.toDomain);
  }

  async save(task: Task): Promise<Task> {
    const taskData = TaskMapper.toPersistence(task);
    const saved = await prisma.task.create({
      data: taskData,
      include: {
        category: true,
        tags: { include: { tag: true } },
        subtasks: true,
        notes: true
      }
    });

    return TaskMapper.toDomain(saved);
  }

  async update(task: Task): Promise<Task> {
    const taskData = TaskMapper.toPersistence(task);
    const updated = await prisma.task.update({
      where: { id: task.id.getValue() },
      data: taskData,
      include: {
        category: true,
        tags: { include: { tag: true } },
        subtasks: true,
        notes: true
      }
    });

    return TaskMapper.toDomain(updated);
  }

  async delete(id: TaskId): Promise<void> {
    await prisma.task.delete({
      where: { id: id.getValue() }
    });
  }
}
```

**マッパー（TaskMapper.ts）**

```typescript
import { Task } from '../../domain/entities/Task';
import { TaskId } from '../../domain/value-objects/TaskId';
import { UserId } from '../../domain/value-objects/UserId';
import { Priority } from '../../domain/value-objects/Priority';
import { Task as PrismaTask } from '@prisma/client';

export class TaskMapper {
  static toDomain(prismaTask: PrismaTask): Task {
    return new Task(
      new TaskId(prismaTask.id),
      new UserId(prismaTask.userId),
      prismaTask.title,
      prismaTask.description,
      new Priority(prismaTask.priority),
      prismaTask.dueDate,
      prismaTask.completed,
      prismaTask.completedAt,
      prismaTask.createdAt,
      prismaTask.updatedAt
    );
  }

  static toPersistence(task: Task): Omit<PrismaTask, 'createdAt' | 'updatedAt'> {
    return {
      id: task.id.getValue(),
      userId: task.userId.getValue(),
      title: task.title,
      description: task.description,
      priority: task.priority.getValue(),
      dueDate: task.dueDate,
      completed: task.completed,
      completedAt: task.completedAt,
      categoryId: null // 必要に応じて実装
    };
  }
}
```

### 4. Presentation層（プレゼンテーション層）

#### 責務

- UIコンポーネントの実装
- ページの実装
- 状態管理
- ユーザーインタラクションの処理

#### ディレクトリ構造

```
src/presentation/
├── components/        # UIコンポーネント
│   ├── tasks/
│   │   ├── TaskCard.tsx
│   │   ├── TaskForm.tsx
│   │   └── TaskList.tsx
│   ├── achievements/
│   │   └── AchievementBadge.tsx
│   └── common/
│       ├── Button.tsx
│       └── Modal.tsx
├── pages/            # Next.jsページ（App Router）
│   ├── page.tsx      # ホームページ
│   ├── tasks/
│   │   └── [id]/
│   │       └── page.tsx
│   └── settings/
│       └── page.tsx
├── hooks/           # カスタムフック
│   ├── useTasks.ts
│   ├── useCreateTask.ts
│   └── useCompleteTask.ts
└── providers/       # コンテキストプロバイダー
    └── TaskProvider.tsx
```

#### 実装例

**カスタムフック（useCreateTask.ts）**

```typescript
import { useState } from 'react';
import { CreateTaskUseCase } from '../../application/use-cases/tasks/CreateTaskUseCase';
import { TaskRepository } from '../../infrastructure/repositories/TaskRepository';
import { CreateTaskDTO } from '../../application/dto/CreateTaskDTO';

export function useCreateTask() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const createTask = async (dto: CreateTaskDTO) => {
    setLoading(true);
    setError(null);

    try {
      const repository = new TaskRepository();
      const useCase = new CreateTaskUseCase(repository);
      const task = await useCase.execute(dto);
      return task;
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to create task');
      throw err;
    } finally {
      setLoading(false);
    }
  };

  return { createTask, loading, error };
}
```

**コンポーネント（TaskForm.tsx）**

```typescript
'use client';

import { useState } from 'react';
import { useCreateTask } from '../../hooks/useCreateTask';
import { Priority } from '../../../domain/value-objects/Priority';

interface TaskFormProps {
  userId: string;
  onSuccess?: () => void;
}

export function TaskForm({ userId, onSuccess }: TaskFormProps) {
  const { createTask, loading, error } = useCreateTask();
  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');
  const [priority, setPriority] = useState<Priority>(Priority.MEDIUM);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    try {
      await createTask({
        userId,
        title,
        description,
        priority
      });
      setTitle('');
      setDescription('');
      setPriority(Priority.MEDIUM);
      onSuccess?.();
    } catch (err) {
      // エラーは既にuseCreateTaskで処理されている
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="タスクのタイトル"
        required
      />
      <textarea
        value={description}
        onChange={(e) => setDescription(e.target.value)}
        placeholder="説明（オプション）"
      />
      <select
        value={priority}
        onChange={(e) => setPriority(e.target.value as Priority)}
      >
        <option value={Priority.LOW}>低</option>
        <option value={Priority.MEDIUM}>中</option>
        <option value={Priority.HIGH}>高</option>
        <option value={Priority.URGENT}>緊急</option>
      </select>
      <button type="submit" disabled={loading}>
        {loading ? '作成中...' : '作成'}
      </button>
      {error && <p className="error">{error}</p>}
    </form>
  );
}
```

## 依存関係の方向

### 依存性逆転の原則

```
Presentation → Application → Domain ← Infrastructure
     ↓              ↓                      ↑
     └─────────────┴──────────────────────┘
            (インターフェース経由)
```

- Presentation層はApplication層に依存
- Application層はDomain層に依存
- Infrastructure層はApplication層のインターフェースを実装
- Domain層はどの層にも依存しない

### 依存性注入（Dependency Injection）

各層の依存関係は、依存性注入パターンを使用して管理します。

```typescript
// 依存性注入の例
const taskRepository = new TaskRepository();
const createTaskUseCase = new CreateTaskUseCase(taskRepository);
```

将来的には、DIコンテナ（例：InversifyJS、tsyringe）の導入を検討します。

## データフロー

### タスク作成のフロー

```
1. ユーザーがTaskFormでタスクを入力
   ↓
2. TaskFormがuseCreateTaskフックを呼び出し
   ↓
3. useCreateTaskがCreateTaskUseCaseを実行
   ↓
4. CreateTaskUseCaseがITaskRepositoryを使用
   ↓
5. TaskRepository（実装）がPrismaクライアントを使用
   ↓
6. PrismaがSupabase（PostgreSQL）にデータを保存
   ↓
7. 結果が逆の順序で返される
   ↓
8. UIが更新される
```

## ディレクトリ構造（全体）

```
src/
├── domain/              # Domain層
│   ├── entities/
│   ├── value-objects/
│   ├── models/
│   └── rules/
├── application/         # Application層
│   ├── use-cases/
│   ├── interfaces/
│   └── dto/
├── infrastructure/      # Infrastructure層
│   ├── database/
│   ├── repositories/
│   ├── services/
│   └── mappers/
└── presentation/        # Presentation層
    ├── components/
    ├── pages/
    ├── hooks/
    └── providers/
```

## テスト戦略

### 単体テスト

- **Domain層**: ビジネスルールのテスト
- **Application層**: ユースケースのテスト（モックを使用）
- **Infrastructure層**: リポジトリのテスト（テストデータベースを使用）

### 統合テスト

- 各層間の統合テスト
- APIエンドポイントのテスト

### E2Eテスト

- ユーザーシナリオに基づくテスト
- PlaywrightやCypressを使用

## パフォーマンス最適化

### キャッシュ戦略

- React QueryやSWRを使用したデータキャッシュ
- サーバーサイドでのキャッシュ（必要に応じて）

### コード分割

- Next.jsの動的インポートを使用
- レイヤーごとに適切に分割

## セキュリティ

### データアクセス制御

- SupabaseのRow Level Security（RLS）を使用
- ユーザーごとのデータ分離

### 入力検証

- Domain層でのバリデーション
- Presentation層での追加検証

## 今後の拡張

- DIコンテナの導入
- イベント駆動アーキテクチャの導入
- CQRSパターンの導入（必要に応じて）
- マイクロサービスへの移行（必要に応じて）



