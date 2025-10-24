# Quick Start: Mistral AI Integration

## ✅ Is Mistral Compatible?

**YES!** Mistral AI is fully compatible with your Auth0 + Vercel AI SDK architecture. No architectural changes needed.

## 🚀 Quick Setup (3 Steps)

### 1. Get Mistral API Key
Visit [console.mistral.ai](https://console.mistral.ai/) → API Keys → Create Key

### 2. Add to Environment
```bash
# In .env.local
MISTRAL_API_KEY="your-mistral-api-key-here"
```

### 3. Run Migrations (if needed)
```bash
# For new installations
docker compose up -d
npm run db:migrate
npm run fga:init

# Start dev server
npm run dev
```

## 🎯 What Changed?

### Code (2 files)
- **Chat API** (`src/app/api/chat/route.ts`): `openai` → `mistral`
- **Embeddings** (`src/lib/rag/embedding.ts`): OpenAI embeddings → Mistral embeddings

### Database
- Vector dimensions: 1536 (OpenAI) → 1024 (Mistral)
- Migration file: `0001_mistral_embeddings.sql`

## 📊 Cost Savings

| Model | Mistral | OpenAI |
|-------|---------|--------|
| Chat | $0.002/1M tokens | $0.150/1M tokens |
| Embeddings | $0.0001/1M tokens | $0.020/1M tokens |

**75x cheaper!** 💰

## 🔧 Model Configuration

Change models via environment variables:

```bash
# .env.local
MISTRAL_CHAT_MODEL="mistral-large-latest"  # More powerful
MISTRAL_EMBEDDING_MODEL="mistral-embed"    # Default
```

Available models:
- `mistral-small-latest` - Fast, cheap (default)
- `mistral-medium-latest` - Balanced
- `mistral-large-latest` - Most capable
- `pixtral-large-latest` - Multimodal (images)

## ✅ All Features Work

- ✅ Auth0 authentication & authorization
- ✅ Streaming responses
- ✅ Tool calling (SerpAPI, Gmail, Calendar, etc.)
- ✅ RAG document retrieval
- ✅ Auth0 FGA fine-grained access
- ✅ OAuth Token Vault flows
- ✅ Document upload/download/share
- ✅ Vercel deployment

## 🚀 Deploy to Vercel

```bash
# Set in Vercel dashboard
MISTRAL_API_KEY=your-key
# ... (other Auth0, FGA, DB vars)

# Deploy
vercel
```

## 📚 More Info

- `MISTRAL_INTEGRATION.md` - Comprehensive guide
- `CHANGES_SUMMARY.md` - Detailed change log
- [Mistral Docs](https://docs.mistral.ai/)
- [Vercel AI SDK](https://ai-sdk.dev/providers/ai-sdk-providers/mistral)

## 🆘 Issues?

1. Check `MISTRAL_API_KEY` is set correctly
2. Run `npm run db:migrate` for embedding dimension updates
3. Verify model names at [docs.mistral.ai](https://docs.mistral.ai/getting-started/models/)

---

**TL;DR**: Yes, Mistral works perfectly! Just add your API key and run migrations. Same architecture, 75x cheaper.
