# 対話システム要件定義書（汎用版）

- 文書ID: `DS-REQ-001`
- バージョン: `2.1.0`
- 作成日: `2026-04-04`
- 最終更新日: `2026-04-04`
- 対象: 複数アプリ（Web / モバイル / CLI / API）で再利用可能な対話システム基盤

## 1. 目的
- 自然な対話を実現する共通基盤を提供する。
- アプリ固有仕様と対話エンジンを分離し、再利用性を高める。
- RAG、ツール実行、権限制御、監査を一貫した設計で実装可能にする。

## 2. スコープ
- 1対1のマルチターン対話。
- ストリーミング応答。
- ツール呼び出しを含むエージェント対話。
- セッション永続化、再開、履歴圧縮。
- RAG検索結果の注入と出典追跡。
- 運用監視（ログ、メトリクス、トレース）。

## 3. 非スコープ
- 音声認識/音声合成エンジン自体の開発。
- ベクトルDBの製品依存チューニング詳細。
- 個別プロダクトUIの画面設計。

## 4. 設計原則
- 分岐中心ではなくイベント駆動で設計する。
- 対話制御、ツール実行、表示、ポリシーを分離する。
- モデル品質はプロンプトと文脈設計で担保し、業務分岐で過剰吸収しない。
- 失敗を前提に、停止理由と再開可能性を常に保持する。

## 5. 論理アーキテクチャ
- `Channel Adapter`: Web/CLI/モバイル向けの入出力変換。
- `Conversation Orchestrator`: ターン実行、状態遷移管理。
- `Prompt Composer`: システム指示、履歴、RAG結果の合成。
- `Model Gateway`: LLM API統合、ストリーム処理。
- `Tool Runtime`: ツール実行、タイムアウト、再試行。
- `Policy Engine`: 権限判定、承認フロー、拒否理由生成。
- `Memory & Retrieval`: 会話履歴、長期記憶、検索結果管理。
- `Session Store`: セッション永続化、復元。
- `Observability`: ログ、メトリクス、トレース集約。

## 6. 対話状態機械
- `idle`
- `receiving_input`
- `calling_model`
- `streaming_assistant`
- `awaiting_tool_execution`
- `awaiting_user_approval`
- `applying_tool_result`
- `completed`
- `failed`
- `cancelled`

## 7. 機能要件（Functional Requirements）

| ID | 要件 | 優先度 | 受入基準 |
|---|---|---|---|
| FR-001 | セッション作成が可能であること | MUST | `session_id` が返却される |
| FR-002 | セッション再開が可能であること | MUST | 過去メッセージが復元される |
| FR-003 | ユーザー発話を時系列で保持すること | MUST | 順序が不変で保存される |
| FR-004 | アシスタント応答をストリーミング配信できること | MUST | delta単位で受信できる |
| FR-005 | 1ターン内で複数ツール呼び出しを処理できること | MUST | 複数 `tool_use` を処理可能 |
| FR-006 | ツール結果を会話へ再注入できること | MUST | `tool_result` が次推論に反映される |
| FR-007 | ツール不要ターンは通常応答で終了できること | MUST | 無限ループしない |
| FR-008 | ターン反復回数上限を設定可能であること | MUST | 上限超過で停止理由を返す |
| FR-009 | トークン予算上限を設定可能であること | MUST | 予算超過で安全停止する |
| FR-010 | システムプロンプトを差し替え可能であること | MUST | 設定変更で反映される |
| FR-011 | 履歴圧縮（コンパクション）を実行できること | SHOULD | 履歴サイズを削減できる |
| FR-012 | 権限モードを保持すること（read/write/full） | MUST | ツール要件と照合される |
| FR-013 | 承認必須ツールを実行前確認できること | MUST | 承認UI/APIが呼ばれる |
| FR-014 | 拒否理由をモデルへ返却できること | MUST | 拒否理由付き `tool_result` を注入 |
| FR-015 | pre/post hook を実行できること | SHOULD | hook結果を実行可否へ反映 |
| FR-016 | ツール実行タイムアウトを設定可能であること | MUST | タイムアウト時に失敗として扱う |
| FR-017 | ツール実行内容を監査ログへ記録できること | SHOULD | `tool/input/output/error` を保存 |
| FR-018 | ツール失敗時も会話を継続可能であること | MUST | セッション破損が発生しない |
| FR-019 | RAG結果を構造化注入できること | MUST | `retrieval_context` を注入可能 |
| FR-020 | 出典情報を保持できること | MUST | source id/url/chunk id を保持 |
| FR-021 | 検索ヒットなし時のフォールバックがあること | MUST | 規定応答を返却 |
| FR-022 | 複数チャネルで同一セッション継続可能であること | SHOULD | Web→CLI継続などに対応 |
| FR-023 | セッションエクスポート可能であること | SHOULD | JSON/Markdownで出力できる |
| FR-024 | モデル切替をセッション継続で行えること | SHOULD | 中断せず切替できる |
| FR-025 | 実行中ターンをキャンセルできること | MUST | キャンセル状態へ遷移 |
| FR-026 | 冪等再送制御を持つこと | SHOULD | 重複要求を二重実行しない |
| FR-027 | 機械可読なエラー分類コードを返すこと | MUST | 例: `policy_denied` |
| FR-028 | プラグインツールを動的ロードできること | SHOULD | 再起動不要で反映可能 |
| FR-029 | 管理者向け実行トレースを参照できること | SHOULD | 1ターンの全イベント追跡可 |
| FR-030 | PIIマスキングポリシーを適用できること | MUST | ログに機微情報を残さない |

## 8. 非機能要件（Non-Functional Requirements）

| ID | 要件 | 目標 |
|---|---|---|
| NFR-001 | 初回トークン遅延（ツールなし） | p95 < 2.5秒 |
| NFR-002 | 初回トークン遅延（ツールあり） | p95 < 8秒 |
| NFR-003 | 可用性 | 月間 99.9%以上 |
| NFR-004 | 永続化耐久性 | 書込成功率 99.99%以上 |
| NFR-005 | 同時セッション数 | 初期 1,000以上 |
| NFR-006 | セキュリティ | 通信TLS、保存時暗号化 |
| NFR-007 | 監査性 | 全ターン追跡可能 |
| NFR-008 | 拡張性 | 新ツール追加でコア改修不要 |
| NFR-009 | 可観測性 | 全ターンに `trace_id` 付与 |
| NFR-010 | 復旧性 | セッション復元 RTO 15分以内 |

## 9. 会話ループ要件（中核）
- 対話制御は以下の反復を基本とする。

```text
1) ユーザー入力を履歴へ追加
2) モデルへ履歴 + 文脈 + ツール定義を送信
3) ストリームイベントを受信し assistant message を構築
4) tool_use がなければターン終了
5) tool_use ごとに:
   - 権限判定
   - pre-hook
   - ツール実行
   - post-hook
   - tool_result を履歴へ追加
6) 2)へ戻る（上限まで）
7) セッション永続化
```

## 10. データ要件（最小スキーマ）

```json
{
  "session": {
    "id": "string",
    "created_at": "epoch_ms",
    "state": "active|closed"
  },
  "message": {
    "id": "string",
    "session_id": "string",
    "role": "user|assistant|tool",
    "blocks": []
  },
  "tool_call": {
    "id": "string",
    "name": "string",
    "input_json": {},
    "status": "pending|done|error"
  },
  "tool_result": {
    "tool_call_id": "string",
    "output": "string",
    "is_error": false
  },
  "policy_decision": {
    "tool_call_id": "string",
    "decision": "allow|deny",
    "reason": "string"
  },
  "retrieval_trace": {
    "query": "string",
    "sources": [
      { "id": "string", "score": 0.0 }
    ]
  }
}
```

