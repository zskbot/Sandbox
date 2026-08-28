# Thiết kế tích hợp OpenAI với Sandbox AI cho OpenClaw/Eios

Tài liệu này mô tả thiết kế tích hợp OpenAI Platform vào Sandbox CodeX để hiện thực hóa OpenClaw: trợ lý AI chính của Eios, vận hành theo pipeline tự động từ ý tưởng đến triển khai production.

## Mục tiêu sản phẩm

OpenClaw biến yêu cầu tự nhiên của người dùng thành ứng dụng chạy được bằng chuỗi tự động:

```text
Human Intent
↓
LLM Planner
↓
Code Generator
↓
Workflow Runtime
↓
Container + Infra
↓
Deployment
↓
Self-healing Debug Loop
```

Mục tiêu của tích hợp OpenAI là cung cấp lớp trí tuệ trung tâm cho các năng lực sau:

- Phân tích intent, phạm vi, ràng buộc và tiêu chí hoàn thành.
- Sinh kiến trúc, chọn stack, tạo backlog và chia nhỏ tác vụ.
- Sinh frontend, backend, API routes, database schema, auth và worker.
- Gọi công cụ sandbox để đọc/ghi file, chạy lệnh, kiểm thử và sửa lỗi.
- Đọc log runtime realtime, phát hiện crash, tạo bản vá và rerun.
- Tổng hợp artifact build, ghi chú triển khai và trạng thái publish live URL.

## Thành phần kiến trúc

```text
Browser IDE / Chat UI
        ↓
OpenClaw Orchestrator API
        ↓
OpenAI Responses API / Agents SDK
        ↓
Tool Gateway
        ↓
Sandbox Runtime ── Container Pool ── Database ── Secrets ── Networking
        ↓
Workflow Engine
        ↓
Build, Deploy, Observability, Self-healing Loop
```

### 1. Browser IDE / Chat UI

Giao diện tiếp nhận yêu cầu, hiển thị kế hoạch, diff, terminal log, preview URL và trạng thái workflow. UI không giữ API key OpenAI; mọi request AI đi qua backend.

### 2. OpenClaw Orchestrator API

Backend điều phối một phiên làm việc gồm:

- `project_id`, `workspace_id`, `conversation_id`, `run_id`.
- Trạng thái intent, plan, tasks, tool calls và deployment.
- Chính sách sandbox: quyền file system, network, secrets, command allowlist và resource limit.
- Streaming event cho UI: token, tool call, log, patch, test result và deployment result.

### 3. OpenAI agent layer

Dùng OpenAI Responses API cho vòng lặp stateful, multimodal và tool-using; dùng Agents SDK khi cần orchestration nhiều agent, handoff, guardrails, tracing và workflow agentic phức tạp. Các vai trò đề xuất:

| Agent | Trách nhiệm | Tool chính |
| --- | --- | --- |
| Planner | Phân tích intent, tạo kiến trúc, chia task | project context, docs search |
| Architect | Chọn stack, schema, API contract, security model | repo scan, dependency catalog |
| Coder | Sinh/sửa code theo task | file read/write, apply patch |
| Runner | Chạy install, test, build, workflow | shell sandbox, package manager |
| Debugger | Đọc log, phân loại lỗi, tạo patch | log stream, test report |
| Release | Build artifact, domain, HTTPS, rollout notes | deploy API, env manager |

### 4. Tool Gateway

Tool Gateway là lớp duy nhất cho phép model tác động vào sandbox. Mỗi tool phải có schema rõ ràng, audit log, timeout và policy kiểm soát.

Tool tối thiểu:

- `read_file`, `write_file`, `apply_patch`, `list_files`.
- `run_command` với timeout, cwd, env allowlist và output truncation.
- `install_dependencies` qua package manager được phát hiện tự động.
- `start_workflow`, `stop_workflow`, `read_logs`, `get_preview_url`.
- `create_database`, `run_migration`, `seed_database`.
- `set_secret`, `get_secret_ref` để không lộ secret plaintext cho model.
- `deploy_project`, `map_domain`, `rollback_deployment`.

### 5. Sandbox Runtime

Runtime tạo môi trường cô lập cho từng workspace:

- Container ephemeral hoặc persistent theo project.
- Giới hạn CPU, RAM, disk, network egress và thời gian chạy.
- Filesystem snapshot để rollback sau patch lỗi.
- Dependency cache để giảm thời gian install.
- Realtime log stream cho frontend và Debugger agent.

### 6. Workflow Engine

Workflow Engine chạy song song frontend, backend và worker bằng một nút Run:

```yaml
workflows:
  dev:
    parallel:
      - name: frontend
        command: npm run dev -- --host 0.0.0.0
        port: 5173
      - name: backend
        command: npm run api
        port: 3000
      - name: worker
        command: npm run worker
```

Engine cần phát sự kiện chuẩn hóa:

- `workflow.started`
- `process.log`
- `process.crashed`
- `healthcheck.failed`
- `test.failed`
- `patch.applied`
- `workflow.ready`
- `deployment.published`

## Luồng tự động generate → run → patch → rerun

```text
1. User gửi yêu cầu
2. Planner tạo plan + acceptance criteria
3. Architect chọn stack + cấu trúc project
4. Coder tạo code trong workspace
5. Runner cài dependency + chạy workflow
6. Debugger đọc log/test/build output
7. Nếu lỗi: tạo patch, áp dụng, rerun từ bước 5
8. Nếu pass: Release build artifact + deploy + trả live URL
```

Pseudo-state machine:

```text
INTAKE → PLAN → GENERATE → RUN → OBSERVE
                         ↘      ↓
                          PATCH ← FAIL
                           ↓
                         VERIFY → DEPLOY → DONE
```

## Chiến lược tích hợp OpenAI API

