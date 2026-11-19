# b4 設計ドキュメント（v0.1 ドラフト）

> 自然言語のチャットから Web アプリを生成・プレビュー・公開できる AI アプリビルダー。
> 参考: Base44 / Lovable。本書は初期設計のたたき台であり、末尾の「未決事項」を詰めながら更新する。

---

## 1. 参考プロダクトの整理

| 観点 | Lovable | Base44 | b4 の方針 |
|---|---|---|---|
| 生成物 | Vite + React + TS + Tailwind + shadcn/ui の SPA | React SPA + プラットフォーム内蔵バックエンド | Lovable 同等のフロント構成 |
| バックエンド | Supabase 連携（ユーザーの Supabase プロジェクト） | **組み込み BaaS**（Entities / Auth / Integrations）を SDK 経由で利用 | **Base44 型の組み込み BaaS**（外部アカウント不要で即動く） |
| コード所有 | GitHub 双方向同期、コードを直接編集可 | コードは見えるが主にプラットフォーム内 | コード閲覧・編集可、GitHub エクスポートは P2 |
| プレビュー | クラウド上の開発サーバ | プラットフォーム上でホスト | MVP はブラウザ内バンドル、将来サーバサンドボックス |
| 公開 | ワンクリック公開・カスタムドメイン | 同左 | 同左（静的ホスティング + BaaS） |

**選定理由**: 「プロンプト1つで DB・認証付きアプリが動く」体験の最短経路は Base44 型。
外部サービス連携（Supabase 等）を前提にするとオンボーディングが重くなり、エージェントの失敗要因も増える。

---

## 2. ゴール / 非ゴール

### ゴール（MVP）
1. チャットで指示 → AI がマルチファイルの React アプリを生成・修正する
2. 生成中のコードがストリーミング表示され、完了後すぐ右ペインでプレビューできる
3. 生成アプリは組み込み BaaS で **データ永続化（CRUD）とエンドユーザー認証** が使える
4. 変更はバージョンとして保存され、任意の時点に戻せる
5. ワンクリックで `*.b4.app` に公開できる

### 非ゴール（MVP では扱わない）
- 任意の npm パッケージ / 任意のサーバサイドコード（Node/Python）の実行
- モバイルネイティブアプリ生成
- リアルタイム共同編集
- Figma インポート、決済連携、カスタムドメイン（P2 以降）

---

## 3. 全体アーキテクチャ

```
┌──────────────────────────── Builder Web (Next.js) ────────────────────────────┐
│  Chat pane  │  Code / File tree  │  Preview (iframe, 別オリジン)              │
└──────┬──────────────────┬───────────────────────────▲────────────────────────┘
       │ SSE / WebSocket  │ REST                      │ postMessage (エラー/ログ)
┌──────▼──────────────────▼──────────┐       ┌────────┴───────────┐
│ API Server (Node/TS)               │       │ Preview Runtime     │
│  - Auth / Projects / Versions      │       │  esbuild-wasm で     │
│  - Agent Orchestrator ─────────────┼──┐    │  ブラウザ内バンドル   │
│  - Build Checker (tsc/esbuild)     │  │    └────────┬───────────┘
└──────┬─────────────────────────────┘  │             │ b4 SDK (fetch)
       │                                │    ┌────────▼───────────────────────┐
┌──────▼──────┐  ┌─────────────┐  ┌─────▼──┐ │ App Runtime API (BaaS)          │
│ PostgreSQL  │  │ Object Store│  │ Claude │ │  - Entities CRUD（JSONB）        │
│ (メタ+App DB)│  │ (S3/R2)     │  │  API   │ │  - End-user Auth                │
└─────────────┘  └─────────────┘  └────────┘ │  - Integrations (LLM, Email,    │
                                             │    File upload)                 │
                                             └─────────────────────────────────┘
```

### コンポーネント