## 11. API要件（例）
- `POST /sessions`
- `GET /sessions/{id}`
- `POST /sessions/{id}/messages`
- `GET /sessions/{id}/events`（SSE/WebSocket）
- `POST /sessions/{id}/cancel`
- `POST /sessions/{id}/resume`
- `GET /sessions/{id}/trace/{turn_id}`

## 12. エラー処理要件
- モデルAPI失敗時は再試行ポリシーに従う。
- ツール失敗は `tool_result(is_error=true)` で返し、会話継続可能とする。
- 権限拒否時は必ず理由を返却する。
- 反復回数上限/予算上限到達時は停止理由を返却する。
- セッション保存失敗時はユーザーへ再開不能リスクを通知する。

## 13. セキュリティ要件
- ツールごとに required permission を定義必須。
- 危険操作は明示承認必須。
- APIキー/トークンはKMS/Vault等で管理。
- ログにはPII/秘密情報をマスクして保存。
- Web取得はドメイン制御（allow/deny）を可能にする。

## 14. 可観測性要件
- 全リクエストに `request_id`、全ターンに `trace_id` を付与する。
- 記録対象:
  - 応答遅延（TTFT/総応答時間）
  - ツール実行時間
  - エラー率
  - 反復回数
  - トークン使用量
- 監査ログは検索可能な形式で保持する。

## 15. テスト要件
- 単体テスト:
  - 状態遷移
  - 権限判定
  - イベント組み立て
- 結合テスト:
  - モデルストリーム→ツール実行→再注入
- E2Eテスト:
  - 通常対話
  - 権限拒否
  - ツール失敗
  - タイムアウト
- 負荷試験:
  - 同時セッション
  - 長時間セッション

## 16. 受入基準
- MUST要件を100%満たす。
- P1/P2の未解決障害がゼロである。
- 監視ダッシュボードとアラートが本番運用可能である。
- 障害時Runbookと復旧手順が整備済みである。

## 17. 実装フェーズ（推奨）
1. Phase 1: 基本対話ループ（単一モデル、永続化なし）
2. Phase 2: 永続化、権限、ツール実行、ストリーミング
3. Phase 3: RAG統合、履歴圧縮、監査
4. Phase 4: プラグイン拡張、運用最適化、SLO達成

## 18. 実装例（コード）

### 18.1 最小ディレクトリ構成例

```text
src/
  domain/
    types.ts
    errors.ts
  orchestrator/
    runTurn.ts
    streamAssembler.ts
  policy/
    permissionPolicy.ts
  tools/
    toolRegistry.ts
    toolExecutor.ts
    hooks.ts
  prompt/
    promptComposer.ts
  retrieval/
    retriever.ts
  storage/
    sessionRepository.ts
  api/
    routes.ts
    sse.ts
tests/
  runTurn.test.ts
  permissionPolicy.test.ts
```

### 18.2 ドメイン型定義（TypeScript）

```ts
// src/domain/types.ts
export type MessageRole = "system" | "user" | "assistant" | "tool";

export type ContentBlock =
  | { type: "text"; text: string }
  | { type: "tool_use"; id: string; name: string; input: string }
  | { type: "tool_result"; tool_use_id: string; tool_name: string; output: string; is_error: boolean };

export interface ConversationMessage {
  id: string;
  role: MessageRole;
  blocks: ContentBlock[];
  created_at: number;
}

export interface Session {
  id: string;
  messages: ConversationMessage[];
  created_at: number;
  updated_at: number;
}

export interface TokenUsage {
  input_tokens: number;
  output_tokens: number;
}

export type AssistantEvent =
  | { type: "text_delta"; text: string }
  | { type: "tool_use"; id: string; name: string; input: string }
  | { type: "usage"; usage: TokenUsage }
  | { type: "message_stop" };

export interface TurnSummary {
  iterations: number;
  assistant_messages: ConversationMessage[];
  tool_results: ConversationMessage[];
  usage: TokenUsage;
  stop_reason: "completed" | "max_iterations" | "max_budget";
}
```

### 18.3 エラー型（機械可読コード）

```ts
// src/domain/errors.ts
export type ErrorCode =
  | "policy_denied"
  | "tool_timeout"
  | "tool_failed"
  | "model_failed"
  | "max_iterations"
  | "max_budget"
  | "session_persist_failed";

export class AppError extends Error {
  constructor(
    public code: ErrorCode,
    message: string,
    public detail?: Record<string, unknown>,
  ) {
    super(message);
  }
}
```

### 18.4 ストリーム組み立て（イベント→assistant message）

```ts
// src/orchestrator/streamAssembler.ts
import { AssistantEvent, ContentBlock, ConversationMessage } from "../domain/types";

export function buildAssistantMessage(
  events: AssistantEvent[],
  messageId: string,
): { message: ConversationMessage; usage?: { input_tokens: number; output_tokens: number } } {
  let text = "";
  let stopped = false;
  let usage: { input_tokens: number; output_tokens: number } | undefined;
  const blocks: ContentBlock[] = [];

  const flushText = () => {
    if (!text) return;
    blocks.push({ type: "text", text });
    text = "";
  };

  for (const ev of events) {
    if (ev.type === "text_delta") text += ev.text;
    if (ev.type === "tool_use") {
      flushText();
      blocks.push({ type: "tool_use", id: ev.id, name: ev.name, input: ev.input });
    }
    if (ev.type === "usage") usage = ev.usage;
    if (ev.type === "message_stop") stopped = true;
  }
  flushText();

  if (!stopped) throw new Error("assistant stream ended without message_stop");
  if (blocks.length === 0) throw new Error("assistant stream produced no content");

  return {
    message: {
      id: messageId,
      role: "assistant",
      blocks,
      created_at: Date.now(),
    },
    usage,
  };
}
```

### 18.5 権限ポリシー実装例

```ts
// src/policy/permissionPolicy.ts
export type PermissionMode = "read-only" | "workspace-write" | "danger-full-access" | "prompt";

const rank: Record<PermissionMode, number> = {
  "read-only": 1,
  "workspace-write": 2,
  "danger-full-access": 3,
  "prompt": 0,
};

export interface PermissionRequest {
  tool_name: string;
  input: string;
  current_mode: PermissionMode;
  required_mode: PermissionMode;
}

export type PermissionDecision = { allow: true } | { allow: false; reason: string };

export class PermissionPolicy {
  constructor(
    private currentMode: PermissionMode,
    private requirements: Record<string, PermissionMode>,
  ) {}

  requiredModeFor(toolName: string): PermissionMode {
    return this.requirements[toolName] ?? "danger-full-access";
  }

  async authorize(
    toolName: string,
    input: string,
    promptApproval?: (req: PermissionRequest) => Promise<PermissionDecision>,
  ): Promise<PermissionDecision> {
    const required = this.requiredModeFor(toolName);
    if (this.currentMode !== "prompt" && rank[this.currentMode] >= rank[required]) {
      return { allow: true };
    }

    if (!promptApproval) {
      return {
        allow: false,
        reason: `tool '${toolName}' requires ${required}; current mode is ${this.currentMode}`,
      };
    }

    return promptApproval({
      tool_name: toolName,
      input,
      current_mode: this.currentMode,
      required_mode: required,
    });
  }
}
```

### 18.6 ツール定義と実行レジストリ

