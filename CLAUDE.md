# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

Praise Todoは、タスク完了時に褒めメッセージ、アニメーション、ポイント、バッジ、ストリークでゲーミフィケーションを行うモチベーション駆動型タスク管理アプリケーションです。Next.js 16、React 19、TypeScript、Tailwind CSS 4で構築されています。

## コマンド

```bash
pnpm dev          # 開発サーバー起動 (http://localhost:3000)
pnpm build        # 本番ビルド
pnpm start        # 本番サーバー起動
pnpm lint         # Biomeリンティング実行 (biome check)
pnpm format       # Biomeでコードフォーマット (biome format --write)
```

## アーキテクチャ

クリーンアーキテクチャに従った4層構造。依存関係は内側に向かう: Presentation → Application → Domain ← Infrastructure

```
src/
├── domain/          # エンティティ、値オブジェクト、ビジネスルール（外部依存なし）
├── application/     # ユースケース、インターフェース（ITaskRepository等）、DTO
├── infrastructure/  # リポジトリ実装、Prismaクライアント、Supabase連携
└── presentation/    # Reactコンポーネント、Next.jsページ、カスタムフック、Context
```

### 主要原則
- **依存性逆転**: InfrastructureはApplicationで定義されたインターフェースを実装
- **単一責任**: 各クラス・モジュールは1つの責任のみ
- **レイヤー分離**: Domainは外部依存ゼロ

## データベース

- Supabase経由のPostgreSQL + Prisma ORM
- スキーマ定義: `docs/database-schema.md`
- Row Level Security（RLS）でユーザーデータ分離

## ドキュメント

- `docs/specification.md` - プロジェクト要件・仕様
- `docs/features.md` - 詳細な機能仕様
- `docs/architecture.md` - クリーンアーキテクチャ設計詳細
- `docs/database-schema.md` - Prismaスキーマとデータベース設計
- `docs/development-workflow.md` - 開発プロセス

## 開発フロー

1. 実装前に`docs/specification.md`と`docs/features.md`を確認
2. 大きな機能単位でGitHub Issueを作成（例: 「タスク管理機能」「褒め機能」）
3. 重要な機能のテストケースを決定（頻繁に修正する部分は後回し）
4. クリーンアーキテクチャの原則に従って実装
5. 実装と同時にテスト作成
6. テスト実行・エラー修正
7. 画面の動作確認
8. 仕様と異なる場合は`docs/specification.md`を更新
9. PRをIssueにリンクしてクローズ

## 技術スタック

- Next.js 16 / React 19 / TypeScript 5
- Tailwind CSS 4 / Biome（リンティング・フォーマット）
- Supabase（PostgreSQL） / Prisma ORM
- パッケージマネージャー: pnpm
