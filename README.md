# Knowledge AI for Obsidian

[简体中文](./README.zh-CN.md) · English

An AI assistant for your Obsidian vault: ask questions and get cited answers grounded in your notes, generate study guides / timelines / mind maps / slide decks from any folder.

Inspired by Google NotebookLM but fully local-first and provider-agnostic — bring your own API key, or run embeddings entirely on-device.

> Status: pre-1.0. Public release of a previously private project; expect rough edges. Issues and PRs welcome.

## Features

- **Grounded RAG chat** with inline `[N]` citations that jump straight to the source chunk in your vault.
- **Hybrid retrieval** — BM25 (Chinese-tokenized via `segmentWords`) + vector embeddings + document-level summary search, merged with Reciprocal Rank Fusion.
- **Multi-format indexing** — Markdown, PDF (text layer), DOCX, XLSX, PPTX. Extraction runs in a Web Worker so the UI stays responsive on large vaults.
- **Image understanding** — Tesseract OCR (`chi_sim` + `eng` by default) and optional LLM vision for image-bearing notes.
- **Structured artifact generation** — summary, study guide, timeline, FAQ, briefing, mind map, and slide deck (PowerPoint export).
- **Provider-agnostic** — any OpenAI-compatible endpoint (DeepSeek, Moonshot/Kimi, GLM, Qwen, OpenAI, …). Local embeddings via `@xenova/transformers` (multilingual-e5-small by default).
- **Save chat answers as notes** — one click turns a cited answer into a permanent vault note.
- **Internationalization** — Chinese (zh-CN) and English (en) UI; auto-detects Obsidian's language.

## Screenshots

_Screenshots coming with v0.4.0._

## Installation

### From Obsidian Community Plugins

_Passed Obsidian's automated review; the listing is live at [community.obsidian.md](https://community.obsidian.md/plugins/notebook-ai). In-app availability follows once the community registry syncs._ Once it appears in the in-app browser:

1. Open Obsidian → Settings → Community plugins → Browse.
2. Search "Knowledge AI" and install.
3. Enable the plugin.

### Manual install (current method)