```ts
// src/tools/toolRegistry.ts
export interface ToolSpec {
  name: string;
  description: string;
  required_permission: "read-only" | "workspace-write" | "danger-full-access";
}

export interface ToolRuntime {
  execute(inputJson: unknown): Promise<string>;
}

export class ToolRegistry {
  private specs = new Map<string, ToolSpec>();
  private runtimes = new Map<string, ToolRuntime>();

  register(spec: ToolSpec, runtime: ToolRuntime): void {
    this.specs.set(spec.name, spec);
    this.runtimes.set(spec.name, runtime);
  }

  getSpec(name: string): ToolSpec | undefined {
    return this.specs.get(name);
  }

  async execute(name: string, inputJson: unknown): Promise<string> {
    const runtime = this.runtimes.get(name);
    if (!runtime) throw new Error(`unsupported tool: ${name}`);
    return runtime.execute(inputJson);
  }
}
```

### 18.7 pre/post Hook 実装例

```ts
// src/tools/hooks.ts
import { spawn } from "node:child_process";

export interface HookRunResult {
  denied: boolean;
  messages: string[];
}

async function runOneHook(command: string, payload: object): Promise<{ code: number; out: string; err: string }> {
  return new Promise((resolve, reject) => {
    const child = spawn("sh", ["-lc", command], { stdio: ["pipe", "pipe", "pipe"] });
    let out = "";
    let err = "";
    child.stdout.on("data", (d) => (out += d.toString()));
    child.stderr.on("data", (d) => (err += d.toString()));
    child.on("error", reject);
    child.on("close", (code) => resolve({ code: code ?? 1, out: out.trim(), err: err.trim() }));
    child.stdin.write(JSON.stringify(payload));
    child.stdin.end();
  });
}

export async function runHooks(
  commands: string[],
  payload: object,
): Promise<HookRunResult> {
  const messages: string[] = [];
  for (const cmd of commands) {
    const r = await runOneHook(cmd, payload);
    if (r.out) messages.push(r.out);
    if (r.code === 2) return { denied: true, messages: messages.length ? messages : ["hook denied"] };
  }
  return { denied: false, messages };
}
```

### 18.8 RAG検索インターフェースと注入

```ts
// src/retrieval/retriever.ts
export interface RetrievedChunk {
  source_id: string;
  title?: string;
  url?: string;
  chunk_id: string;
  content: string;
  score: number;
}

export interface Retriever {
  search(query: string, topK: number): Promise<RetrievedChunk[]>;
}
```

```ts
// src/prompt/promptComposer.ts
import { ConversationMessage } from "../domain/types";
import { RetrievedChunk } from "../retrieval/retriever";

export interface PromptInput {
  systemInstructions: string[];
  messages: ConversationMessage[];
  retrieved: RetrievedChunk[];
}

export function composePrompt(input: PromptInput): string[] {
  const sections: string[] = [];
  sections.push(...input.systemInstructions);
  if (input.retrieved.length > 0) {
    const sources = input.retrieved
      .map((c, i) => `[S${i + 1}] ${c.title ?? c.source_id} score=${c.score.toFixed(3)}\n${c.content}`)
      .join("\n\n");
    sections.push(
      "# Retrieval Context",
      "以下は検索結果です。回答では必要に応じて出典参照を明示してください。",
      sources,
    );
  }
  return sections;
}
```

### 18.9 会話ループ実装（中核）

```ts
// src/orchestrator/runTurn.ts
import { buildAssistantMessage } from "./streamAssembler";
import { PermissionPolicy } from "../policy/permissionPolicy";
import { ToolRegistry } from "../tools/toolRegistry";
import { runHooks } from "../tools/hooks";
import { AssistantEvent, ContentBlock, Session, TurnSummary } from "../domain/types";

interface ModelGateway {
  stream(request: { system_prompt: string[]; messages: Session["messages"] }): Promise<AssistantEvent[]>;
}

interface Options {
  maxIterations: number;
  maxBudgetTokens: number;
}

export async function runTurn(params: {
  session: Session;
  userInput: string;
  systemPrompt: string[];
  model: ModelGateway;
  tools: ToolRegistry;
  policy: PermissionPolicy;
  preHooks: string[];
  postHooks: string[];
  options: Options;
}): Promise<TurnSummary> {
  const { session, model, tools, policy, preHooks, postHooks, options } = params;
  session.messages.push({
    id: crypto.randomUUID(),
    role: "user",
    blocks: [{ type: "text", text: params.userInput }],
    created_at: Date.now(),
  });

  let iterations = 0;
  let inputTokens = 0;
  let outputTokens = 0;
  const assistantMessages = [];
  const toolResults = [];

  while (true) {
    iterations += 1;
    if (iterations > options.maxIterations) {
      return {
        iterations,
        assistant_messages: assistantMessages,
        tool_results: toolResults,
        usage: { input_tokens: inputTokens, output_tokens: outputTokens },
        stop_reason: "max_iterations",
      };
    }

    const events = await model.stream({ system_prompt: params.systemPrompt, messages: session.messages });
    const assembled = buildAssistantMessage(events, crypto.randomUUID());
    if (assembled.usage) {
      inputTokens += assembled.usage.input_tokens;
      outputTokens += assembled.usage.output_tokens;
    }
    session.messages.push(assembled.message);
    assistantMessages.push(assembled.message);

    if (inputTokens + outputTokens > options.maxBudgetTokens) {
      return {
        iterations,
        assistant_messages: assistantMessages,
        tool_results: toolResults,
        usage: { input_tokens: inputTokens, output_tokens: outputTokens },
        stop_reason: "max_budget",
      };
    }

    const toolUses = assembled.message.blocks.filter((b): b is Extract<ContentBlock, { type: "tool_use" }> => b.type === "tool_use");
    if (toolUses.length === 0) {
      return {
        iterations,
        assistant_messages: assistantMessages,
        tool_results: toolResults,
        usage: { input_tokens: inputTokens, output_tokens: outputTokens },
        stop_reason: "completed",
      };
    }

    for (const toolUse of toolUses) {
      const auth = await policy.authorize(toolUse.name, toolUse.input);
      if (!auth.allow) {
        const denied = {
          id: crypto.randomUUID(),
          role: "tool" as const,
          blocks: [{
            type: "tool_result" as const,
            tool_use_id: toolUse.id,
            tool_name: toolUse.name,
            output: auth.reason,
            is_error: true,
          }],
          created_at: Date.now(),
        };
        session.messages.push(denied);
        toolResults.push(denied);
        continue;
      }

      const pre = await runHooks(preHooks, {
        hook_event_name: "PreToolUse",
        tool_name: toolUse.name,
        tool_input_json: toolUse.input,
      });
      if (pre.denied) {
        const denied = {
          id: crypto.randomUUID(),
          role: "tool" as const,
          blocks: [{
            type: "tool_result" as const,
            tool_use_id: toolUse.id,
            tool_name: toolUse.name,
            output: pre.messages.join("\n") || "PreToolUse hook denied",
            is_error: true,
          }],
          created_at: Date.now(),
        };
        session.messages.push(denied);
        toolResults.push(denied);
        continue;
      }

      let output = "";
      let isError = false;
      try {
        const parsed = JSON.parse(toolUse.input);
        output = await tools.execute(toolUse.name, parsed);
      } catch (e) {
        isError = true;
        output = String(e);
      }

      const post = await runHooks(postHooks, {
        hook_event_name: "PostToolUse",
        tool_name: toolUse.name,
        tool_input_json: toolUse.input,
        tool_output: output,
        tool_result_is_error: isError,
      });
      if (post.messages.length > 0) output = `${output}\n\nHook feedback:\n${post.messages.join("\n")}`;
      if (post.denied) isError = true;

      const resultMessage = {
        id: crypto.randomUUID(),
        role: "tool" as const,
        blocks: [{
          type: "tool_result" as const,
          tool_use_id: toolUse.id,
          tool_name: toolUse.name,
          output,
          is_error: isError,
        }],
        created_at: Date.now(),
      };
      session.messages.push(resultMessage);
      toolResults.push(resultMessage);
    }
  }
}
```

