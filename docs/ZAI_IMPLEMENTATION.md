# Z.AI (GLM) Implementation Guide

## Overview

This document explains how Z.AI (Zhipu AI) was integrated into thepopebot as a first-class LLM provider. Use this guide if you want to implement similar providers or understand the architecture.

## What is Z.AI?

Z.AI (Zhipu AI) is a Chinese AI company providing the GLM (General Language Model) family. Their API is **OpenAI-compatible**, making integration straightforward.

### Key Features
- **OpenAI-compatible API** - Uses the same `/chat/completions` endpoint format
- **Excellent reasoning and coding capabilities**
- **Multiple model variants** - From flagship to fast/lightweight
- **Competitive pricing**

## Implementation Details

### 1. Model Configuration (`lib/ai/model.js`)

Added a new `zai` case in the `createModel()` switch statement:

```javascript
case 'zai': {
  const { ChatOpenAI } = await import('@langchain/openai');
  const apiKey = process.env.ZAI_API_KEY;
  if (!apiKey) {
    throw new Error('ZAI_API_KEY environment variable is required for Z.AI provider');
  }
  return new ChatOpenAI({
    modelName,
    maxTokens,
    apiKey,
    configuration: {
      baseURL: 'https://api.z.ai/v1',
    },
  });
}
```

**Why ChatOpenAI?** Since Z.AI is OpenAI-compatible, we reuse LangChain's `ChatOpenAI` class with a custom `baseURL`.

### 2. Default Model Mapping

Added to `DEFAULT_MODELS`:

```javascript
const DEFAULT_MODELS = {
  anthropic: 'claude-sonnet-4-20250514',
  openai: 'gpt-4o',
  google: 'gemini-2.5-pro',
  zai: 'glm-4.7',  // <-- Added
};
```

### 3. Provider Registry (`setup/lib/providers.mjs`)

Added Z.AI to the `PROVIDERS` export:

```javascript
zai: {
  label: 'GLM (Z.AI / Zhipu)',
  name: 'Z.AI',
  envKey: 'ZAI_API_KEY',
  keyPrefix: '',
  keyPage: 'https://z.ai/model-api',
  builtin: false,
  baseUrl: 'https://api.z.ai/v1',
  api: 'openai-completions',
  models: [
    { id: 'glm-4.7', name: 'GLM-4.7 (Flagship)', default: true },
    { id: 'glm-4.6', name: 'GLM-4.6' },
    { id: 'glm-4.5', name: 'GLM-4.5' },
    { id: 'glm-4.5-air', name: 'GLM-4.5 Air (Fast)' },
  ],
},
```

### 4. Environment Variables

| Variable | Description |
|----------|-------------|
| `LLM_PROVIDER=zai` | Selects Z.AI as the provider |
| `LLM_MODEL=glm-4.7` | (Optional) Model name, defaults to `glm-4.7` |
| `ZAI_API_KEY` | Your Z.AI API key |

## How to Add a New OpenAI-Compatible Provider

Use this checklist to add any OpenAI-compatible provider:

### Step 1: Add to `lib/ai/model.js`

1. Add default model to `DEFAULT_MODELS`
2. Add a new case in `createModel()` switch:
   ```javascript
   case 'your-provider': {
     const { ChatOpenAI } = await import('@langchain/openai');
     const apiKey = process.env.YOUR_PROVIDER_API_KEY;
     if (!apiKey) {
       throw new Error('YOUR_PROVIDER_API_KEY environment variable is required');
     }
     return new ChatOpenAI({
       modelName,
       maxTokens,
       apiKey,
       configuration: {
         baseURL: 'https://api.your-provider.com/v1',
       },
     });
   }
   ```

### Step 2: Add to `setup/lib/providers.mjs`

```javascript
your-provider: {
  label: 'Your Provider Name',
  name: 'YourProvider',
  envKey: 'YOUR_PROVIDER_API_KEY',
  keyPrefix: '',
  keyPage: 'https://your-provider.com/keys',
  builtin: false,
  baseUrl: 'https://api.your-provider.com/v1',
  api: 'openai-completions',
  models: [
    { id: 'model-1', name: 'Model 1', default: true },
    { id: 'model-2', name: 'Model 2' },
  ],
},
```

### Step 3: Update Documentation

Add your provider to `docs/RUNNING_DIFFERENT_MODELS.md`:
1. Add to the environment variables table
2. Add to the providers table
3. Add a usage section if needed

### Step 4: Test

```bash
# Event Handler test
LLM_PROVIDER=your-provider \
YOUR_PROVIDER_API_KEY=your-key \
node -e "import('./lib/ai/model.js').then(m => m.createModel().then(r => console.log('Success:', r)))"

# Job test (GitHub Actions)
npx thepopebot set-var LLM_PROVIDER your-provider
npx thepopebot set-var LLM_MODEL model-1
npx thepopebot set-agent-secret YOUR_PROVIDER_API_KEY your-key
```

## API Endpoint Details

### Z.AI API
- **Base URL**: `https://api.z.ai/v1`
- **Chat Endpoint**: `/chat/completions`
- **Auth Header**: `Authorization: Bearer YOUR_API_KEY`
- **Format**: OpenAI-compatible JSON

### Example Request

```bash
curl https://api.z.ai/v1/chat/completions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "glm-4.7",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## Available Models

| Model ID | Description | Best For |
|----------|-------------|----------|
| `glm-4.7` | Flagship model | Complex reasoning, coding, agents |
| `glm-4.6` | Balanced | General purpose |
| `glm-4.5` | Standard | Everyday tasks |
| `glm-4.5-air` | Fast & lightweight | Quick responses, high throughput |

## Getting an API Key

1. Visit [https://z.ai/model-api](https://z.ai/model-api)
2. Sign up or log in
3. Generate an API key from the dashboard
4. Add to your `.env` file or GitHub secrets

## Alternative: Using Custom Provider

If you don't want to modify the code, you can use the `custom` provider:

```bash
# Event Handler
LLM_PROVIDER=custom
LLM_MODEL=glm-4.7
OPENAI_BASE_URL=https://api.z.ai/v1
CUSTOM_API_KEY=your-zai-api-key
```

```bash
# Jobs
npx thepopebot set-var LLM_PROVIDER custom
npx thepopebot set-var LLM_MODEL glm-4.7
npx thepopebot set-var OPENAI_BASE_URL https://api.z.ai/v1
npx thepopebot set-agent-secret CUSTOM_API_KEY your-zai-api-key
```

This works immediately without code changes but won't appear in the setup wizard.

## Summary

Z.AI integration required:
1. **~15 lines** in `lib/ai/model.js` (provider case)
2. **~12 lines** in `setup/lib/providers.mjs` (registry entry)
3. **Documentation updates** in `docs/RUNNING_DIFFERENT_MODELS.md`

The key insight is that any OpenAI-compatible API can be integrated with minimal code by reusing `ChatOpenAI` with a custom `baseURL`.