| コンポーネント | 役割 |
|---|---|
| **Builder Web** | ビルダー UI。チャット、ファイルツリー、Monaco エディタ、プレビュー iframe、バージョン履歴 |
| **API Server** | プロジェクト/バージョン管理、エージェント実行、ジョブキュー、SSE 配信 |
| **Agent Orchestrator** | Claude とのツール呼び出しループ。ファイル操作・スキーマ定義・ビルド検証をツールとして提供 |
| **Build Checker** | 生成コードを型チェック/バンドルしてエラーを返す（エージェントの自己修正用）。ワーカープロセスで隔離実行 |
| **Preview Runtime** | ファイル群を受け取りブラウザ内でバンドル → iframe で実行。ランタイムエラーを親へ通知 |
| **App Runtime API (BaaS)** | 生成アプリが SDK 経由で呼ぶ API。アプリ単位でテナント分離 |
| **Publisher** | 公開時に本番ビルドを作り静的ホスティング（CDN）へ配置 |

---

## 4. 生成アプリの仕様（テンプレート）

エージェントが「必ずこの土台の上に書く」固定テンプレート。出力の安定性に直結するため最重要。

```
/index.html
/src/main.tsx
/src/App.tsx              ← ルーティング（react-router）
/src/pages/*.tsx
/src/components/*.tsx
/src/components/ui/*      ← shadcn/ui（プリセット、編集非推奨）
/src/lib/b4.ts            ← b4 SDK クライアント（自動生成・編集禁止）
/b4/entities/*.json       ← エンティティ定義（JSON Schema）
```

- **許可パッケージのホワイトリスト**: react, react-dom, react-router, tailwind, shadcn/ui 一式, lucide-react, date-fns, recharts, zod, react-hook-form など。
  プレビュー側で事前バンドル済み CDN（esm）として提供 → 任意 npm を許さない代わりに起動が速い。
- **b4 SDK の利用イメージ**:

```ts
import { b4 } from "@/lib/b4";

const tasks = await b4.entities.Task.list({ sort: "-created_at", limit: 50 });
await b4.entities.Task.create({ title: "買い物", done: false });
const me = await b4.auth.me();
const reply = await b4.integrations.llm.invoke({ prompt: "要約して: ..." });
```

---

## 5. 組み込み BaaS（App Runtime）

### エンティティ
- エンティティは JSON Schema で定義（`/b4/entities/Task.json`）。エージェントは `define_entity` ツールで作成/変更。
- 保存は PostgreSQL の汎用テーブル + JSONB。MVP ではアプリごとに物理テーブルを作らない。

```sql
app_records(
  id uuid pk, app_id uuid, entity text, data jsonb,
  created_by uuid null, created_at, updated_at
)
-- index: (app_id, entity, created_at), GIN(data)
```

- サーバ側で JSON Schema バリデーション。スキーマ変更は追加的変更のみ自動適用、破壊的変更はユーザー確認。

### アクセス制御
- エンティティ単位の簡易ルール: `public_read`, `owner_only`, `authenticated` など宣言的に指定。
- 生成アプリから届くリクエストは **アプリごとの公開キー + エンドユーザー JWT**。管理系操作はビルダー側からのみ。

### エンドユーザー認証
- メール + マジックリンク / Google OAuth を提供。`app_users` はアプリ単位で分離。

### Integrations
- `llm.invoke`（プラットフォームのクレジットから消費）、`email.send`、`files.upload`（S3 署名付き URL）。
- レート制限・クォータをアプリ/ワークスペース単位で適用。

---

## 6. AI エージェント設計

### 6.1 方式
- **Claude Messages API + ツール使用ループを自前ホスト**する。
  理由: ファイルは自社 DB/ストアが正であり、ビルド検証・プレビュー連携・バージョン保存をループ内で密に制御したいため。
  （代替: Anthropic の Managed Agents。ループとサンドボックスをホストしてくれるが、プレビュー連携とファイル正本の扱いを要検証 → 未決事項へ）
- モデル: `claude-opus-5-5`（adaptive thinking、effort はコーディング用途のため `high` から評価開始）。
  拒否時のフォールバックとしてサーバサイド `fallbacks` を有効にする。
- ストリーミング必須。`write_file` 等のツールには `eager_input_streaming: true` を付け、
  生成中のコードを UI にリアルタイム表示する（受信側でスキーマ検証してから適用）。

### 6.2 ツール一覧