### 18.10 セッション永続化（PostgreSQL例）

```sql
-- db/schema.sql
create table sessions (
  id text primary key,
  created_at bigint not null,
  updated_at bigint not null
);

create table messages (
  id text primary key,
  session_id text not null references sessions(id) on delete cascade,
  role text not null,
  blocks jsonb not null,
  created_at bigint not null
);

create index idx_messages_session_created_at on messages(session_id, created_at);
```

```ts
// src/storage/sessionRepository.ts
import { Pool } from "pg";
import { Session } from "../domain/types";

export class SessionRepository {
  constructor(private db: Pool) {}

  async save(session: Session): Promise<void> {
    await this.db.query("begin");
    try {
      await this.db.query(
        `insert into sessions(id, created_at, updated_at)
         values($1, $2, $3)
         on conflict (id) do update set updated_at = excluded.updated_at`,
        [session.id, session.created_at, Date.now()],
      );
      await this.db.query("delete from messages where session_id = $1", [session.id]);
      for (const m of session.messages) {
        await this.db.query(
          `insert into messages(id, session_id, role, blocks, created_at)
           values($1, $2, $3, $4::jsonb, $5)`,
          [m.id, session.id, m.role, JSON.stringify(m.blocks), m.created_at],
        );
      }
      await this.db.query("commit");
    } catch (e) {
      await this.db.query("rollback");
      throw e;
    }
  }
}
```

### 18.11 SSE API実装例（Express）

```ts
// src/api/sse.ts
import type { Response } from "express";

export function initSSE(res: Response): void {
  res.setHeader("Content-Type", "text/event-stream; charset=utf-8");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.flushHeaders();
}

export function sendSSE(res: Response, event: string, data: unknown): void {
  res.write(`event: ${event}\n`);
  res.write(`data: ${JSON.stringify(data)}\n\n`);
}
```

```ts
// src/api/routes.ts
import express from "express";
import { initSSE, sendSSE } from "./sse";

const app = express();
app.use(express.json());

app.post("/sessions", async (_req, res) => {
  const sessionId = crypto.randomUUID();
  res.status(201).json({ session_id: sessionId });
});

app.get("/sessions/:id/events", async (req, res) => {
  initSSE(res);
  sendSSE(res, "snapshot", { session_id: req.params.id });
  const keepAlive = setInterval(() => sendSSE(res, "heartbeat", { ts: Date.now() }), 15000);
  req.on("close", () => clearInterval(keepAlive));
});
```

### 18.12 冪等制御とキャンセル例

```ts
// src/orchestrator/idempotency.ts
const inFlight = new Map<string, Promise<unknown>>();

export async function runIdempotent<T>(requestId: string, fn: () => Promise<T>): Promise<T> {
  const existing = inFlight.get(requestId);
  if (existing) return existing as Promise<T>;
  const p = fn().finally(() => inFlight.delete(requestId));
  inFlight.set(requestId, p);
  return p;
}
```

```ts
// src/orchestrator/cancel.ts
const aborters = new Map<string, AbortController>();

export function createTurnAborter(turnId: string): AbortSignal {
  const ac = new AbortController();
  aborters.set(turnId, ac);
  return ac.signal;
}

export function cancelTurn(turnId: string): boolean {
  const ac = aborters.get(turnId);
  if (!ac) return false;
  ac.abort();
  aborters.delete(turnId);
  return true;
}
```

### 18.13 テスト例（Jest）

```ts
// tests/permissionPolicy.test.ts
import { PermissionPolicy } from "../src/policy/permissionPolicy";

test("workspace-write cannot run danger tool without approval", async () => {
  const policy = new PermissionPolicy("workspace-write", { bash: "danger-full-access" });
  const decision = await policy.authorize("bash", '{"command":"rm -rf /"}');
  expect(decision.allow).toBe(false);
});
```

```ts
// tests/runTurn.test.ts
import { runTurn } from "../src/orchestrator/runTurn";

test("runTurn finishes when no tool_use exists", async () => {
  const session = { id: "s1", messages: [], created_at: Date.now(), updated_at: Date.now() };
  const model = {
    async stream() {
      return [
        { type: "text_delta", text: "こんにちは" },
        { type: "usage", usage: { input_tokens: 10, output_tokens: 5 } },
        { type: "message_stop" },
      ] as const;
    },
  };

  const summary = await runTurn({
    session,
    userInput: "hi",
    systemPrompt: ["You are helpful"],
    model,
    tools: { execute: async () => "", getSpec: () => undefined } as any,
    policy: { authorize: async () => ({ allow: true }) } as any,
    preHooks: [],
    postHooks: [],
    options: { maxIterations: 8, maxBudgetTokens: 1000 },
  });

  expect(summary.stop_reason).toBe("completed");
  expect(summary.assistant_messages.length).toBe(1);
});
```

### 18.14 設定ファイル例

```json
{
  "model": "gpt-5.4-mini",
  "maxIterations": 8,
  "maxBudgetTokens": 6000,
  "permissionMode": "workspace-write",
  "hooks": {
    "preToolUse": ["./hooks/pre.sh"],
    "postToolUse": ["./hooks/post.sh"]
  },
  "tools": {
    "read_file": { "requiredPermission": "read-only" },
    "write_file": { "requiredPermission": "workspace-write" },
    "bash": { "requiredPermission": "danger-full-access" }
  },
  "retrieval": {
    "topK": 5,
    "minScore": 0.72
  }
}
```

### 18.15 実装チェックリスト（最初の2週間）
- Day 1-2: 型定義、会話イベント、エラーコード実装。
- Day 3-4: `runTurn` の最小ループ実装（ツールなし）。
- Day 5-6: ツール実行、権限判定、拒否処理実装。
- Day 7-8: SSE配信、セッション保存。
- Day 9-10: RAG注入、出典付き回答フォーマット。
- Day 11-12: hook、タイムアウト、再試行。
- Day 13-14: テスト拡充と負荷試験。

## 19. 実装例（Python）

### 19.1 最小ディレクトリ構成例

```text
app/
  domain/
    types.py
    errors.py
  orchestrator/
    stream_assembler.py
    run_turn.py
  policy/
    permission_policy.py
  tools/
    registry.py
    hooks.py
  prompt/
    composer.py
  retrieval/
    retriever.py
  storage/
    session_repo.py
  api/
    main.py
tests/
  test_run_turn.py
  test_permission_policy.py
```

### 19.2 ドメイン型定義

```python
# app/domain/types.py
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Literal, Union
import time

MessageRole = Literal["system", "user", "assistant", "tool"]

@dataclass
class TextBlock:
    type: Literal["text"] = "text"
    text: str = ""

@dataclass
class ToolUseBlock:
    type: Literal["tool_use"] = "tool_use"
    id: str = ""
    name: str = ""
    input: str = ""

@dataclass
class ToolResultBlock:
    type: Literal["tool_result"] = "tool_result"
    tool_use_id: str = ""
    tool_name: str = ""
    output: str = ""
    is_error: bool = False

ContentBlock = Union[TextBlock, ToolUseBlock, ToolResultBlock]

@dataclass
class ConversationMessage:
    id: str
    role: MessageRole
    blocks: list[ContentBlock]
    created_at: int = field(default_factory=lambda: int(time.time() * 1000))

@dataclass
class TokenUsage:
    input_tokens: int = 0
    output_tokens: int = 0

@dataclass
class Session:
    id: str
    messages: list[ConversationMessage] = field(default_factory=list)
    created_at: int = field(default_factory=lambda: int(time.time() * 1000))
    updated_at: int = field(default_factory=lambda: int(time.time() * 1000))
```

