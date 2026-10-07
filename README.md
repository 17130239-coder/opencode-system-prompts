# OpenCode System Prompts Collection

Thư mục này chứa đầy đủ các file system prompt nguyên bản của dự án [OpenCode](https://github.com/anomalyco/opencode) (`packages/opencode/src/session/prompt/`) được chia thành 2 phiên bản: **`.txt`** và **`.md`**, kèm theo mã nguồn TypeScript điều phối runtime.

## Cấu trúc thư mục

- **`./` (Root)**: Các file định dạng gốc `.txt`
- **[`BUILD_AND_PLAN.md`](./BUILD_AND_PLAN.md)**: 📖 **Tài liệu chuyên sâu về kiến trúc, cơ chế quyền hạn và prompt của BUILD & PLAN**
- **[`./markdown/`](./markdown/)**: Phiên bản định dạng Markdown `.md` (tiện đọc và preview trong IDE)
- **[`./pipeline/`](./pipeline/)**: Mã nguồn logic lắp ráp prompt động (`system.ts`, `instruction.ts`, `reminders.ts`)
- **[`./subagents/`](./subagents/)**: Prompts cho các subagent phụ trợ (`explore`, `compaction`, `summary`, `title`, `generate`)

---

## Bảng tra cứu nhanh các Prompt

| Tên File (.txt) | Tên File (.md) | Mô hình / Chức năng |
| :--- | :--- | :--- |
| [`anthropic.txt`](./anthropic.txt) | [`anthropic.md`](./markdown/anthropic.md) | Anthropic Claude (Claude 3.5 / 3.7 Sonnet) |
| [`gemini.txt`](./gemini.txt) | [`gemini.md`](./markdown/gemini.md) | Google Gemini (Gemini 1.5 / 2.0 Pro & Flash) |
| [`beast.txt`](./beast.txt) | [`beast.md`](./markdown/beast.md) | OpenAI Reasoning & Top-tier (GPT-4, o1, o3) |
| [`gpt.txt`](./gpt.txt) | [`gpt.md`](./markdown/gpt.md) | OpenAI GPT tiêu chuẩn |
| [`gpt-astra.txt`](./gpt-astra.txt) | [`gpt-astra.md`](./markdown/gpt-astra.md) | GPT-6 / Astra experimental |
| [`kimi.txt`](./kimi.txt) | [`kimi.md`](./markdown/kimi.md) | Moonshot AI (Kimi) |
| [`meta.txt`](./meta.txt) | [`meta.md`](./markdown/meta.md) | Meta / Muse models (Spark, Glimmer) |
| [`codex.txt`](./codex.txt) | [`codex.md`](./markdown/codex.md) | OpenAI Codex |
| [`copilot-gpt-5.txt`](./copilot-gpt-5.txt) | [`copilot-gpt-5.md`](./markdown/copilot-gpt-5.md) | GitHub Copilot / GPT-5 |
| [`trinity.txt`](./trinity.txt) | [`trinity.md`](./markdown/trinity.md) | Trinity model |
| [`default.txt`](./default.txt) | [`default.md`](./markdown/default.md) | Default Fallback prompt |
| [`plan.txt`](./plan.txt) | [`plan.md`](./markdown/plan.md) | Standard Plan Agent |
| [`plan-mode.txt`](./plan-mode.txt) | [`plan-mode.md`](./markdown/plan-mode.md) | Experimental Plan Mode |
| [`plan-reminder-anthropic.txt`](./plan-reminder-anthropic.txt) | [`plan-reminder-anthropic.md`](./markdown/plan-reminder-anthropic.md) | Plan Reminder cho Claude |
| [`build-switch.txt`](./build-switch.txt) | [`build-switch.md`](./markdown/build-switch.md) | Switch notification: Plan $\rightarrow$ Build |