| ツール | 説明 |
|---|---|
| `list_files` | ファイルツリーとサイズ |
| `read_file(path)` | ファイル内容 |
| `write_file(path, content)` | 新規作成 / 全置換 |
| `edit_file(path, old_str, new_str)` | 部分置換（大きいファイルの修正はこちらを推奨） |
| `delete_file(path)` / `rename_file` | |
| `define_entity(name, schema, access)` | エンティティ定義の作成・更新 |
| `check_build()` | 型チェック + バンドル。エラー一覧を返す |
| `get_preview_errors()` | プレビュー iframe から報告されたランタイムエラー/コンソール |
| `search_docs(query)` | b4 SDK・テンプレート・shadcn の社内ドキュメント検索 |

### 6.3 1 ターンの流れ

```
ユーザー指示
  → [コンテキスト構築] system(固定) + tools(固定) + プロジェクト概要(ファイルツリー, エンティティ一覧)
  → Claude: 計画(短文) → read/edit/write ツール群（並列呼び出し可）
  → check_build() → エラーあれば自己修正（最大 N=3 回）
  → 変更をバージョンとしてコミット → プレビュー更新
  → プレビューのランタイムエラーを検知したら「直しますか？」ボタンで次ターンへ
```

- ツール実行は **作業コピー（ドラフト）** に対して行い、ターン完了時に 1 バージョンとして確定。中断時は破棄可能。
- ファイル内容はプロンプトに全量は入れない。ファイルツリー + 直近編集ファイル + 必要に応じ `read_file`。

### 6.4 プロンプトキャッシュ
- 並び順 `tools → system → messages` の先頭を固定化（system にタイムスタンプ等を入れない、ツール定義順を固定）。
- 会話途中の運用指示は top-level `system` を書き換えず、messages 内の system メッセージで追加。
- `usage.cache_read_input_tokens` をメトリクス化してヒット率を監視。
- 長い会話はサーバサイドの compaction を検討（P1）。

### 6.5 品質の担保
- 評価セット（典型プロンプト 30〜50 件: TODO、CRM、予約、ダッシュボード…）を用意し、
  「ビルド成功率」「ランタイムエラー率」「要件充足（LLM 判定）」を回帰テストとして計測。
- テンプレート・system プロンプト変更時は必ず評価を回す。

---

## 7. データモデル（ビルダー側）

```
users(id, email, name, created_at)
workspaces(id, name, plan, credit_balance)
workspace_members(workspace_id, user_id, role)
projects(id, workspace_id, name, slug, current_version_id, visibility, created_at)
project_versions(id, project_id, parent_id, message_id, manifest jsonb, created_at)
   -- manifest: { "src/App.tsx": "<blob sha>", ... }
blobs(sha pk, size, storage_key)               -- コンテンツアドレス型、重複排除
chat_messages(id, project_id, role, content jsonb, created_at)
agent_runs(id, project_id, message_id, status, model, input_tokens, output_tokens,
           cache_read_tokens, cost, started_at, finished_at, error)
entities(id, project_id, name, schema jsonb, access jsonb, version)
deployments(id, project_id, version_id, url, status, created_at)
app_users(id, app_id, email, provider, created_at)
app_records(...)                               -- 5章参照
```

- バージョンは Git 風のスナップショット（manifest + blob）。差分表示・ロールバックが容易で、GitHub エクスポートにも変換しやすい。

---

## 8. プレビュー方式

| 方式 | 長所 | 短所 | 採否 |
|---|---|---|---|
| A. ブラウザ内バンドル（esbuild-wasm + import map） | サーバコスト≒0、起動が速い | 任意 npm・Node API 不可 | **MVP 採用** |
| B. WebContainers | ブラウザ内で Node が動く | 商用ライセンス要、重い | 不採用 |
| C. サーバサンドボックス（Firecracker / E2B / Fly Machines 等）で Vite dev server | 任意パッケージ・フルスタック可、本物の HMR | コスト・起動待ち・運用負荷 | P2 で導入 |

- プレビュー iframe は **別オリジン**（`*.preview.b4.dev`）+ `sandbox` 属性で隔離。ビルダーのセッションに触れさせない。
- iframe 内でエラー境界・`window.onerror`・console をフックし `postMessage` で親へ送る。