### 19.3 エラーコード

```python
# app/domain/errors.py
from dataclasses import dataclass
from typing import Literal

ErrorCode = Literal[
    "policy_denied",
    "tool_timeout",
    "tool_failed",
    "model_failed",
    "max_iterations",
    "max_budget",
    "session_persist_failed",
]

@dataclass
class AppError(Exception):
    code: ErrorCode
    message: str
    detail: dict | None = None

    def __str__(self) -> str:
        return f"{self.code}: {self.message}"
```

### 19.4 ストリーム組み立て

```python
# app/orchestrator/stream_assembler.py
from __future__ import annotations
from dataclasses import dataclass
from app.domain.types import ConversationMessage, TextBlock, ToolUseBlock, TokenUsage

@dataclass
class AssistantEvent:
    type: str
    text: str | None = None
    id: str | None = None
    name: str | None = None
    input: str | None = None
    usage: TokenUsage | None = None

def build_assistant_message(events: list[AssistantEvent], message_id: str) -> tuple[ConversationMessage, TokenUsage | None]:
    text_buf = ""
    stopped = False
    usage: TokenUsage | None = None
    blocks = []

    def flush_text() -> None:
        nonlocal text_buf
        if text_buf:
            blocks.append(TextBlock(text=text_buf))
            text_buf = ""

    for ev in events:
        if ev.type == "text_delta" and ev.text:
            text_buf += ev.text
        elif ev.type == "tool_use":
            flush_text()
            blocks.append(ToolUseBlock(id=ev.id or "", name=ev.name or "", input=ev.input or "{}"))
        elif ev.type == "usage":
            usage = ev.usage
        elif ev.type == "message_stop":
            stopped = True

    flush_text()
    if not stopped:
        raise RuntimeError("assistant stream ended without message_stop")
    if not blocks:
        raise RuntimeError("assistant stream produced no content")

    return ConversationMessage(id=message_id, role="assistant", blocks=blocks), usage
```

### 19.5 権限ポリシー

```python
# app/policy/permission_policy.py
from __future__ import annotations
from dataclasses import dataclass
from typing import Literal, Callable, Awaitable

PermissionMode = Literal["read-only", "workspace-write", "danger-full-access", "prompt"]
Decision = dict

RANK = {
    "read-only": 1,
    "workspace-write": 2,
    "danger-full-access": 3,
    "prompt": 0,
}

@dataclass
class PermissionRequest:
    tool_name: str
    input: str
    current_mode: PermissionMode
    required_mode: PermissionMode

class PermissionPolicy:
    def __init__(self, current_mode: PermissionMode, requirements: dict[str, PermissionMode]) -> None:
        self.current_mode = current_mode
        self.requirements = requirements

    def required_mode_for(self, tool_name: str) -> PermissionMode:
        return self.requirements.get(tool_name, "danger-full-access")

    async def authorize(
        self,
        tool_name: str,
        input_json: str,
        prompt_approval: Callable[[PermissionRequest], Awaitable[Decision]] | None = None,
    ) -> Decision:
        required = self.required_mode_for(tool_name)
        if self.current_mode != "prompt" and RANK[self.current_mode] >= RANK[required]:
            return {"allow": True}
        if prompt_approval is None:
            return {
                "allow": False,
                "reason": f"tool '{tool_name}' requires {required}; current mode is {self.current_mode}",
            }
        return await prompt_approval(
            PermissionRequest(
                tool_name=tool_name,
                input=input_json,
                current_mode=self.current_mode,
                required_mode=required,
            )
        )
```

### 19.6 ツールレジストリ

```python
# app/tools/registry.py
from __future__ import annotations
from dataclasses import dataclass
from typing import Protocol, Any

@dataclass
class ToolSpec:
    name: str
    description: str
    required_permission: str

class ToolRuntime(Protocol):
    async def execute(self, input_json: Any) -> str: ...

class ToolRegistry:
    def __init__(self) -> None:
        self._specs: dict[str, ToolSpec] = {}
        self._runtimes: dict[str, ToolRuntime] = {}

    def register(self, spec: ToolSpec, runtime: ToolRuntime) -> None:
        self._specs[spec.name] = spec
        self._runtimes[spec.name] = runtime

    def spec(self, name: str) -> ToolSpec | None:
        return self._specs.get(name)

    async def execute(self, name: str, input_json: Any) -> str:
        runtime = self._runtimes.get(name)
        if runtime is None:
            raise RuntimeError(f"unsupported tool: {name}")
        return await runtime.execute(input_json)
```

### 19.7 Hook 実行

```python
# app/tools/hooks.py
from __future__ import annotations
import asyncio
import json

async def run_hooks(commands: list[str], payload: dict) -> tuple[bool, list[str]]:
    messages: list[str] = []
    for cmd in commands:
        proc = await asyncio.create_subprocess_shell(
            cmd,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        out, err = await proc.communicate(json.dumps(payload).encode("utf-8"))
        text = out.decode().strip()
        if text:
            messages.append(text)
        if proc.returncode == 2:
            return True, messages or ["hook denied"]
        if proc.returncode not in (0, 2):
            warn = err.decode().strip() or f"hook exit={proc.returncode}"
            messages.append(f"warning: {warn}")
    return False, messages
```

### 19.8 RAG注入

```python
# app/retrieval/retriever.py
from __future__ import annotations
from dataclasses import dataclass

@dataclass
class RetrievedChunk:
    source_id: str
    chunk_id: str
    content: str
    score: float
    title: str | None = None
    url: str | None = None

class Retriever:
    async def search(self, query: str, top_k: int) -> list[RetrievedChunk]:
        raise NotImplementedError
```

```python
# app/prompt/composer.py
from __future__ import annotations
from app.retrieval.retriever import RetrievedChunk

def compose_system_sections(base_instructions: list[str], retrieved: list[RetrievedChunk]) -> list[str]:
    out = list(base_instructions)
    if retrieved:
        lines = []
        for idx, c in enumerate(retrieved, start=1):
            lines.append(f"[S{idx}] {c.title or c.source_id} score={c.score:.3f}\n{c.content}")
        out.extend([
            "# Retrieval Context",
            "以下は検索結果です。回答では必要に応じて出典参照を明示してください。",
            "\n\n".join(lines),
        ])
    return out
```

### 19.9 会話ループ（中核）

