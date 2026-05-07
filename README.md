# NEXUS
👾Both for Mobile and Dekstop

A powerful, single-file RAG (Retrieval-Augmented Generation) chat interface with local AI model support, built-in knowledge base indexing, and conversation memory.

## Features(The most easiest way to have a minimal interface for your ollama AI Models)

✨ **RAG Knowledge Base** — Upload and index text/code files; AI automatically retrieves relevant context when answering questions  
🤖 **Local AI Models** — Connect to local LLMs (Ollama, LM Studio, etc.) or cloud APIs  
💾 **Persistent Chat History** — Conversations saved locally with search & rename  
🧠 **Conversation Memory** — Store context that carries across chats  
🌐 **Web Search** — Optional live web results integration  
📄 **Multi-Format Support** — Index TXT, MD, JS, PY, JSON, CSV, and more  
⚡ **Fast & Responsive** — Dark mode interface optimized for modern browsers  
📱 **Mobile-Friendly** — Touch-optimized design  

## Getting Started

Simply open `index.html` in your browser. No build process, no dependencies needed.

```bash
# Clone or download this repo
git clone https://github.com/user-sb737/nexus.git
cd nexus-v8

# Open in browser
open index-1.html
```

## Configuration

### Using Local Models

1. Install [Ollama](https://ollama.ai) or [LM Studio](https://lmstudio.ai)
2. Run a model locally (e.g., `ollama run mistral`)
3. In NEXUS, add the model with endpoint: `http://localhost:11434/api/chat` (Ollama) or `http://localhost:1234/v1/chat/completions` (LM Studio)

### Using Cloud APIs

Add your API key in the Model Settings section and configure the endpoint URL.

## How to Use

1. **Add a Model** — Configure local or cloud AI endpoint in the sidebar
2. **Upload Knowledge Files** — Drop files in the RAG section to build your knowledge base
3. **Ask Questions** — The AI will automatically retrieve relevant context from your files
4. **Manage Conversations** — View history, rename chats, or start fresh

## Storage

All data is stored **locally in your browser** using localStorage:
- Chat history
- Conversation memory
- Settings
- Indexed documents

Clear browser data to reset everything.

## API Support

Currently supports:
- OpenAI-compatible endpoints (Ollama, LM Studio, OpenAI API, etc.)
- Custom endpoints with OpenAI `/v1/chat/completions` interface

## Privacy

✅ Your data stays on your device  
✅ No telemetry or tracking  
✅ Open source — inspect the code  

## Development

To modify the code, edit `index.html` directly — it's a single, self-contained file.

## License

MIT — Feel free to use, modify, and distribute.

## Contributing

Issues, suggestions, and PRs welcome!

---

Built with ❤️ as a minimal, powerful chat interface.
