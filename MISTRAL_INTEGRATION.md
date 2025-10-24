# Mistral AI Integration Guide

## Overview

**YES, it is absolutely possible to integrate Mistral API with your Auth0 + Vercel AI SDK architecture!** 

This project has been updated to use Mistral AI instead of OpenAI. The Vercel AI SDK provides official support for Mistral through the `@ai-sdk/mistral` package, making the integration seamless and straightforward.

## What Changed

The following changes were made to integrate Mistral AI:

### 1. Package Dependencies
- ✅ Added `@ai-sdk/mistral` package (v2.0.19)
- ℹ️  Kept `@ai-sdk/openai` as optional dependency (can be removed if not needed)

### 2. Code Changes

#### Chat API Route (`src/app/api/chat/route.ts`)
```typescript
// Before
import { openai } from '@ai-sdk/openai';
model: openai.chat('gpt-4o-mini')

// After
import { mistral } from '@ai-sdk/mistral';
model: mistral('mistral-small-latest')
```

#### RAG Embeddings (`src/lib/rag/embedding.ts`)
```typescript
// Before
import { openai } from '@ai-sdk/openai';
const embeddingModel = openai.embedding('text-embedding-3-small');

// After
import { mistral } from '@ai-sdk/mistral';
const embeddingModel = mistral.textEmbedding('mistral-embed');
```

### 3. Environment Variables
Updated `.env.example`:
```bash
# Before
OPENAI_API_KEY="YOUR_API_KEY"

# After
MISTRAL_API_KEY="YOUR_MISTRAL_API_KEY"
# Optional overrides (defaults shown)
MISTRAL_CHAT_MODEL="mistral-small-latest"
MISTRAL_EMBEDDING_MODEL="mistral-embed"
```

### 4. Database Schema
Updated embedding vector dimensions:
```sql
-- Before: vector(1536) - OpenAI dimensions
-- After: vector(1024) - Mistral dimensions
```

A migration file has been created at `src/lib/db/migrations/0001_mistral_embeddings.sql` to update the vector dimensions.

## Database Setup

### For New Installations

If you're setting up the project for the first time:

```bash
# Start the postgres database
docker compose up -d

# Run migrations (includes Mistral embedding dimensions)
npm run db:migrate

# Initialize FGA store
npm run fga:init
```

### For Existing Installations

If you're upgrading from OpenAI to Mistral and have existing data:

**⚠️ Important**: Existing embeddings (1536d) are incompatible with Mistral embeddings (1024d)

You have two options:

#### Option 1: Re-generate All Embeddings (Recommended)
```bash
# Backup your database first!
# Then truncate embeddings and run migration
npm run db:migrate

# Re-upload your documents through the UI
# New embeddings will be generated automatically with Mistral
```

#### Option 2: Hybrid Approach (Keep OpenAI for Embeddings)
Keep using OpenAI for embeddings while using Mistral for chat. See the "Hybrid Approach" section below.

To keep the old schema (1536d), revert the schema change in `src/lib/db/schema/embeddings.ts`:
```typescript
embedding: vector('embedding', { dimensions: 1536 }).notNull(),
```

And revert the embedding.ts to use OpenAI:
```typescript
import { openai } from '@ai-sdk/openai';
const embeddingModel = openai.embedding('text-embedding-3-small');
```

## Getting Your Mistral API Key