```python
# app/orchestrator/run_turn.py
from __future__ import annotations
import json
import uuid
from dataclasses import dataclass
from app.domain.types import Session, ConversationMessage, TextBlock, ToolResultBlock, TokenUsage
from app.orchestrator.stream_assembler import build_assistant_message, AssistantEvent
from app.policy.permission_policy import PermissionPolicy
from app.tools.registry import ToolRegistry
from app.tools.hooks import run_hooks

@dataclass
class TurnSummary:
    iterations: int
    assistant_messages: list[ConversationMessage]
    tool_results: list[ConversationMessage]
    usage: TokenUsage
    stop_reason: str

class ModelGateway:
    async def stream(self, system_prompt: list[str], messages: list[ConversationMessage]) -> list[AssistantEvent]:
        raise NotImplementedError

async def run_turn(
    *,
    session: Session,
    user_input: str,
    system_prompt: list[str],
    model: ModelGateway,
    tools: ToolRegistry,
    policy: PermissionPolicy,
    pre_hooks: list[str],
    post_hooks: list[str],
    max_iterations: int = 8,
    max_budget_tokens: int = 6000,
) -> TurnSummary:
    session.messages.append(
        ConversationMessage(
            id=str(uuid.uuid4()),
            role="user",
            blocks=[TextBlock(text=user_input)],
        )
    )

    iterations = 0
    usage = TokenUsage()
    assistant_messages: list[ConversationMessage] = []
    tool_results: list[ConversationMessage] = []

    while True:
        iterations += 1
        if iterations > max_iterations:
            return TurnSummary(iterations, assistant_messages, tool_results, usage, "max_iterations")

        events = await model.stream(system_prompt, session.messages)
        assistant_msg, turn_usage = build_assistant_message(events, str(uuid.uuid4()))
        if turn_usage:
            usage.input_tokens += turn_usage.input_tokens
            usage.output_tokens += turn_usage.output_tokens

        session.messages.append(assistant_msg)
        assistant_messages.append(assistant_msg)

        if usage.input_tokens + usage.output_tokens > max_budget_tokens:
            return TurnSummary(iterations, assistant_messages, tool_results, usage, "max_budget")

        uses = [b for b in assistant_msg.blocks if getattr(b, "type", "") == "tool_use"]
        if not uses:
            return TurnSummary(iterations, assistant_messages, tool_results, usage, "completed")

        for tool_use in uses:
            decision = await policy.authorize(tool_use.name, tool_use.input)
            if not decision.get("allow", False):
                denied = ConversationMessage(
                    id=str(uuid.uuid4()),
                    role="tool",
                    blocks=[
                        ToolResultBlock(
                            tool_use_id=tool_use.id,
                            tool_name=tool_use.name,
                            output=decision.get("reason", "denied"),
                            is_error=True,
                        )
                    ],
                )
                session.messages.append(denied)
                tool_results.append(denied)
                continue

            pre_denied, pre_messages = await run_hooks(
                pre_hooks,
                {
                    "hook_event_name": "PreToolUse",
                    "tool_name": tool_use.name,
                    "tool_input_json": tool_use.input,
                },
            )
            if pre_denied:
                denied = ConversationMessage(
                    id=str(uuid.uuid4()),
                    role="tool",
                    blocks=[
                        ToolResultBlock(
                            tool_use_id=tool_use.id,
                            tool_name=tool_use.name,
                            output="\n".join(pre_messages) or "PreToolUse hook denied",
                            is_error=True,
                        )
                    ],
                )
                session.messages.append(denied)
                tool_results.append(denied)
                continue

            output = ""
            is_error = False
            try:
                parsed = json.loads(tool_use.input)
                output = await tools.execute(tool_use.name, parsed)
            except Exception as e:  # noqa: BLE001
                is_error = True
                output = str(e)

            post_denied, post_messages = await run_hooks(
                post_hooks,
                {
                    "hook_event_name": "PostToolUse",
                    "tool_name": tool_use.name,
                    "tool_input_json": tool_use.input,
                    "tool_output": output,
                    "tool_result_is_error": is_error,
                },
            )
            if post_messages:
                output = f"{output}\n\nHook feedback:\n" + "\n".join(post_messages)
            if post_denied:
                is_error = True

            result_msg = ConversationMessage(
                id=str(uuid.uuid4()),
                role="tool",
                blocks=[
                    ToolResultBlock(
                        tool_use_id=tool_use.id,
                        tool_name=tool_use.name,
                        output=output,
                        is_error=is_error,
                    )
                ],
            )
            session.messages.append(result_msg)
            tool_results.append(result_msg)
```

### 19.10 永続化（SQLite簡易例）

```python
# app/storage/session_repo.py
from __future__ import annotations
import json
import sqlite3
from app.domain.types import Session, ConversationMessage

class SessionRepository:
    def __init__(self, path: str = "app.db") -> None:
        self.conn = sqlite3.connect(path)
        self.conn.execute(
            "create table if not exists sessions (id text primary key, payload text not null)"
        )
        self.conn.commit()

    def save(self, session: Session) -> None:
        payload = {
            "id": session.id,
            "messages": [self._msg_to_dict(m) for m in session.messages],
            "created_at": session.created_at,
            "updated_at": session.updated_at,
        }
        self.conn.execute(
            "insert or replace into sessions(id, payload) values(?, ?)",
            (session.id, json.dumps(payload)),
        )
        self.conn.commit()

    @staticmethod
    def _msg_to_dict(m: ConversationMessage) -> dict:
        return {
            "id": m.id,
            "role": m.role,
            "blocks": [b.__dict__ for b in m.blocks],
            "created_at": m.created_at,
        }
```

### 19.11 API（FastAPI + SSE）

```python
# app/api/main.py
from __future__ import annotations
import asyncio
import json
import uuid
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/sessions")
async def create_session() -> dict:
    return {"session_id": str(uuid.uuid4())}

@app.get("/sessions/{session_id}/events")
async def session_events(session_id: str) -> StreamingResponse:
    async def gen():
        yield f"event: snapshot\ndata: {json.dumps({'session_id': session_id})}\n\n"
        while True:
            await asyncio.sleep(15)
            yield f"event: heartbeat\ndata: {json.dumps({'ts': int(asyncio.get_event_loop().time())})}\n\n"

    return StreamingResponse(gen(), media_type="text/event-stream")
```

### 19.12 テスト例（pytest）

```python
# tests/test_permission_policy.py
import pytest
from app.policy.permission_policy import PermissionPolicy

@pytest.mark.asyncio
async def test_workspace_write_cannot_run_danger_without_prompt():
    policy = PermissionPolicy("workspace-write", {"bash": "danger-full-access"})
    decision = await policy.authorize("bash", '{"command":"rm -rf /"}')
    assert decision["allow"] is False
```

```python
# tests/test_run_turn.py
import pytest
from app.domain.types import Session
from app.orchestrator.stream_assembler import AssistantEvent
from app.orchestrator.run_turn import run_turn

class DummyModel:
    async def stream(self, _system_prompt, _messages):
        return [
            AssistantEvent(type="text_delta", text="こんにちは"),
            AssistantEvent(type="usage", usage=type("U", (), {"input_tokens": 10, "output_tokens": 5})()),
            AssistantEvent(type="message_stop"),
        ]

class DummyTools:
    async def execute(self, _name, _input_json):
        return "ok"

class DummyPolicy:
    async def authorize(self, _name, _input_json):
        return {"allow": True}

@pytest.mark.asyncio
async def test_run_turn_no_tool_use_completed():
    session = Session(id="s1")
    summary = await run_turn(
        session=session,
        user_input="hi",
        system_prompt=["You are helpful"],
        model=DummyModel(),
        tools=DummyTools(),
        policy=DummyPolicy(),
        pre_hooks=[],
        post_hooks=[],
    )
    assert summary.stop_reason == "completed"
    assert len(summary.assistant_messages) == 1
```

## 20. マスター仕様補強（抜け漏れ防止）

本章は、実装例中心の前章を「本番運用可能な設計規範」へ引き上げるための補強要件である。  
本章の `MUST` は優先的に実装し、`SHOULD` は Phase 3 までに、`MAY` は運用負荷に応じて導入する。

## 21. 不変条件（System Invariants）