---

## 9. 公開（デプロイ）

1. 指定バージョンを本番ビルド（esbuild、minify）
2. 成果物を Object Store へ → CDN 配下の `https://<slug>.b4.app` で配信
3. App Runtime API は共通エンドポイント（`api.b4.app/apps/<app_id>/...`）
4. P2: カスタムドメイン（TLS 自動発行）、環境変数/シークレット

---

## 10. セキュリティ / マルチテナンシー

- 生成コードは **常に信頼しない**: プレビュー・公開ともに別オリジン、CSP 設定。
- Build Checker はネットワーク遮断・CPU/メモリ/時間制限付きのワーカーで実行。
- BaaS はすべてのクエリで `app_id` を必須にし、行レベルで分離（Postgres RLS も併用）。
- プロンプトインジェクション対策: ユーザーアプリ内のデータや外部取得テキストはエージェントへ「データ」として渡し、ツール権限を最小化（エージェントはプロジェクト外へアクセス不可）。
- 課金悪用対策: ワークスペース単位のクレジット、エージェント1ターンあたりのトークン上限・ツール呼び出し回数上限。

---

## 11. 技術スタック（提案）

| 層 | 選定 |
|---|---|
| モノレポ | pnpm workspaces + Turborepo |
| Builder Web | Next.js (App Router) + Tailwind + shadcn/ui + Monaco |
| API Server | Node.js + TypeScript（Hono or Fastify）、SSE |
| エージェント | `@anthropic-ai/sdk`（Messages API, streaming, tool use） |
| ジョブキュー | BullMQ（Redis） |
| DB | PostgreSQL（Drizzle ORM） |
| ストレージ | S3 互換（Cloudflare R2） |
| プレビュー | esbuild-wasm + esm CDN（自前ミラー） |
| 認証（ビルダー） | Auth.js or Clerk |
| 監視 | OpenTelemetry + 任意の APM |

```
apps/
  web/          Builder UI
  api/          API Server + Agent Orchestrator
  runtime/      App Runtime API (BaaS)
packages/
  sdk/          生成アプリ用 b4 SDK
  template/     生成アプリのテンプレート
  preview/      ブラウザ内バンドラ
  db/           スキーマ・マイグレーション
  agent/        プロンプト・ツール定義・評価セット
```

---

## 12. ロードマップ

| フェーズ | 内容 | 完了条件 |
|---|---|---|
| **P0: 技術検証（1〜2週）** | エージェントループ + ブラウザ内プレビューの最小構成（DB なし、ローカル保存） | 「TODO アプリ作って」→ 動くプレビューが出る |
| **P1: MVP（4〜6週）** | ユーザー/プロジェクト/バージョン、BaaS（Entities + Auth）、公開、評価セット | 評価セットでビルド成功率 90%+ |
| **P2: 拡張** | GitHub エクスポート、カスタムドメイン、サーバサンドボックス（フルスタック）、画像/スクショ入力、Visual Edit（要素クリックで修正） | — |
| **P3: 事業化** | 課金（Stripe）、チーム機能、テンプレートギャラリー | — |

---

## 13. 未決事項（要判断）

1. **ターゲット**: 個人開発者 / 非エンジニア / 社内ツール用途（B2B） のどれを主眼にするか → テンプレートと BaaS 機能の優先度が変わる
2. **バックエンド方式の最終確定**: 組み込み BaaS（本案）か、Supabase 連携か、両対応か
3. **エージェントのホスティング**: 自前ループ（本案）か Managed Agents か（プレビュー連携の可否を P0 で検証）
4. **プレビュー**: MVP をブラウザ内バンドルで割り切ってよいか（任意 npm 不可の制約を許容できるか）
5. **インフラ**: 想定クラウド（AWS / GCP / Cloudflare / Vercel+Fly 等）
6. **コスト方針**: LLM 利用料をクレジット制でユーザーに転嫁するか、モデル使い分け（軽いタスクを安価なモデルへ）を行うか
7. **公開範囲**: OSS にするか、クローズドな SaaS か
