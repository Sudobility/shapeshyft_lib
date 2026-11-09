# @sudobility/shapeshyft_lib

Business logic library for ShapeShyft: Zustand stores, manager hooks that wrap
`@sudobility/shapeshyft_client` with client-side caching, pre-built endpoint
templates, and lightweight schema-validation utilities. Ships as ESM only.

## Installation

Published to **GitHub Packages** (not the public npm registry). Consuming
projects need the `@sudobility` scope pointed at GitHub Packages, e.g. in
`.npmrc`:

```
@sudobility:registry=https://npm.pkg.github.com
```

with an auth token that has `read:packages`. Then:

```bash
bun add @sudobility/shapeshyft_lib
```

### Peer dependencies

Install these in the consuming app (they are not bundled):

- `react` >= 18
- `@tanstack/react-query` >= 5
- `zustand` >= 5
- `@sudobility/shapeshyft_client`, `@sudobility/shapeshyft_types`, `@sudobility/types`

## Usage

```typescript
import { useKeysStore, applyTemplate, textClassifierTemplate } from '@sudobility/shapeshyft_lib';

// Reactive store read
const keys = useKeysStore(state => state.getKeys(entitySlug));

// Apply a template -> project + endpoint create payloads.
// 3rd arg is a binding: an LLM key id string, or an EndpointBindingFields object.
const { project, endpoints } = applyTemplate(textClassifierTemplate, 'my-classifier', 'key-123');
```

## Stores

Zustand stores providing client-side caching, scoped by key:

| Store | Scope key | Persisted |
|-------|-----------|-----------|
| `useKeysStore` | entitySlug | in-memory |
| `useProjectsStore` | entitySlug | in-memory |
| `useEndpointsStore` | `entitySlug:projectId` | in-memory |
| `useSettingsStore` | userId | in-memory |
| `useAnalyticsStore` | entitySlug | in-memory |
| `useProviderModelsStore` | LlmProvider | localStorage (1h TTL) |

Manager hooks (`useKeysManager`, `useProjectsManager`, `useEndpointsManager`,
`useSettingsManager`, `useAnalyticsManager`, `useProviderModelsManager`) wrap the
matching `shapeshyft_client` hook and keep its result in the store, preferring
fresh server data and falling back to cache. Other hooks: `useEndpointTester`,
`useBudgetTracker`, `useProjectTemplates`, `useEndpointTemplates`.

## Endpoint Templates

9 pre-built project templates: Text Classifier, Sentiment Analyzer, Data
Extractor, Content Generator, Localization, Image Recognition, Image Generation,
Image Processing, Audio Transcription. Access all via `ALL_TEMPLATES`.

## Development

```bash
bun install          # Install deps (requires GitHub Packages auth for @sudobility/*)
bun run build        # Compile to dist/ (tsc) + rewrite ESM import specifiers
bun run build:watch  # Watch-mode build
bun run clean        # Remove dist/
bun run typecheck    # tsc --noEmit
bun run lint         # ESLint (lint:fix to autofix)
bun run format       # Prettier write (format:check to verify)
bun run test:unit    # Run Vitest once
bun run test:watch   # Vitest watch mode
bun run verify       # typecheck + lint + test:unit + build
```

## Related Packages

- `@sudobility/shapeshyft_client` — API hooks (peer dependency this library wraps)
- `@sudobility/shapeshyft_types` — type definitions
- `shapeshyft_app` — downstream frontend consumer

## License

BUSL-1.1