### 21.1 会話整合性不変条件
- `INV-001` (`MUST`): すべての `tool_use` には最終的に対応する `tool_result` が存在すること。
- `INV-002` (`MUST`): 1セッション内のメッセージ順序は単調増加時刻または単調増加シーケンス番号で確定可能であること。
- `INV-003` (`MUST`): `assistant` メッセージの途中ストリームが失敗した場合、未完了メッセージを確定履歴に残さないこと。
- `INV-004` (`MUST`): `stop_reason` は空で終わらないこと（`completed|max_iterations|max_budget|aborted|error` のいずれか）。
- `INV-005` (`MUST`): 同一 `request_id` の多重実行時、結果は冪等に1回分として扱うこと。

### 21.2 セキュリティ不変条件
- `INV-006` (`MUST`): ツール実行前に権限判定を必ず通すこと。
- `INV-007` (`MUST`): 権限拒否時は、理由を機械可読コードと人間可読文で返すこと。
- `INV-008` (`MUST`): 監査ログからAPIキー等の秘密情報を復元できないこと。

### 21.3 可観測性不変条件
- `INV-009` (`MUST`): 全ターンに `trace_id`、全外部依存呼び出しに `span_id` を付与すること。
- `INV-010` (`MUST`): エラーは分類コード付きで集計可能であること。

## 22. 役割分担（RACI）

| 領域 | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| 会話オーケストレータ | Backend | Tech Lead | ML Engineer | PM |
| プロンプト設計 | ML Engineer | Tech Lead | Backend | PM |
| ツールランタイム | Backend | Tech Lead | SRE | Security |
| ポリシー/承認 | Security | Security Lead | Backend | PM |
| SLO/監視 | SRE | SRE Lead | Backend | PM |
| リリース判定 | Tech Lead | Engineering Manager | SRE/Security | 全員 |

## 23. 契約仕様（Contract-First）

### 23.1 イベント契約（SSE/WS）
- `message_start`
- `assistant_delta`
- `tool_use`
- `tool_result`
- `usage`
- `message_stop`
- `error`

### 23.2 イベント契約JSONスキーマ（例）

```json
{
  "event": "tool_result",
  "trace_id": "trc_...",
  "session_id": "ses_...",
  "turn_id": "turn_...",
  "payload": {
    "tool_use_id": "tool_...",
    "tool_name": "read_file",
    "is_error": false,
    "output": "..."
  },
  "ts": 1712200000000
}
```

### 23.3 後方互換ポリシー
- `MUST`: 既存クライアントが参照する必須フィールドは破壊変更しない。
- `MUST`: 削除ではなく `deprecate -> dual-write -> remove` の3段階移行を行う。
- `SHOULD`: スキーマバージョンをイベントに埋め込む（`schema_version`）。

## 24. データライフサイクル要件

### 24.1 保存
- `MUST`: セッション本体、メッセージ、ツール実行ログを論理分離して保存。
- `MUST`: 監査ログは改ざん検知可能なストレージに保管。

### 24.2 保持期間
- 会話本文: 30〜180日（事業要件による）。
- 監査メタデータ: 180〜400日。
- セキュリティイベント: 400日以上推奨。

### 24.3 削除
- `MUST`: ユーザー削除要求に対するハードデリートまたは不可逆匿名化に対応。
- `MUST`: バックアップにも削除反映手順を持つ。

## 25. SLI / SLO / エラーバジェット

### 25.1 SLI 定義
- `SLI-001`: TTFT（Time To First Token）
- `SLI-002`: Turn completion success rate
- `SLI-003`: Tool execution success rate
- `SLI-004`: Session resume success rate
- `SLI-005`: Permission decision latency

### 25.2 SLO 目標
- `SLO-001`: `SLI-002 >= 99.5%`（月間）
- `SLO-002`: `SLI-001 p95 < 2.5s`（ツールなし）
- `SLO-003`: `SLI-001 p95 < 8s`（ツールあり）
- `SLO-004`: `SLI-004 >= 99.9%`

### 25.3 エラーバジェット運用
- 月間エラーバジェット超過時:
  - 新機能開発を一時停止。
  - 信頼性改善タスクへ自動切替。
  - 1週間で再評価。

## 26. 容量計画（Capacity Planning）

### 26.1 入力パラメータ
- 同時セッション数
- 1ターンあたり平均トークン
- ツール呼び出し率
- 平均ツール実行時間

### 26.2 見積ルール
- `QPS_model = active_sessions * turn_rate * (1 + retry_rate)`
- `QPS_tool = active_sessions * turn_rate * tool_call_rate`
- `worker_count >= ceil(QPS_tool * p95_tool_latency_seconds * safety_factor)`

### 26.3 安全率
- `safety_factor = 1.5` を初期値とし、実績に応じて見直す。

## 27. 障害対応・復旧設計（DR）

### 27.1 障害レベル定義
- `SEV-1`: 全面停止 / 重大データ欠損
- `SEV-2`: 主要機能停止（例: ツール実行不能）
- `SEV-3`: 部分機能劣化

### 27.2 復旧目標
- `RTO`: 15分以内（SEV-1）
- `RPO`: 5分以内（セッションメタデータ）

### 27.3 復旧手順（最低要件）
- `MUST`: セッション再開APIのヘルスチェックを優先。
- `MUST`: モデル依存障害時はフォールバックモデルへ切替可能。
- `MUST`: ツール層障害時は「ツール停止モード」で会話継続可能。

## 28. セキュリティ詳細要件（STRIDE）

| 分類 | 主要リスク | 主要対策 |
|---|---|---|
| Spoofing | セッションなりすまし | 短寿命トークン、デバイスバインド |
| Tampering | ログ改ざん | 追記専用監査ログ、署名 |
| Repudiation | 実行否認 | 監査証跡、決定理由保存 |
| Information Disclosure | 秘密漏えい | マスキング、暗号化、最小権限 |
| Denial of Service | 連続重負荷 | レート制限、キュー、サーキットブレーカ |
| Elevation of Privilege | 権限昇格 | ツール別権限宣言、承認必須 |

### 28.1 プロンプトインジェクション対策
- `MUST`: 外部取得テキストを「命令」ではなく「データ」として扱う。
- `MUST`: システム指示より下位優先のコンテキスト領域を明示。
- `SHOULD`: 危険指示検知ルール（regex/分類器）を導入。

## 29. 品質評価フレームワーク（自然さ/有用性）

### 29.1 オフライン評価
- 既知質問セット（FAQ/業務タスク）で回帰評価。
- 指標:
  - 正確性
  - 指示追従性
  - 根拠提示率
  - 不要拒否率

### 29.2 オンライン評価
- クリック率ではなくタスク完了率を主指標にする。
- ユーザー評価（5段階）と再質問率を併用する。

### 29.3 ガードレール評価
- 危険要求への拒否率
- 個人情報漏えい率
- 承認フロー逸脱率

## 30. 段階的実装計画（最小構成からの補強）

### Phase A: Core Loop（最小）
- 1モデル
- 1セッション
- ストリーミング
- ツールなし

### Phase B: Tool Basic
- ツールレジストリ
- 権限モード
- 拒否理由注入

### Phase C: Reliability
- タイムアウト
- 再試行
- 履歴圧縮
- 冪等

### Phase D: RAG + Governance
- 検索注入
- 出典管理
- 監査と評価基盤

### Phase E: Production Hardening
- SLO運用
- DR訓練
- セキュリティレビュー
- コスト最適化

## 31. トレーサビリティマトリクス（要求→実装→検証）