### Backend-only key management

- Lưu `OPENAI_API_KEY` trong secret manager của Eios, không đưa xuống browser.
- Mỗi tenant/project có quota, budget và model policy riêng.
- Gắn metadata `project_id`, `workspace_id`, `run_id` vào request để trace chi phí và lỗi.

### Model policy

- Dùng model reasoning mạnh cho Planner, Architect và Debugger.
- Dùng model nhanh/kinh tế hơn cho tác vụ lặp lại như tóm tắt log hoặc tạo commit note.
- Cho phép cấu hình theo project: `quality`, `balanced`, `fast`.

### State và memory

- Conversation state lưu trong backend theo `conversation_id`.
- Project memory gồm kiến trúc hiện tại, dependency, quyết định kỹ thuật, lỗi đã sửa và deployment history.
- Không đưa toàn bộ repo vào prompt; dùng retrieval theo file/task để giảm token và rủi ro nhiễu.

### Guardrails

- Trước khi chạy lệnh nguy hiểm, Tool Gateway kiểm tra allowlist và quyền workspace.
- Không cho model đọc secret plaintext; chỉ dùng secret reference.
- Chặn lệnh phá hoại như xóa root, fork bomb, exfiltration token hoặc crypto miner.
- Yêu cầu human approval cho thao tác production nhạy cảm: xóa database, rotate secret, rollback lớn, thay domain chính.

## Data model đề xuất

| Bảng | Mục đích |
| --- | --- |
| `projects` | Metadata dự án, owner, stack, trạng thái |
| `workspaces` | Sandbox/container gắn với project |
| `conversations` | Phiên chat và state OpenClaw |
| `runs` | Một lần thực thi plan/generate/run/deploy |
| `tasks` | Task nhỏ do Planner tạo |
| `tool_calls` | Audit mọi tool call, input redacted, output summary |
| `runtime_events` | Log/process/test/build/deploy events |
| `patches` | Diff do agent tạo, trạng thái apply/rollback |
| `deployments` | Artifact, URL, domain, version, rollback target |
| `secrets` | Secret reference, provider, scope, rotation metadata |

## API routes đề xuất

| Route | Chức năng |
| --- | --- |
| `POST /api/openclaw/chat` | Nhận yêu cầu người dùng, stream agent events |
| `POST /api/openclaw/runs` | Tạo run từ plan hoặc prompt |
| `GET /api/openclaw/runs/:id/events` | SSE/WebSocket stream trạng thái |
| `POST /api/workspaces` | Khởi tạo workspace sandbox |
| `POST /api/workspaces/:id/tools/run-command` | Chạy command qua Tool Gateway |
| `GET /api/workspaces/:id/logs` | Stream log runtime |
| `POST /api/workflows/:id/start` | Chạy frontend/backend/worker song song |
| `POST /api/deployments` | Build và publish live URL |
| `POST /api/deployments/:id/rollback` | Rollback phiên bản |

## Prompt contract cho OpenClaw

System instruction nên cố định vai trò, quyền hạn và vòng lặp:

```text
Bạn là OpenClaw, AI software factory agent của Eios.
Nhiệm vụ: biến intent của user thành ứng dụng production-ready trong sandbox.
Luôn làm theo vòng lặp: clarify only if blocked → plan → generate → run → observe → patch → verify → deploy.
Chỉ thao tác qua tools được cấp. Không tiết lộ secret. Luôn tạo patch nhỏ, kiểm thử sau mỗi thay đổi, và báo trạng thái bằng event có cấu trúc.
```

Output schema cho plan:

```json
{
  "intent": "string",
  "architecture": "string",
  "stack": ["string"],
  "tasks": [
    { "id": "T1", "title": "string", "acceptance": ["string"] }
  ],
  "risks": ["string"],
  "next_action": "generate|ask_user|run|deploy"
}
```

## Roadmap triển khai

### Phase 1: Agent foundation

- Tạo Orchestrator API với streaming events.
- Tích hợp OpenAI backend-only.
- Cài Tool Gateway cho file, patch, command và log.
- Tạo một agent OpenClaw đơn giản: plan → code → run.

### Phase 2: Sandbox runtime

- Container workspace, dependency manager và process supervisor.
- Workflow Engine chạy nhiều process song song.
- Realtime log stream và preview URL.
- Snapshot/rollback filesystem.

### Phase 3: Self-healing loop

- Phân loại lỗi install/build/runtime/test.
- Tự tạo patch từ log và rerun có giới hạn số vòng.
- Lưu knowledge lỗi đã sửa vào project memory.
- Dashboard hiển thị diff, test và confidence.

### Phase 4: Production deployment

- Build artifact, HTTPS, domain mapping và rollback.
- Secret manager theo project/environment.
- Autoscaling policy và health checks.
- Cost, rate limit, audit log và team permissions.

## Tiêu chí hoàn thành MVP

- User nhập một yêu cầu web app đơn giản và nhận được preview URL.
- OpenClaw tạo plan, sinh code, chạy workflow và hiển thị log realtime.
- Khi build/runtime lỗi, OpenClaw tự patch ít nhất một vòng và rerun.
- API key OpenAI không bao giờ xuất hiện ở browser hoặc log.
- Mọi tool call được audit theo run và workspace.
- Có đường deploy tạo live URL HTTPS cho bản pass kiểm thử.

## Tài liệu OpenAI liên quan

- Responses API: endpoint hợp nhất cho tương tác stateful, multimodal, tool-using trong agentic workflow.
- Agents SDK: toolkit để xây agent có tools, context, handoffs, streaming, tracing và eval.
- Tools/function calling: cách định nghĩa hành động an toàn để model gọi qua schema.
- Conversation state, streaming, background mode và evals là các phần nên dùng khi mở rộng runtime production.