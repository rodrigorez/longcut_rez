# LongCut Original — Arquitetura do Projeto Fonte

> **Referência:** [SamuelZ12/longcut](https://github.com/SamuelZ12/longcut) (AGPL-3.0)
> **Commit original:** `c80c874` — 483 commits no branch `main`
> **Cópia local:** `.orig/` (253 arquivos, cópia fiel do fork)

Este documento é uma engenharia reversa completa do código original do LongCut,
criada para orientar a implementação de novas features no fork `rodrigorez/longcut_rez`.

Nenhum dos padrões descritos aqui deve ser alterado — apenas estendido com
arquivos novos ou modificações mínimas nos existentes (conforme especificado em `SPEC.md`).

---

## Sumário

1. [Provider System](#1-provider-system)
2. [API Routes](#2-api-routes)
3. [Component Architecture](#3-component-architecture)
4. [Database Schema](#4-database-schema)
5. [AI Processing Pipeline](#5-ai-processing-pipeline)
6. [Type System](#6-type-system)
7. [Key Patterns & Conventions](#7-key-patterns--conventions)

---

## 1. Provider System

### 1.1 Interface Contract (`lib/ai-providers/types.ts`)

```typescript
type ProviderKey = 'grok' | 'gemini' | 'minimax';

interface ProviderBehavior {
  forceFullTranscriptTopicGeneration: boolean;
  forceSmartModeOnClient: boolean;
}

interface ProviderGenerateParams {
  prompt: string;
  model?: string;
  temperature?: number;
  topP?: number;
  maxOutputTokens?: number;
  timeoutMs?: number;
  zodSchema?: ZodTypeAny;
  schemaName?: string;
  metadata?: Record<string, unknown>;
}

interface ProviderGenerateResult {
  content: string;
  rawResponse?: unknown;
  usage?: { promptTokens?: number; completionTokens?: number; totalTokens?: number; latencyMs?: number };
  provider?: string;
  model?: string;
}

interface ProviderAdapter {
  readonly name: string;
  readonly defaultModel: string;
  generate(params: ProviderGenerateParams): Promise<ProviderGenerateResult>;
}
```

### 1.2 Configuration (`lib/ai-providers/provider-config.ts`)

```typescript
const PROVIDER_ORDER: ProviderKey[] = ['grok', 'gemini', 'minimax'];

const PROVIDER_DEFAULT_MODELS = {
  grok:    'grok-4-1-fast-non-reasoning',
  gemini:  'gemini-2.5-flash-lite',
  minimax: 'MiniMax-M3',
};

const PROVIDER_BEHAVIORS = {
  grok:    { forceFullTranscriptTopicGeneration: true,  forceSmartModeOnClient: true  },
  gemini:  { forceFullTranscriptTopicGeneration: false, forceSmartModeOnClient: false },
  minimax: { forceFullTranscriptTopicGeneration: false, forceSmartModeOnClient: true  },
};

const PROVIDER_ENV_KEYS = {
  grok:    'XAI_API_KEY',
  gemini:  'GEMINI_API_KEY',
  minimax: 'MINIMAX_API_KEY',
};
```

**Helper functions** (10 exported):
| Function | Returns | Purpose |
|---|---|---|
| `normalizeProviderKey(value?)` | `ProviderKey \| undefined` | Case-insensitive normalization |
| `getConfiguredProviderKey(preferred?)` | `ProviderKey \| undefined` | Checks `preferred` → `AI_PROVIDER` → `NEXT_PUBLIC_AI_PROVIDER` |
| `getEffectiveProviderKey(preferred?)` | `ProviderKey` | Fallback chain; ultimate fallback `'grok'` |
| `getProviderDefaultModel(key)` | `string` | Returns model from `PROVIDER_DEFAULT_MODELS` |
| `getProviderModelDefaults(preferred?)` | `{ defaultModel, fastModel, proModel }` | Respects `AI_DEFAULT_MODEL`, `AI_FAST_MODEL`, `AI_PRO_MODEL` env overrides |
| `getProviderBehavior(key)` | `ProviderBehavior` | Returns behavior flags |
| `getProviderPriorityOrder()` | `ProviderKey[]` | Copy of `PROVIDER_ORDER` |
| `getProviderFallbackOrder(currentKey, availableKeys?)` | `ProviderKey[]` | Fallback providers excluding current |

### 1.3 Registry (`lib/ai-providers/registry.ts`)

**Factory pattern with singleton cache:**
```typescript
const providerFactories: Record<ProviderKey, ProviderFactory> = {
  grok:    createGrokAdapter,
  gemini:  createGeminiAdapter,
  minimax: createMiniMaxAdapter,
};

const providerEnvGuards: Record<ProviderKey, () => string | undefined> = {
  grok:    () => process.env.XAI_API_KEY,
  gemini:  () => process.env.GEMINI_API_KEY,
  minimax: () => process.env.MINIMAX_API_KEY,
};

const providerCache: Partial<Record<ProviderKey, ProviderAdapter>> = {};
```

**Resolution flow:**
1. `getProviderKey()` → `resolveProviderKey()`:
   - Check env vars (`AI_PROVIDER` / `NEXT_PUBLIC_AI_PROVIDER`)
   - If none set, iterate `PROVIDER_ORDER` and use first whose env key exists
   - Fallback: `'grok'`
2. `getProvider(key?)` → `resolveProviderKey()` → `ensureProvider()` → cache

**Fallback in `generateStructuredContent()`:**
- Tries primary provider
- On retryable error (429, 5xx, timeout, overload) AND no explicit provider → try fallback
- If fallback also fails → throw original error
- `isRetryableError()`: checks message for rate limit, 429, 5xx, timeout, overload

### 1.4 Adapter Pattern (reference: Grok)

```typescript
export function createGrokAdapter(): ProviderAdapter {
  const apiKey = process.env.XAI_API_KEY;
  if (!apiKey) throw new Error('XAI_API_KEY not configured');
  return {
    name: 'grok',
    defaultModel: 'grok-4-1-fast-non-reasoning',
    async generate(params) {
      const requestStartedAt = Date.now();
      const controller = buildAbortController(params.timeoutMs);
      try {
        const payload = buildPayload(params, apiKey);
        const response = await fetch('https://api.x.ai/v1/chat/completions', payload);
        // parse, handle errors, normalize usage
        return { content, rawResponse, usage, provider: 'grok', model };
      } finally { controller?.abort(); }
    },
  };
}
```

**Structured output (Grok/Gemini):** Native JSON schema via `response_format: { type: 'json_schema', json_schema: {...} }`

**Structured output (MiniMax):** Prompt-based + Zod validation client-side (no native support)

**Gemini unique patterns:**
- Internal model cascade: `['gemini-2.5-flash-lite', 'gemini-3-flash', 'gemini-3-pro']`
- Timeout via `Promise.race` (not AbortController)
- Custom `convertToGeminiSchema()` for SchemaType enum
- Error classification into 5 categories

**MiniMax unique patterns:**
- `buildPrompt()` appends JSON schema to prompt text
- `normalizeStructuredContent()`: JSON.parse + Zod validation
- `stripReasoningBlocks()` removes `<think>`/`<thinking>` tags
- `reasoning_split: true` always in payload

### 1.5 Client Config (`lib/ai-providers/client-config.ts`)

```typescript
function resolveClientProviderKey(): ClientProviderKey {
  return normalizeProviderKey(process.env.NEXT_PUBLIC_AI_PROVIDER) ?? 'grok';
}
// Exports: getClientProviderKey(), shouldForceSmartModeOnClient()
```

### 1.6 AI Client (`lib/ai-client.ts`)

Application-level wrapper:
- `generateAIResponse(prompt, options)` → `Promise<string>`
- `generateAIResult(prompt, options)` → `Promise<ProviderGenerateResult>`
- `listAvailableAIProviders()` → `ProviderKey[]`

---

## 2. API Routes

### 2.1 Router Inventory (27 rotas)

| Category | Routes |
|---|---|
| Core Video | `video-info`, `transcript`, `generate-topics`, `generate-summary`, `quick-preview`, `top-quotes`, `generate-image` |
| AI & Chat | `chat`, `suggested-questions` |
| User Data | `video-analysis`, `update-video-analysis`, `notes/`, `notes/all`, `notes/enhance`, `toggle-favorite`, `link-video`, `verify-video-link` |
| Auth | `auth/signout` |
| Limits | `check-limit`, `check-video-cache`, `image-limit` |
| Security | `csrf-token` |
| Utility | `random-video`, `translate`, `email/send-welcome`, `newsletter`, `stripe/criar-sessao`, `subscription/portal`, `webhooks/stripe` |

### 2.2 Security Middleware (`lib/security-middleware.ts`)

**Universal wrapping pattern:**
```typescript
async function handler(request: NextRequest) { ... }
export const POST = withSecurity(handler, config);  // ou SECURITY_PRESETS.PUBLIC
```

**Middleware execution order:**
1. Allowed Methods check → 405
2. Authentication (`supabase.auth.getUser()`) → 401
3. Rate limiting (`RateLimiter.check()`) → 429
4. Body size (`content-length` header) → 413
5. CSRF validation (`validateCSRF()`) → 403 (POST/PUT/PATCH/DELETE only)
6. Execute inner `handler(req)`
7. Inject security headers:
   - `X-Content-Type-Options: nosniff`
   - `X-Frame-Options: DENY`
   - `X-XSS-Protection: 1; mode=block`
   - `Referrer-Policy: strict-origin-when-cross-origin`
   - `Permissions-Policy: camera=(), microphone=(), geolocation=()`
8. CSRF token injection if missing
9. Catch-all 500 (no detail leak)

**Security presets:**

| Preset | requireAuth | rateLimit | maxBodySize | allowedMethods | csrfProtection |
|---|---|---|---|---|---|
| `PUBLIC` | No | API_GENERAL (60/min) | 1MB | GET, POST | No |
| `AUTHENTICATED` | Yes | AUTH_GENERATION (20/hr) | 5MB | GET, POST, PUT, DELETE | **Yes** |
| `AUTHENTICATED_READ_ONLY` | Yes | READ_ONLY (100/min) | 1MB | GET | No |
| `STRICT` | Yes | 10/min | 512KB | POST | Yes |

### 2.3 CSRF Protection (`lib/csrf-protection.ts`)

**Double Submit Cookie pattern:**
- Token: `crypto.randomBytes(32).toString('hex')` (64 chars)
- Set in httpOnly, SameSite=Strict cookie named `csrf-token`
- Client must echo it in `X-CSRF-Token` header
- Comparison: cookie value === header value

**Endpoint:** `GET /api/csrf-token` (requires auth)
- Reuses existing valid token if present
- Sets both cookie + `X-CSRF-Token` response header

**Client helper:** `csrfFetch(url, options)` automatically includes CSRF token

### 2.4 Rate Limiting

**Preset limits:**

| Limit | Window | Max | Used By |
|---|---|---|---|
| API_GENERAL | 1 min | 60 | PUBLIC preset |
| AUTH_GENERATION | 1 hr | 20 | AUTHENTICATED preset |
| AUTH_CHAT | 1 min | 30 | Chat route (internal) |
| ANON_CHAT | 1 min | 10 | Chat route (internal) |
| AUTH_VIDEO_GENERATION | 24 hr | 5 | — |
| ANON_GENERATION | 24 hr | 1 | — |
| SUGGESTED_QUESTIONS | 1 min | 20 | suggested-questions |
| READ_ONLY | 1 min | 100 | AUTHENTICATED_READ_ONLY |
| VIDEO_GENERATION_FREE_UNREGISTERED | 30 days | 0 | Anon users |
| VIDEO_GENERATION_FREE_REGISTERED | 30 days | 3 | Free users |
| VIDEO_GENERATION_PRO | 30 days | 100 | Pro users |
| AUTH_ATTEMPT | 15 min | 5 | Login |

**Dual rate limiting pattern:** Coarse limit in middleware + fine-grained internal limit in handler (chat, video-analysis).

### 2.5 Route Handler Patterns

**Error handling standard:**
```typescript
async function handler(request: NextRequest) {
  try {
    // business logic, Zod validation
    return NextResponse.json({ ... });
  } catch (error) {
    console.error('[ROUTE_NAME] Error:', error);
    return NextResponse.json({ error: 'User-safe message' }, { status: 500 });
  }
}
```

**Zod validation pattern:**
```typescript
try {
  validatedData = someSchema.parse(body);
} catch (error) {
  if (error instanceof z.ZodError) {
    return NextResponse.json({ error: 'Validation failed', details: formatValidationError(error) }, { status: 400 });
  }
  throw error;
}
```

### 2.6 Key Route Details

#### `POST /api/video-info`
- PUBLIC, no CSRF
- Body: `{ url: string }`
- Response: `{ videoId, title, author, thumbnail, duration, description, tags }`

#### `POST /api/transcript`
- PUBLIC
- Body: `{ url, lang?, expectedDuration? }`
- Dual strategy: YouTube InnerTube API (free) → Supadata (paid fallback)
- Post-processing: `mergeTranscriptSegmentsIntoSentences()`
- Returns: `{ videoId, transcript[], language, availableLanguages[], transcriptDuration, isPartial, coverageRatio }`

#### `POST /api/generate-topics`
- Custom security: `{ maxBodySize: 10MB, allowedMethods: ['POST'] }`
- Rate limiting is internal
- Body: `{ transcript, includeCandidatePool?, excludeTopicKeys?, videoInfo?, mode?, language? }`
- Returns: `{ topics[], topicCandidates? }`

#### `POST /api/video-analysis` (main analysis endpoint)
- PUBLIC
- Orchestrates full flow: cache check → limit check → AI generation → DB save → credit consume
- Guest flow: 1 free analysis via cookies
- Subscription flow: `canGenerateVideo()` → credit consumption via RPC
- Returns: `{ topics[], themes[], cached, topicCandidates?, modelUsed?, videoDbId? }`

#### `POST /api/check-video-cache`
- PUBLIC
- Checks `video_analyses.youtube_id`
- Returns full analysis if cached: `{ cached, videoId, videoDbId, topics, transcript, videoInfo, summary, suggestedQuestions, cacheDate, ownedByCurrentUser }`

#### `POST /api/notes` (GET/POST/DELETE)
- AUTHENTICATED + CSRF
- GET: `?youtubeId=...` or `?videoId=...` → `{ notes: Note[] }`
- POST: `{ youtubeId?, videoId?, source, sourceId?, text, metadata? }` → `{ note: Note }` (201)
- DELETE: `{ noteId }` → `{ success: true }`
- Single handler with `req.method` dispatch (only route that does this)

#### `GET /api/notes/all`
- AUTHENTICATED + CSRF
- Joins `user_notes` → `video_analyses` for video metadata
- Returns: `{ notes: NoteWithVideo[] }`

#### `POST /api/link-video`
- AUTHENTICATED, **no CSRF** ("CSRF not needed as auth is already required")
- Body: `{ videoId }` → `{ success, alreadyLinked }`
- Links anonymous analysis to authenticated user

#### `POST /api/toggle-favorite`
- AUTHENTICATED, no CSRF
- Body: `{ videoId, isFavorite }` → `{ success, isFavorite }`
- Uses Supabase `upsert` with `onConflict: 'user_id,video_id'`

#### `POST /api/auth/signout`
- **No security middleware** (bare export)
- Deletes all `sb-*` cookies

### 2.7 Audit Logging (`lib/audit-logger.ts`)

Captured events in `audit_logs` table:
- **Auth**: LOGIN, LOGOUT, SIGNUP, PASSWORD_RESET
- **Video**: VIDEO_ANALYSIS_CREATE, VIDEO_ANALYSIS_UPDATE, VIDEO_FAVORITE_TOGGLE
- **AI**: AI_GENERATION, AI_CHAT
- **Billing**: SUBSCRIPTION_CREATED/UPDATED/CANCELED, PAYMENT_FAILED
- **Security**: RATE_LIMIT_EXCEEDED, VALIDATION_FAILED, UNAUTHORIZED_ACCESS

Sanitization: redacts passwords/tokens/keys, truncates long strings, masks emails.

### 2.8 Async Utilities (`lib/promise-utils.ts`)

| Utility | Signature | Purpose |
|---|---|---|
| `safePromise<T>` | `(promise) => Promise<[T \| null, Error \| null]>` | Go-style error handling |
| `backgroundOperation(name, fn, onError?)` | Fire-and-forget with error logging | Non-critical saves |
| `withTimeout(promise, ms, controller?)` | Timeout with AbortController | Request timeouts |
| `AbortManager` | Class with `add()`, `abortAll()`, `cleanup()` | Centralized abort management |

### 2.9 Root Middleware (`middleware.ts`)

Applied to all paths except `/api/webhooks`, static files, images.

1. **Supabase Session Update**: `updateSession(request)`
2. **Content-Security-Policy**:
   - `script-src`: self, YouTube, Google APIs (`*.googleapis.com`), Stripe
   - `frame-src`: YouTube only
   - `connect-src`: Supabase, Google APIs, YouTube, Stripe
   - `img-src`: `i.ytimg.com`, `img.youtube.com`, `*.ytimg.com`, `lh3.googleusercontent.com`
3. **Security headers**: same set as API middleware
4. **HSTS** in production

---

## 3. Component Architecture

### 3.1 Component Hierarchy

```
RootLayout (app/layout.tsx)
├── AuthProvider (contexts/auth-context.tsx)
├── Header
│   └── UserMenu (avatar / "Sign In")
├── Main Content
│   └── AnalyzePage (app/analyze/[videoId]/page.tsx)
│       ├── VideoHeader (title, author, favorite)
│       ├── YouTubePlayer (iframe + command execution)
│       ├── HighlightsPanel
│       │   ├── ThemeSelector (chip buttons + custom input)
│       │   ├── TopicCard[] (highlight reels with color)
│       │   └── VideoProgressBar (timeline overlay)
│       └── RightColumnTabs
│           ├── TranscriptViewer (segments, search, translation)
│           ├── AIChat (non-streaming, citations, presets)
│           └── NotesPanel (grouped by source)
│               └── NoteEditor (AI enhance, timestamp capture)
├── AuthModal (Google OAuth + email/password)
├── ToastProvider (sonner)
└── Analytics (Vercel)
```

**Shared components:** `SelectionActions`, `ChatMessage`, `SuggestedQuestions`, `ImageCheatsheetCard`, `LanguageSelector`

### 3.2 State Management

**AnalyzePage**: ~40 `useState` hooks (all state local to page — no global store except AuthContext).

Key state groups:
- **Video**: `videoId`, `videoInfo`, `transcript`, `videoDuration`, `currentTime`
- **Topics**: `topics`, `baseTopics`, `themes`, `selectedTheme`, `themeTopicsMap`, `usedTopicKeys`
- **Playback**: `playbackCommand`, `selectedTopic`, `isPlayingAll`, `playAllIndex`
- **Notes**: `notes`, `editingNote`, `selectedText`, `selectionMetadata`
- **UI**: `error`, `isRateLimitError`, `isExportOpen`

### 3.3 Playback Command System

**Type** (`lib/types.ts`):
```typescript
type PlaybackCommandType = 'SEEK' | 'PLAY_TOPIC' | 'PLAY_SEGMENT' | 'PLAY' | 'PAUSE' | 'PLAY_ALL' | 'PLAY_CITATIONS';

interface PlaybackCommand {
  type: PlaybackCommandType;
  time?: number;
  topic?: Topic;
  segment?: TranscriptSegment;
  citations?: Citation[];
  autoPlay?: boolean;
}
```

**Flow:**
1. Parent sets `playbackCommand` state
2. `YouTubePlayer` receives via prop, watches in `useEffect`
3. If player not ready → queues in `pendingPlaybackCommandRef`
4. Once ready (`onReady`) → executes via `playerRef.current` methods:
   - `seekTo(time, true)` for seek
   - `playVideo()` / `pauseVideo()`
   - `PLAY_ALL`: iterates topics with `setInterval` checking `getCurrentTime()`
   - `PLAY_CITATIONS`: builds temp topic with `isCitationReel=true`, auto-advances

### 3.4 YouTubePlayer (`components/youtube-player.tsx`)

**Props:** `videoId`, `selectedTopic`, `onTimeUpdate`, `playbackCommand`, `onCommandExecuted`, `onPlayerReady`, `topics`, `onTopicSelect`, `onPlayTopic`, `transcript`, `isPlayingAll`/`playAllIndex`/`setPlayAllIndex`/`setIsPlayingAll`, `renderControls`, `onDurationChange`, `selectedLanguage`, `onRequestTranslation`

**Exposed handle** (`useImperativeHandle`): `seekTo(time)`

**Implementation:**
- YouTube IFrame Player API (`YT.Player`)
- Dynamic script loading
- Time update: 100ms internal, throttled to 500ms for external callback
- Cleanup: destroys player on unmount/videoId change

### 3.5 RightColumnTabs (`components/right-column-tabs.tsx`)

**Props:** transcript, selectedTopic, onTimestampClick, currentTime, topics, citationHighlight, videoId, videoTitle, videoInfo, onCitationClick, showChatTab, cachedSuggestedQuestions, notes/onSaveNote/editingNote/onSaveEditingNote/onCancelEditing, onTakeNoteFromSelection, isAuthenticated/onRequestSignIn, selectedLanguage/translationCache/onRequestTranslation/onLanguageChange/availableLanguages/currentSourceLanguage, onRequestExport/exportButtonState, onAddNote

**Exposed handle** (`forwardRef`): `switchToTranscript()`, `switchToChat()`, `switchToNotes()`

**Tabs:** Transcript | Chat | Notes (Chat hidden when `showChatTab=false`)

### 3.6 AIChat (`components/ai-chat.tsx`)

**Props:** transcript, topics, videoId, videoTitle?, videoInfo?, onCitationClick, onTimestampClick, cachedSuggestedQuestions?, onSaveNote, onTakeNoteFromSelection, selectedLanguage, translationCache, onRequestTranslation, isAuthenticated, onRequestSignIn

**Local state:** `messages`, `input`, `isLoading`, `suggestedQuestions`, `loadingQuestions`, `followUpQuestions`, `loadingFollowUps`, `followUpAnchorId`, `askedQuestions`

**Key behaviors:**
- **NOT streaming** — full JSON response from `/api/chat`
- Retry logic: up to 2 retries for AbortError/temporarily unavailable/empty responses
- Presets: "Key Takeaways" (calls `/api/generate-summary`), "Top Quotes" (calls `/api/top-quotes`)
- Follow-up questions generated per assistant reply
- `SelectionActions` integration with `EXPLAIN_SELECTION_EVENT` custom event

### 3.7 TranscriptViewer (`components/transcript-viewer.tsx`)

**Props:** transcript, selectedTopic, onTimestampClick, currentTime?, topics?, citationHighlight?, onTakeNoteFromSelection, videoId?, selectedLanguage, onRequestTranslation, onRequestExport, exportButtonState

**Key features:**
- Per-segment highlighting with character-offset precision
- Search with prev/next navigation
- Auto-scroll to current segment while playing (disabled on user scroll)
- Per-segment lazy translation
- `SelectionActions` with transcript position metadata

### 3.8 NotesPanel & NoteEditor

**NotesPanel props:** notes[], onDeleteNote, editingNote, onSaveEditingNote, onCancelEditing, isAuthenticated, onSignInClick, currentTime?, onTimestampClick?, onAddNote?

- Notes grouped by source (chat/takeaways/transcript/custom)
- Empty states: sign-in prompt (unauthenticated) vs instructional (authenticated)

**NoteEditor props:** selectedText, metadata?, currentTime?, onSave, onCancel
- Two text areas: quote + additional notes
- "Enhance with AI" button (calls `/api/notes/enhance`)
- "Capture Timestamp" button (captures current video time)

### 3.9 AuthModal (`components/auth-modal.tsx`)

**Props:** open, onOpenChange, onSuccess?, trigger? (`'generation-limit'` | `'save-video'` | `'manual'` | `'save-note'`), currentVideoId?

**Tabs:** Sign In / Sign Up
**Methods:** Google OAuth (`signInWithOAuth`), email/password
**Post-auth flow:**
1. Stores `pendingVideoId` in `sessionStorage` before sign-in
2. Home page detects pending video on mount
3. Calls `/api/link-video` with exponential backoff
4. Clears `pendingVideoId` on success

### 3.10 Other Components

**SelectionActions**: Detects text selection, floating toolbar with "Explain" + "Take Notes". Uses `SafePortal` for DOM positioning.

**VideoHeader**: Shows title, author, duration, favorite toggle (auth-gated). Optimistic UI.

**UrlInput**: Real-time URL validation via `extractVideoId()`, mode selector, "I'm Feeling Lucky".

**HighlightsPanel**: Three states: generating (spinner), pre-generation (Sparkles CTA), has topics (progress bar + play/pause).

**ThemeSelector**: Chip buttons for themes, "Your Topic" inline input (max 60 chars), "Overall highlights" reset.

**TopicCard**: Color-coded (HSL from index+videoId), shows dot + title + duration, click = select + play.

**ChatMessage**: User (right-aligned card), Assistant (markdown + citation buttons `[1]`, `[2]`), images (clickable thumbnail zoom).

**SummaryViewer**: Markdown parsing into collapsible sections, clickable timestamp buttons, copy/save actions.

---

## 4. Database Schema

### 4.1 Tables (13 total)

#### `video_analyses` — Core cache table
| Column | Type | Notes |
|---|---|---|
| `id` | `uuid PK` | `uuid_generate_v4()` |
| `youtube_id` | `text UNIQUE NOT NULL` | YouTube video ID |
| `title` | `text NOT NULL` | |
| `author` | `text` | |
| `duration` | `integer NOT NULL` | Seconds |
| `thumbnail_url` | `text` | |
| `transcript` | `jsonb NOT NULL` | TranscriptSegment[] |
| `topics` | `jsonb` | Topic[] |
| `summary` | `jsonb` | Summary object |
| `suggested_questions` | `jsonb` | |
| `model_used` | `text` | |
| `language` | `text` | ISO code |
| `available_languages` | `jsonb` | |
| `created_by` | `uuid FK→users` | Owner |
| `created_at` | `timestamptz NOT NULL` | |
| `updated_at` | `timestamptz NOT NULL` | Auto-updated |

#### `profiles` — User profile
| Column | Type | Notes |
|---|---|---|
| `id` | `uuid PK FK→auth.users` | |
| `email` | `text` | |
| `subscription_tier` | `text NOT NULL` | `'free'\|'basic'\|'premium'` |
| `free_generations_used` | `integer NOT NULL DEFAULT 0` | |
| `topup_credits` | `integer NOT NULL DEFAULT 0` | |
| `topic_generation_mode` | `text NOT NULL` | `'smart'\|'fast'` |

#### `user_notes` — Personal notes
| Column | Type | Notes |
|---|---|---|
| `id` | `uuid PK` | |
| `user_id` | `uuid FK→auth.users NOT NULL` | |
| `video_id` | `uuid FK→video_analyses NOT NULL` | |
| `source` | `text NOT NULL` | `'chat'\|'takeaways'\|'transcript'\|'custom'` |
| `source_id` | `text` | Optional reference |
| `note_text` | `text NOT NULL` | The content |
| `metadata` | `jsonb` | NoteMetadata |

#### `user_videos` — User-video relationship
| Column | Type | Notes |
|---|---|---|
| `id` | `uuid PK` | |
| `user_id` | `uuid FK→profiles NOT NULL` | |
| `video_id` | `uuid FK→video_analyses NOT NULL` | |
| `is_favorite` | `boolean NOT NULL DEFAULT false` | |
| `accessed_at` | `timestamptz NOT NULL` | |
| **UNIQUE** | `(user_id, video_id)` | |

#### `video_generations` — Quota tracking
| Column | Type | Notes |
|---|---|---|
| `id` | `uuid PK` | |
| `user_id` | `uuid FK→auth.users` | Nullable for anonymous |
| `identifier` | `text NOT NULL` | `user:{id}` or `anon:{hash}` |
| `youtube_id` | `text NOT NULL` | |
| `video_id` | `uuid FK→video_analyses` | |
| `counted_toward_limit` | `boolean NOT NULL DEFAULT true` | |
| `subscription_tier` | `text` | |

#### Other tables:
- `rate_limits` — Sliding window rate limiting
- `audit_logs` — Security/audit events
- `image_generations` — Image generation quota
- `topup_purchases` — One-time credit purchases
- `stripe_events` — Stripe webhook deduplication
- `pending_welcome_emails` — Email queue

### 4.2 Materialized Views (5)

| View | Purpose |
|---|---|
| `user_activity_summary` | DAU aggregation |
| `user_growth_metrics` | Daily signups + tier distribution |
| `revenue_metrics` | MRR, subs, top-up revenue |
| `video_usage_metrics` | Daily generation + caching efficiency |
| `feature_adoption_metrics` | Smart/fast mode, notes, favorites |

### 4.3 Key RPC Functions (12)

| Function | Purpose |
|---|---|
| `consume_video_credit_atomically()` | Atomic credit check + consume with dedup |
| `check_video_generation_allowed()` | Read-only preflight |
| `insert_video_analysis_server()` | Secure insert/update with ownership |
| `update_video_analysis_secure()` | Ownership-verified update |
| `consume_image_credit_atomically()` | Image generation quota |
| `increment_topup_credits()` | Add purchased credits |

### 4.4 RLS Policies

- `profiles`: User SELECT/UPDATE own; Service role ALL
- `video_analyses`: Anyone SELECT; Auth'd INSERT/UPDATE; Service ALL
- `user_videos`: User SELECT/INSERT/UPDATE/DELETE own
- `user_notes`: User SELECT/INSERT/UPDATE/DELETE own
- `rate_limits`: Anyone SELECT/INSERT; Service DELETE

### 4.5 Supabase Clients

**Browser** (`lib/supabase/client.ts`):
- `createBrowserClient(url, anonKey)` from `@supabase/ssr`
- Runtime validation of env vars

**Server** (`lib/supabase/server.ts`):
- `createServerClient(url, anonKey)` from `@supabase/ssr`
- Async `cookies()` from `next/headers`
- Explicit `getAll()`/`setAll()` with silent failure on Server Components

**Middleware** (`lib/supabase/middleware.ts`):
- `updateSession(request)` for use in root `middleware.ts`
- Auto-refreshes session, deletes invalid tokens

---

## 5. AI Processing Pipeline

### 5.1 Transcript Chunking (`lib/ai-processing.ts`)

```typescript
function chunkTranscript(
  segments: TranscriptSegment[],
  chunkDurationSeconds: number = 300,    // 5 min default
  overlapSeconds: number = 45
): TranscriptChunk[]
```

**Algorithm:**
1. Effective duration: `max(180, chunkDurationSeconds)`
2. Effective overlap: `min(max(overlapSeconds, 0), floor(chunkDuration / 2))`
3. Step: `max(60, chunkDuration - overlap)`
4. Sliding window: advance by step, collect segments until `windowEndTarget`
5. Tail chunk if final gap > 5 seconds

### 5.2 Topic Generation (`generateTopicsFromTranscript`)

**Three-tier fallback:**

1. **Provider-backed** (primary + fallback provider):
   - Smart mode OR forceFullTranscript: `runSinglePassTopicGeneration()` — full transcript in 1 call
   - Fast mode + short video (<30min): tries single-pass first
   - Chunked pipeline:
     - Split into 5-min chunks, parallel AI calls (up to 2 candidates/chunk)
     - Deduplicate candidates
     - Split into time boundaries (first-3/5, last-2/5)
     - Reduce each subset via AI selection

2. **Local fallback** (`buildFallbackTopics`):
   - 6 equal segments, generic titles `"Highlights from 05:00-10:00"`

### 5.3 Model Selection

```typescript
function getProviderBackedTopicModel(mode, videoDuration, fastModel, proModel):
  if mode !== 'smart'       → fastModel
  if videoDuration <= 30min → fastModel
  else                      → proModel
```

### 5.4 Quote Matching (`lib/quote-matcher.ts`)

**Multi-strategy matching:**
1. **Exact** (Boyer-Moore substring search)
2. **Normalized** (normalized text comparison)
3. **Fuzzy** (3-gram Jaccard similarity coefficient)
4. **Timestamp-range** (segment joining by proximity)

**Transcript indexing:** Builds word positions + n-gram maps for efficient search.

### 5.5 Zod Schemas (`lib/schemas.ts`)

| Schema | Type | Used By |
|---|---|---|
| `topicGenerationSchema` | `z.array({ title, quote?: { timestamp, text } })` | Topic generation |
| `topicQuoteSchema` | `{ timestamp: /^\[\d+:\d+-\d+:\d+\]$/, text: max(20000) }` | Quote validation |
| `suggestedQuestionsSchema` | `z.array(z.string())` | Questions |
| `chatResponseSchema` | `{ answer: string, timestamps: string[] }` | Chat response |
| `summaryTakeawaysSchema` | `z.array(...).min(4).max(6)` | Summary |
| `quickPreviewSchema` | `{ overview: string }` | Preview |
| `topQuotesSchema` | `z.array(...).min(1).max(5)` | Top quotes |

---

## 6. Type System

### 6.1 Core Types (`lib/types.ts` — 171 lines)

```typescript
interface TranscriptSegment {
  text: string;
  start: number;        // seconds
  duration: number;
  translatedText?: string;
  startSegmentIdx?: number;
  endSegmentIdx?: number;
  startCharOffset?: number;
  endCharOffset?: number;
}

interface Topic {
  id: string;
  title: string;
  translatedTitle?: string;
  description?: string;
  duration: number;
  segments: TranscriptSegment[];
  keywords?: string[];
  quote?: { timestamp: string; text: string };
  isCitationReel?: boolean;
  autoPlay?: boolean;
}

interface TopicCandidate {
  key: string;
  title: string;
  translatedTitle?: string;
  quote: { timestamp: string; text: string };
}

type TopicGenerationMode = 'smart' | 'fast';

interface VideoData {
  videoId: string;
  title: string;
  transcript: TranscriptSegment[];
  topics: Topic[];
}

interface Citation {
  number: number;
  text: string;
  start: number; end: number;
  startSegmentIdx: number; endSegmentIdx: number;
  startCharOffset: number; endCharOffset: number;
}

interface ChatMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  citations?: Citation[];
  timestamp: string;
  imageUrl?: string;
  imageMetadata?: { prompt?: string; model?: string };
}

interface VideoInfo {
  videoId: string;
  title: string;
  author: string;
  thumbnail: string;
  duration: number | null;
  description?: string;
  tags?: string[];
  language?: string;
  availableLanguages?: string[];
}

type PlaybackCommandType = 'SEEK' | 'PLAY_TOPIC' | 'PLAY_SEGMENT' | 'PLAY' | 'PAUSE' | 'PLAY_ALL' | 'PLAY_CITATIONS';

interface PlaybackCommand {
  type: PlaybackCommandType;
  time?: number;
  topic?: Topic;
  segment?: TranscriptSegment;
  citations?: Citation[];
  autoPlay?: boolean;
}
```

### 6.2 Note Types

```typescript
type NoteSource = 'chat' | 'takeaways' | 'transcript' | 'custom';

interface NoteMetadata {
  transcript?: { start: number; end?: number; segmentIndex?: number; topicId?: string };
  chat?: { messageId: string; role: 'user' | 'assistant'; timestamp?: string };
  selectedText?: string;
  selectionContext?: string;
  timestampLabel?: string;
  extra?: Record<string, unknown>;
  [key: string]: unknown;
}

interface Note {
  id: string;
  userId: string;
  videoId: string;
  source: NoteSource;
  sourceId?: string;
  text: string;
  metadata?: NoteMetadata | null;
  createdAt: string;
  updatedAt: string;
}

interface NoteWithVideo extends Note {
  video: { youtubeId: string; title: string; author: string; thumbnailUrl: string; duration: number; slug?: string | null } | null;
}
```

---

## 7. Key Patterns & Conventions

### 7.1 File Structure
- **API routes**: `app/api/{name}/route.ts` exporting wrapped handlers
- **Components**: `components/{name}.tsx` (flat, no subdirectories for components)
- **Lib**: `lib/` for all business logic (providers, utils, types, supabase)
- **Types**: Centralized in `lib/types.ts` (not in individual files)

### 7.2 State Management
- No global state beyond AuthContext
- ~40 `useState` hooks in AnalyzePage, prop-drilled to children
- `forwardRef` + `useImperativeHandle` for imperative child access (YouTubePlayer, RightColumnTabs)

### 7.3 Data Flow
1. Cache check first (`check-video-cache`)
2. AI generation via providers
3. DB save via RPC (ownership-verified)
4. Credit consumption (save-then-consume, atomic via RPC)
5. Background operations for non-critical tasks

### 7.4 Error Handling
- Top-level try/catch in every API handler
- Zod validation returns 400 with `formatValidationError()`
- Graceful degradation: AI failures return fallback data, never 500 to user
- `backgroundOperation` for non-critical task errors (logged, not shown)

### 7.5 Authentication Flow
1. `pendingVideoId` in `sessionStorage` before auth redirect
2. Home page detects pending video → `/api/link-video`
3. Exponential backoff if video not yet persisted
4. Clear `sessionStorage` on success

### 7.6 CSS/Styling
- Tailwind CSS utility classes throughout
- No CSS-in-JS or CSS modules (except Tailwind)
- Consistent color system via `getTopicColor()`/`getTopicHSLColor()`

### 7.7 Testing
- Tests in `lib/__tests__/`
- Scripts in `scripts/` (validation, Stripe, etc.)
- No E2E tests visible in the codebase

---

## Apêndice: Env Vars do Projeto Original

| Env Var | Obrigatório | Provider | Uso |
|---|---|---|---|
| `SUPABASE_URL` | Sim | — | Supabase project |
| `SUPABASE_ANON_KEY` | Sim | — | Supabase auth |
| `SUPABASE_SERVICE_ROLE_KEY` | Sim | — | Service role ops |
| `GEMINI_API_KEY` | Condicional | Gemini | AI provider |
| `XAI_API_KEY` | Condicional | Grok | AI provider |
| `MINIMAX_API_KEY` | Condicional | MiniMax | AI provider |
| `AI_PROVIDER` | Não | — | Provider selection |
| `NEXT_PUBLIC_AI_PROVIDER` | Não | — | Client provider |
| `AI_DEFAULT_MODEL` | Não | — | Model override |
| `AI_FAST_MODEL` | Não | — | Fast model override |
| `AI_PRO_MODEL` | Não | — | Pro model override |
| `YOUTUBE_API_KEY` | Não | — | YouTube Data API |
| `STRIPE_SECRET_KEY` | Não | — | Stripe payments |
| `CSRF_SALT` | Sim | — | CSRF salt |
| `RESEND_API_KEY` | Não | — | Welcome emails |
| `POSTMARK_SERVER_TOKEN` | Não | — | Newsletter |