| Requirement | 実装モジュール | テスト種別 | 監視指標 |
|---|---|---|---|
| FR-004 | orchestrator + api/sse | 結合 + E2E | TTFT |
| FR-012 | policy engine | 単体 | permission latency |
| FR-018 | tool runtime | 単体 + 結合 | tool error rate |
| FR-019 | prompt composer + retriever | 結合 | retrieval hit rate |
| FR-025 | cancel module | E2E | abort completion rate |

## 32. リリースゲート（Go/No-Go）

### 32.1 機能ゲート
- `MUST`: FRのMUST項目100%実装済み。
- `MUST`: 主要API契約テスト100%通過。

### 32.2 品質ゲート
- `MUST`: 重大バグ（P1/P2）ゼロ。
- `MUST`: E2E主要シナリオ成功率 >= 99%。
- `SHOULD`: 負荷試験で目標同時接続を満たす。

### 32.3 運用ゲート
- `MUST`: ダッシュボードとアラート定義完了。
- `MUST`: 当番体制とRunbook確定。
- `MUST`: ロールバック手順検証済み。

## 33. 変更管理（Change Management）

### 33.1 変更クラス
- `Class-1`: 互換性影響なし（設定追加）
- `Class-2`: 互換性影響あり（イベント仕様変更）
- `Class-3`: セキュリティ/可用性へ影響

### 33.2 承認フロー
- `Class-1`: Tech Lead承認
- `Class-2`: Tech Lead + QA承認
- `Class-3`: Tech Lead + Security + SRE承認

### 33.3 変更テンプレート
- 変更理由
- 影響範囲
- 互換性評価
- ロールバック案
- 検証結果

## 34. 実装開始前チェック（必須）
- 要件IDと実装タスクが1対1対応している。
- APIスキーマが確定している。
- エラーコード一覧が確定している。
- ログ項目とマスキング方針が確定している。
- SLOとアラート閾値が確定している。
- フェーズ分割（A〜E）の完了条件が合意されている。

## 35. 最終漏れ監査（Critical Gap Audit）

### 35.1 監査結論
- 現行仕様は、対話ループ、ツール実行、RAG注入、監査、SLO、DRまで含み、汎用対話基盤として高水準である。
- ただし、複数アプリ・複数顧客環境へ展開する際に事故要因となる領域（認証連携、テナント分離、コスト統制、有人エスカレーション、実験統制）の明文化が不足していた。
- 以下を追加必須要件として定義する。

### 35.2 追加機能要件（FR-031〜FR-045）

| ID | 要件 | 優先度 | 受入基準 |
|---|---|---|---|
| FR-031 | 認証済みユーザーのみセッション操作できること（JWT/OIDC等） | MUST | 未認証リクエストは `401` で拒否される |
| FR-032 | 全データに `tenant_id` を付与しテナント分離を強制すること | MUST | 異なる `tenant_id` 間で参照不可 |
| FR-033 | 同一セッション同時更新時に競合制御できること（optimistic lock） | MUST | `revision` 不一致で `409` を返す |
| FR-034 | ユーザー/テナント/ツール単位のレート制限を適用できること | MUST | 制限超過時に `429` と再試行情報を返す |
| FR-035 | セッション/日次/テナント単位のコスト上限を設定できること | MUST | 上限超過時は安全停止し理由を返す |
| FR-036 | 入力/出力/ツール結果へモデレーションを適用できること | MUST | ブロック/マスク/要確認の判定が残る |
| FR-037 | 自動応答不能時に有人エスカレーションできること | SHOULD | `handoff_requested` 状態へ遷移可能 |
| FR-038 | プロンプトテンプレートを版管理し即時ロールバックできること | MUST | `prompt_version` 指定で再現可能 |
| FR-039 | モデルルーティングとフォールバック切替ができること | MUST | 主要モデル障害時に代替へ切替可能 |
| FR-040 | 構造化出力スキーマを強制検証できること | SHOULD | スキーマ不一致時に修復再試行される |
| FR-041 | 低信頼・曖昧入力時に確認質問へ遷移できること | SHOULD | 不明瞭入力で `clarification` 応答になる |
| FR-042 | 長期記憶の書込条件（同意・閾値）を制御できること | SHOULD | 同意なしで長期記憶へ保存されない |
| FR-043 | 記憶の訂正・忘却要求に応答できること | SHOULD | 指定エントリの削除/匿名化が可能 |
| FR-044 | すべてのターンで `prompt/model/policy` 版を追跡できること | MUST | トレースに3種のバージョンが記録される |
| FR-045 | 機能フラグ/A-Bテストで挙動を安全切替できること | MAY | フラグ単位で対象ユーザーを限定可能 |

### 35.3 追加非機能要件（NFR-011〜NFR-020）

| ID | 要件 | 目標 |
|---|---|---|
| NFR-011 | テナント分離安全性 | クロステナント漏えい 0件 |
| NFR-012 | 認証基盤連携可用性 | 認証連携成功率 99.9%以上 |
| NFR-013 | レート制限判定性能 | 判定レイテンシ p95 < 20ms |
| NFR-014 | モデレーション判定性能 | 追加遅延 p95 < 300ms |
| NFR-015 | コスト超過検知遅延 | 5分以内 |
| NFR-016 | フォールバック切替時間 | 60秒以内 |
| NFR-017 | 有人引継ぎ開始SLA | 要請後 5分以内 |
| NFR-018 | 忘却/削除要求SLA | 7日以内 |
| NFR-019 | プロンプト再現性 | 監査ID指定で同一版を再実行可能 |
| NFR-020 | 実験の安全性 | A/B適用で品質劣化を自動検知し停止可能 |

## 36. 追加データ契約要件（漏れ防止）

### 36.1 追加必須フィールド
- `session`: `tenant_id`, `user_id`, `revision`, `cost_budget`, `cost_spent`
- `message`: `moderation_status`, `prompt_version`, `model_version`, `policy_version`
- `tool_call`: `idempotency_key`, `approved_by`, `risk_level`
- `trace`: `experiment_id`, `flag_set`, `fallback_from`, `fallback_to`

### 36.2 互換性ルール
- `MUST`: 新フィールドは追加互換を原則とし既存必須の意味を変更しない。
- `MUST`: 変更時は `schema_version` を更新し、少なくとも1リリースは旧版を受理する。

## 37. 追加API要件（運用実装向け）
- `GET /tenants/{tenant_id}/usage`（コスト/トークン消費）
- `POST /sessions/{id}/handoff`（有人引継ぎ要求）
- `POST /sessions/{id}/memory/forget`（記憶忘却要求）
- `GET /sessions/{id}/versions/{turn_id}`（prompt/model/policy版参照）
- `POST /admin/prompts/rollback`（即時ロールバック）

## 38. 追加テスト要件（欠落検知用）

### 38.1 E2E必須シナリオ
- 未認証アクセス拒否（FR-031）
- クロステナント参照拒否（FR-032）
- 同時更新競合（FR-033）
- レート制限超過（FR-034）
- コスト上限到達停止（FR-035）
- モデレーション遮断/マスク（FR-036）
- フォールバック切替（FR-039）
- バージョン付き再現実行（FR-044）

### 38.2 カオス/回帰
- 認証基盤遅延時の縮退挙動
- 主要モデル障害時の代替経路
- 特定ツール高エラー率時のサーキットブレーカ挙動

## 39. 最終Go/No-Go補強条件
- `MUST`: FR-031〜FR-039 を本番前に満たすこと。
- `MUST`: クロステナント漏えい試験を通過していること。
- `MUST`: 1週間のステージング運用で SLO違反がエラーバジェット内であること。
- `SHOULD`: FR-040〜FR-045 を Phase E までに段階導入すること。