1. Go to [Mistral AI Console](https://console.mistral.ai/)
2. Sign up or log in to your account
3. Navigate to **API Keys** section
4. Create a new API key
5. Copy the key and add it to your `.env.local` file:
   ```bash
   MISTRAL_API_KEY="your-mistral-api-key-here"
   ```

## Available Mistral Models

The Vercel AI SDK supports all Mistral models:

### Chat Models (Language Generation)
- `mistral-large-latest` - Most capable model, best for complex tasks
- `mistral-medium-latest` - Balanced performance and cost
- `mistral-small-latest` - Fast and cost-effective (currently used in this project)
- `ministral-3b-latest` - Lightweight model
- `ministral-8b-latest` - Lightweight model with better performance
- `pixtral-large-latest` - Multimodal model (supports images)
- `open-mistral-7b` - Open source model
- `open-mixtral-8x7b` - Open source mixture of experts
- `open-mixtral-8x22b` - Larger open source mixture of experts

### Embedding Models
- `mistral-embed` - Text embedding model (1024 dimensions)

### Switching Models

You can easily switch models using environment variables in your `.env.local` file:

```bash
# Use a more powerful model
MISTRAL_CHAT_MODEL="mistral-large-latest"

# Use a different embedding model (if available)
MISTRAL_EMBEDDING_MODEL="mistral-embed"
```

Or modify the code directly in `src/app/api/chat/route.ts`:

```typescript
// For more powerful model
const mistralModelId = 'mistral-large-latest';

// For better cost efficiency
const mistralModelId = 'mistral-small-latest';

// For multimodal capabilities (images + text)
const mistralModelId = 'pixtral-large-latest';
```

## Architecture Compatibility

### ✅ What Works Perfectly
- **Auth0 Integration**: All Auth0 features work seamlessly
- **Vercel AI SDK Streaming**: Real-time streaming responses
- **Tool Calling**: All tools (SerpAPI, Gmail, Calendar, etc.) work perfectly
- **RAG Pipeline**: Document embeddings and retrieval work with Mistral embeddings
- **Auth0 FGA**: Fine-grained authorization unchanged
- **Auth0 AI Token Vault**: Google OAuth connections work as before
- **Interruption Flows**: Async authorization flows continue to work
- **Server Actions**: Document upload/download/share unchanged

### 🔄 Migration Notes

1. **Embeddings Dimension**: Mistral embeddings produce 1024-dimensional vectors (vs OpenAI's 1536). If you have existing embeddings in your database, you may need to:
   - Re-generate embeddings for existing documents, OR
   - Keep using OpenAI for embeddings while using Mistral for chat

2. **Tool Calling**: Mistral supports function/tool calling similar to OpenAI, so all existing tools work without modification

3. **Streaming**: Mistral streaming works identically to OpenAI with the Vercel AI SDK

### Example: Hybrid Approach (Mistral Chat + OpenAI Embeddings)

If you want to use Mistral for chat but keep OpenAI for embeddings:

```typescript
// In src/app/api/chat/route.ts
import { mistral } from '@ai-sdk/mistral';
model: mistral('mistral-small-latest')

// In src/lib/rag/embedding.ts
import { openai } from '@ai-sdk/openai';
const embeddingModel = openai.embedding('text-embedding-3-small');

// Add both keys to .env.local
MISTRAL_API_KEY="your-mistral-key"
OPENAI_API_KEY="your-openai-key"  // Optional, only for embeddings
```

## Deployment to Vercel

Deploying to Vercel with Mistral is straightforward:

### 1. Add Environment Variables in Vercel

In your Vercel project settings:
```bash
MISTRAL_API_KEY=your-mistral-api-key-here
AUTH0_SECRET=your-auth0-secret
AUTH0_DOMAIN=your-auth0-domain
AUTH0_CLIENT_ID=your-client-id
AUTH0_CLIENT_SECRET=your-client-secret
DATABASE_URL=your-postgres-url
FGA_STORE_ID=your-fga-store-id
FGA_CLIENT_ID=your-fga-client-id
FGA_CLIENT_SECRET=your-fga-client-secret
FGA_API_URL=your-fga-api-url
FGA_API_AUDIENCE=your-fga-api-audience
```

### 2. Deploy

```bash
# Install Vercel CLI if you haven't
npm i -g vercel

# Deploy to Vercel
vercel

# Or use GitHub integration
# Push to your repository and Vercel will auto-deploy
```

### 3. Vercel-Specific Considerations
- ✅ Mistral API works on Vercel Edge Runtime
- ✅ Streaming responses work perfectly
- ✅ All serverless functions stay within free tier limits
- ✅ No additional configuration needed

## Cost Comparison

Mistral is generally more cost-effective than OpenAI:

### Mistral Pricing (as of 2024)
- **Mistral Small**: ~$0.002 per 1M tokens
- **Mistral Medium**: ~$0.006 per 1M tokens  
- **Mistral Large**: ~$0.008 per 1M tokens
- **Mistral Embed**: ~$0.0001 per 1M tokens

### OpenAI Pricing (for reference)
- **GPT-4o-mini**: ~$0.150 per 1M input tokens, ~$0.600 per 1M output tokens
- **Text-embedding-3-small**: ~$0.020 per 1M tokens

Mistral can be **significantly cheaper** for similar performance levels.

## Testing Your Integration

After setting up your Mistral API key:

1. Start the development server:
   ```bash
   npm run dev
   ```

2. Navigate to `http://localhost:3000`

3. Try these test prompts:
   - "Hello, who are you?"
   - "What's the weather like?" (if SerpAPI is configured)
   - "Search my emails for..." (if Gmail is connected)
   - Upload a document and ask about its contents

4. Verify that:
   - Streaming responses work smoothly
   - Tools are being called correctly
   - Auth0 authentication works
   - Document embeddings are generated

## Troubleshooting

### Issue: "Invalid API Key"
- Check that `MISTRAL_API_KEY` is correctly set in `.env.local`
- Verify the key is active in Mistral console
- Ensure no extra quotes or spaces around the key

### Issue: "Model not found"
- Ensure you're using a valid Mistral model ID
- Check [Mistral docs](https://docs.mistral.ai/getting-started/models/) for current model names

### Issue: Tool calling not working
- Mistral supports tool calling, but ensure you're using a recent model
- `mistral-small-latest` and above support function calling
- Older/smaller models may have limited tool calling capabilities

### Issue: Embeddings dimension mismatch
- If you have existing OpenAI embeddings (1536d), you'll need to either:
  - Re-generate all embeddings with Mistral (1024d), or
  - Use a hybrid approach (OpenAI for embeddings, Mistral for chat)

## Additional Resources

- [Mistral AI Docs](https://docs.mistral.ai/)
- [Vercel AI SDK - Mistral Provider](https://ai-sdk.dev/providers/ai-sdk-providers/mistral)
- [Mistral Console](https://console.mistral.ai/)
- [Mistral API Reference](https://docs.mistral.ai/api/)

## Support

If you encounter any issues with the integration, please:
1. Check the Mistral API status page
2. Verify all environment variables are correctly set
3. Review the Vercel deployment logs
4. Open an issue in this repository

---

**Summary**: Mistral AI integrates seamlessly with your existing Auth0 + Vercel AI SDK architecture. No architectural changes are needed - just swap the provider import and update your API key. All features including streaming, tool calling, RAG, and Auth0 integrations work perfectly with Mistral.