1. Download `main.js`, `manifest.json`, `styles.css`, and the two `ort-wasm*.wasm` files from the latest [release](https://github.com/david46liu/obsidian-knowledge-ai/releases).
2. Copy them to `<your-vault>/.obsidian/plugins/notebook-ai/`.
3. Reload Obsidian and enable Knowledge AI under Community plugins.

> The plugin is desktop-only (mobile is out of scope due to embedding worker + WASM requirements).

## Quick start

1. **Add a provider** — Settings tab → Providers → Add. Paste your OpenAI-compatible base URL and API key. For DeepSeek use `https://api.deepseek.com/v1`; for Moonshot use `https://api.moonshot.cn/v1`; for OpenAI use `https://api.openai.com/v1`.
2. **Assign tasks** — Settings → Task assignment. Pick a chat model, optionally an embedding model, optionally a summary model. Tasks default to your first provider's default model.
3. **(Optional) Enable vector retrieval** — Settings → Vector retrieval. Toggle on; pick local (default: multilingual-e5-small, downloaded on first use, ~110 MB) or external API embeddings.
4. **Create a Notebook** — Settings → Notebooks → Add, pick a folder. Or right-click any folder in the file explorer → "Create Notebook from this folder".
5. **Chat** — Click the ribbon icon (or open command palette → "Open Notebook AI chat"). Ask questions, follow citations to the source.
6. **Generate artifacts** — Switch to the "Artifacts" tab in the chat view; pick a kind (summary / study guide / timeline / …) → it streams.

See [docs/USAGE.md](./docs/USAGE.md) for advanced topics (query expansion, summary backfill, OCR, custom system prompts, multi-folder notebooks).

## Configuration tips

- **Large vaults (> 2 000 documents)** — turn on document-level summaries: open a notebook card → click "Backfill summaries". This precomputes a per-document summary so the retriever can pick the right document first, then dig into chunks. Default concurrency is 1 (kind to rate-limited Chinese providers); raise to 3-4 for DeepSeek/OpenAI.
- **Chinese content** — leave the default tokenizer alone; it falls back to character n-grams for CJK so BM25 still works without a Chinese segmenter dependency.
- **Cost control** — vector retrieval is local by default. The only API calls are: chat completions, optional reranking, optional document-summary generation, and optional API embeddings.

## 国内网络与本地模型（42model）接入

> 本 fork 针对国内网络环境做了兼容性优化，重点解决「海外源下载慢/失败导致启动或使用报错」。

### 国内安装依赖与构建（避免 GitHub 超时）

国内 `npm install` 会因 `sharp` 从 GitHub 下载二进制而超时失败。使用国内镜像即可：

```bash
cd notebook-ai
npm install --registry=https://registry.npmmirror.com \
  --sharp_libvips_binary_host=https://registry.npmmirror.com/-/binary/sharp-libvips \
  --sharp_binary_host=https://registry.npmmirror.com/-/binary/sharp
npm run build
```

### 接入 42model（活水模型，本地 OpenAI 兼容端点）

42model 在 `http://127.0.0.1:11520` 暴露 OpenAI 兼容 API，需要一个 API key（在 42model 应用「接入」页获取）。

1. **添加 Provider**：设置 → Providers → Add：
   - Base URL：`http://127.0.0.1:11520`
   - API key：从 42model「接入」页复制
   - 默认模型：`qwen3.5:2b-q8_0`（对话）
   - 勾选 **Supports embeddings**
2. **任务分配**：设置 → Task assignment，把 Chat 任务指向 42model provider。
3. **向量检索**：设置 → Vector retrieval → 开启 → Embedding 来源选 **外部 API**，选 42model provider，模型填 `qwen3-embedding:0.6b-q8_0`。这样绕开 HuggingFace 下载，中文效果更好。

### 中文检索建议

- **本地模型**：预设已新增 `Xenova/bge-small-zh-v1.5`（中文优先），比默认 multilingual-e5-small 中文更准。
- **外部 API**：42model 的 Qwen3-Embedding、或国产 bge-m3 系接口中文最佳。
- 中文 BM25 分词已内建，无需额外配置。

> 本地模型默认从 HuggingFace 下载（约 110MB）。若下载失败，界面会提示改用「外部 API」来源，避免海外源不可达导致报错。

## Privacy and data

- No telemetry. The plugin never phones home.
- All indexes and chunks live under `<vault>/.obsidian/plugins/knowledge-ai/data/` (JSONL files; compacted on plugin reload).
- API requests are sent directly to the provider you configure. Your API key is stored in the same plugin folder.

## Development

```bash
npm install
npm run dev        # incremental build (watch)
npm run build      # production build
npm test           # vitest
npm run type-check # tsc --noEmit
```

To test the build against a real vault:

```bash
node scripts/deploy.mjs <path-to-vault>
```

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the contribution workflow.

## Acknowledgements

- [Obsidian](https://obsidian.md/) — the platform.
- [`@xenova/transformers`](https://github.com/xenova/transformers.js) and [ONNX Runtime Web](https://github.com/microsoft/onnxruntime) — on-device embeddings.
- [`pdfjs-dist`](https://github.com/mozilla/pdf.js), [`mammoth`](https://github.com/mwilliamson/mammoth.js), [`xlsx`](https://github.com/SheetJS/sheetjs), [`@xmldom/xmldom`](https://github.com/xmldom/xmldom) — file extraction.
- [`MiniSearch`](https://github.com/lucaong/minisearch) — BM25 retrieval.
- [`Tesseract.js`](https://github.com/naptha/tesseract.js) — OCR.
- [Google NotebookLM](https://notebooklm.google/) — UX inspiration.

## License

[MIT](./LICENSE) — see file for full text.
