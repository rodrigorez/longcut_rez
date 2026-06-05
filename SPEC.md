# LongCut Rez — Especificação Técnica

> **Projeto original:** [SamuelZ12/longcut](https://github.com/SamuelZ12/longcut) (AGPL-3.0)
> **Fork:** [rodrigorez/longcut_rez](https://github.com/rodrigorez/longcut_rez)
> **Modificações por:** rodrigorez
> **Data da spec:** Junho 2026

---

## Status do Projeto

| Item | Valor |
|---|---|
| Branch | `main` |
| Último commit fork | `ff98c31` — chore: initial fork setup |
| Último commit original | `c80c874` — Merge pull request #66 (SamuelZ12) |
| Total commits | 484 (483 originais + 1 nosso) |
| Remote | `https://github.com/rodrigorez/longcut_rez.git` |
| Backup original | `.orig/` — 253 arquivos, cópia fiel do fork |

---

## Estrutura do .orig

O diretório `.orig/` contém o código fonte original do fork `rodrigorez/longcut_rez` (idêntico ao repositório `SamuelZ12/longcut`). Nenhum destes arquivos será modificado.

```
.orig/
├── .github/                    ← GitHub Actions workflows
├── .jules/                     ← Jules config
├── app/                        ← Next.js App Router (pages + API)
│   ├── api/                    ← 32 rotas de API
│   ├── analyze/[videoId]/      ← Página principal de análise
│   ├── all-notes/              ← Dashboard de notas
│   ├── my-videos/              ← Biblioteca de vídeos
│   ├── settings/               ← Configurações de perfil
│   └── auth/                   ← Fragmentos de autenticação
├── components/                 ← Componentes React (ui/, ai-chat, notes-panel, etc.)
├── contexts/                   ← AuthContext (Supabase)
├── docs/                       ← Documentação
├── lib/                        ← Core (ai-providers/, supabase/, utils, types)
│   ├── ai-providers/           ← Adapters: grok, gemini, minimax
│   ├── supabase/               ← Clientes browser/server
│   └── __tests__/              ← Testes unitários
├── public/                     ← Assets estáticos
├── resources/                  ← Recursos diversos
├── scripts/                    ← Scripts de validação, Stripe, etc.
├── supabase/                   ← Migrations SQL
├── .env.example                ← Exemplo de env vars
├── .env.local.example          ← Exemplo de env vars local
├── .gitignore                  ← Ignora node_modules, .env*, .next, etc.
├── CLAUDE.md                   ← Guia do projeto original
├── LICENSE                     ← AGPL-3.0
├── README.md                   ← Documentação original
├── components.json             ← shadcn/ui config
├── eslint                      ← ESLint config
├── eslint.config.mjs           ← ESLint flat config
├── middleware.ts                ← Supabase session + CSP headers
├── next.config.ts              ← Next.js config
├── package.json                ← Dependências (Next 15, React 19, etc.)
├── package-lock.json           ← Lock file (npm)
├── pnpm-lock.yaml              ← Lock file (pnpm)
├── postcss.config.mjs          ← PostCSS + Tailwind
└── tsconfig.json               ← TypeScript config
```

---

## Sumário das Modificações

| # | Feature | Fase | Prioridade |
|---|---------|------|------------|
| 0 | YouTube Pinned Comment | 0 | Média |
| 1 | DeepSeek como Provider de IA | 1 | Alta |
| 2 | Otimizar chunking p/ Gemini 1M tokens | 2 | Alta |
| 3 | Screenshot nas Notas (thumbnail YouTube) | 3 | Média |
| 4 | Exportar Análise (.md / .txt) | 4 | Alta |
| 5 | Export Dialog + Download | 5 | Alta |
| 6 | Google Drive Upload | 6 | Baixa |
| 7 | Setup final + validação | 7 | Alta |

---

## Fase 0 — YouTube Pinned Comment

### Objetivo
Buscar o comentário fixado pelo autor do vídeo (se houver) via YouTube Data API v3 e exibi-lo na página de análise.

### API
```
GET https://www.googleapis.com/youtube/v3/commentThreads
  ?part=snippet
  &videoId={videoId}
  &order=relevance
  &maxResults=1
  &key={YOUTUBE_API_KEY}
```

O primeiro resultado com `snippet.isPinned: true` é o comentário fixado.

### Arquivos

#### `lib/youtube-comment-provider.ts` (NOVO)
```typescript
export async function fetchPinnedComment(
  videoId: string,
  apiKey: string
): Promise<PinnedComment | null>
```

- Faz chamada à YouTube Data API
- Retorna `null` se não houver comentário fixado ou se `YOUTUBE_API_KEY` não estiver configurada
- Trata erros de cota e rate limit silenciosamente (graceful degradation)

#### `lib/types.ts` (MODIFICADO)
```typescript
export interface VideoInfo {
  // ... campos existentes
  pinnedComment?: {
    text: string;
    author: string;
    likes: number;
    publishedAt: string;
  };
}
```

#### `lib/video-info-provider.ts` (MODIFICADO)
- Integrar `fetchPinnedComment` dentro de `fetchYouTubeVideoInfo`
- Só chamar se `YOUTUBE_API_KEY` estiver definida

#### `app/api/video-info/route.ts` (MODIFICADO)
- O campo `pinnedComment` já é propagado automaticamente via `VideoInfo`

#### `app/analyze/[videoId]/page.tsx` (MODIFICADO)
- Exibir `pinnedComment` no cabeçalho ou acima dos tópicos
- Design: ícone "📌" + texto do comentário (máx 3 linhas, expandível)

### Cache
O pinned comment é salvo junto com a análise no Supabase (`video_analyses.pinned_comment`)

---

## Fase 1 — DeepSeek como Provider

### Objetivo
Adicionar DeepSeek como provider de texto via API compatível com OpenAI.

### Provider Interface
O sistema já define o contrato (`lib/ai-providers/types.ts`):
```typescript
export interface ProviderAdapter {
  readonly name: string;
  readonly defaultModel: string;
  generate(params: ProviderGenerateParams): Promise<ProviderGenerateResult>;
}
```

DeepSeek usa o mesmo padrão de API que Grok (OpenAI-compatible), então o adapter será criado a partir de `grok-adapter.ts`.

### Arquivos

#### `lib/ai-providers/deepseek-adapter.ts` (NOVO)
| Parâmetro | Valor |
|---|---|
| `name` | `'deepseek'` |
| `defaultModel` | `'deepseek-chat'` |
| `baseUrl` | `https://api.deepseek.com/v1` |
| `envKey` | `DEEPSEEK_API_KEY` |
| `baseUrlEnvOverride` | `DEEPSEEK_API_BASE_URL` |

Endpoint: `POST {baseUrl}/chat/completions`
Structured output: `response_format: { type: 'json_schema', json_schema: {...} }`

#### `lib/ai-providers/types.ts` (MODIFICADO)
```typescript
export type ProviderKey = 'grok' | 'gemini' | 'minimax' | 'deepseek';
```

#### `lib/ai-providers/provider-config.ts` (MODIFICADO)
```typescript
const PROVIDER_ORDER: ProviderKey[] = ['grok', 'gemini', 'minimax', 'deepseek'];
// + default model, behavior, env key
```

#### `lib/ai-providers/registry.ts` (MODIFICADO)
```typescript
import { createDeepSeekAdapter } from './deepseek-adapter';
// + factory + env guard
```

### Configuração Recomendada
```bash
AI_PROVIDER=gemini              # Gemini 2.5 Flash (grátis, 1M tokens)
NEXT_PUBLIC_AI_PROVIDER=gemini
DEEPSEEK_API_KEY=sk-...         # Fallback automático
```

O fallback automático do registry tenta DeepSeek se o provider primário falhar (rate limit, timeout, 5xx).

---

## Fase 2 — Otimizar Chunking p/ Gemini 1M Tokens

### Objetivo
Aproveitar a janela de **1M tokens** do Gemini 2.5 Flash para eliminar chunking no modo `smart` de geração de tópicos.

### Estado Atual (lib/ai-processing.ts)
1. Se vídeo > 30min: tenta enviar transcrição completa (1 chamada)
2. Se falha: chunking em janelas de ~5min
3. Redução/seleção (2 chamadas adicionais)

Para vídeo de 60min: **~14-16 chamadas LLM**.

### Mudança
Adicionar `hasLongContext` ao `ProviderBehavior`:
```typescript
export interface ProviderBehavior {
  forceFullTranscriptTopicGeneration: boolean;
  forceSmartModeOnClient: boolean;
  hasLongContext: boolean;    // NOVO: janela >= 1M tokens
}
```

Gemini:
```typescript
gemini: {
  forceFullTranscriptTopicGeneration: false,
  forceSmartModeOnClient: false,
  hasLongContext: true,
}
```

Quando `hasLongContext === true`:
- Pular chunking completamente
- Enviar transcrição completa em 1 chamada
- Aumentar `maxOutputTokens`

### Impacto
| Cenário | Antes | Depois |
|---------|-------|--------|
| Vídeo 60min, modo smart | ~16 chamadas | **1 chamada** |
| Latência | 30-60s | **5-10s** |
| Custo (Gemini Flash) | ~$0.10 | **~$0.01** |

---

## Fase 3 — Screenshot nas Notas

### Objetivo
Ao criar uma nota em um timestamp, capturar o frame do vídeo via thumbnail do YouTube.

### Técnica
```
https://img.youtube.com/vi/{videoId}/{segundos}.jpg
```
Qualidade: 320×180. CSP já permite `img.youtube.com`. O iframe do YouTube é cross-origin — não é possível capturar canvas real.

### Arquivos

#### `lib/types.ts` (MODIFICADO)
```typescript
export interface NoteMetadata {
  // ... campos existentes
  thumbnailUrl?: string;   // NOVO
}
```

#### `lib/hooks/use-thumbnail.ts` (NOVO)
```typescript
export function useThumbnail(videoId: string) {
  const getThumbnailUrl = useCallback((timestampSeconds: number): string => {
    return `https://img.youtube.com/vi/${videoId}/${Math.floor(timestampSeconds)}.jpg`;
  }, [videoId]);
  return { getThumbnailUrl };
}
```

#### `components/note-editor.tsx` (MODIFICADO)
- Receber `videoId` como prop
- No `handleSave`, quando `capturedTimestamp` estiver setado: gerar URL da thumbnail e incluir no `NoteMetadata`
- URL é gerada lazy — não fazer fetch no momento do save

#### `components/notes-panel.tsx` (MODIFICADO)
```tsx
{note.metadata?.thumbnailUrl && (
  <img
    src={note.metadata.thumbnailUrl}
    alt={`Frame em ${note.metadata.timestampLabel}`}
    className="w-[120px] h-[68px] object-cover rounded-md float-left mr-3 mb-1"
    loading="lazy"
  />
)}
```

### Na Exportação
- `.md` com minutagem: `![Frame em 03:45](https://img.youtube.com/vi/{id}/225.jpg)`
- `.txt`: apenas o timestamp

---

## Fase 4 — Exportar Análise (.md / .txt)

### Objetivo
Exportar análise completa como Markdown ou TXT, com/sem timestamps.

### Dados Incluídos
```
CABEÇALHO:
  ─ Título + URL + Autor + Duração
  ─ Descrição completa do vídeo
  ─ Comentário fixado (se houver)
  ─ Data de upload do vídeo
  ─ Data de exportação

CORPO (selecionável):
  ☑ Destaques (tópicos + quotes + keywords)
  ☑ Resumo (takeaways)
  ☑ Transcrição completa
  ☑ Notas pessoais
```

### Arquivos

#### `lib/export/types.ts` (NOVO)
```typescript
export type ExportFormat = 'markdown' | 'txt';
export type TimestampMode = 'with' | 'without';

export interface ExportOptions {
  format: ExportFormat;
  timestampMode: TimestampMode;
  includeTopics: boolean;
  includeSummary: boolean;
  includeTranscript: boolean;
  includeNotes: boolean;
}

export interface ExportData {
  title: string;
  author: string;
  url: string;
  videoId: string;
  duration: number | null;
  description?: string;
  pinnedComment?: PinnedComment | null;
  uploadDate?: string;
  exportDate: string;
  topics?: Topic[];
  summary?: string;
  transcript?: TranscriptSegment[];
  notes?: Note[];
}
```

#### `lib/export/formatters.ts` (NOVO)
```typescript
export function buildMarkdown(data: ExportData, options: ExportOptions): string;
export function buildText(data: ExportData, options: ExportOptions): string;
```

**Exemplo .md com minutagem:**
```markdown
# Título do Vídeo

**Autor:** Nome do Canal
**URL:** https://youtube.com/watch?v=XXXX
**Duração:** 1h 30min
**Data de exportação:** 5 de junho de 2026

---

## Destaques

### 1. Introdução à IA [00:15–05:30]
Descrição…

> "Citação relevante" — [02:30]
```

**Exemplo .txt sem minutagem:**
```
TÍTULO DO VÍDEO
Autor: Nome do Canal
...

DESCRIÇÃO
...

DESTAQUES
1. Introdução à IA
Descrição…
```

---

## Fase 5 — Export Dialog + Download

### Objetivo
UI para configurar e disparar a exportação.

### Gate
- Botão visível apenas se `isAuthenticated === true`
- Grátis para usuários logados (diferente do transcript export que é Pro-only)

### Arquivos

#### `components/export-dialog.tsx` (NOVO)
```
┌──────────────────────────────────────┐
│  Exportar Análise                    │
│                                      │
│  Formato:                            │
│  ○ Markdown (.md)   ○ Texto (.txt)   │
│                                      │
│  Minutagem:                          │
│  ○ Com timestamps   ○ Sem timestamps │
│                                      │
│  Incluir:                            │
│  ☑ Destaques  ☑ Resumo              │
│  ☑ Transcrição  ☑ Notas             │
│                                      │
│  [  Baixar  ]  [ Salvar no Drive ]   │
└──────────────────────────────────────┘
```

**Fluxo:** selecionar opções → `buildMarkdown()`/`buildText()` → `Blob` → `URL.createObjectURL()` → download → toast

#### `components/export-button.tsx` (NOVO)
- Ícone de download no cabeçalho da página
- Tooltip "Faça login para exportar" se não logado

#### `components/right-column-tabs.tsx` (MODIFICADO)
- Adicionar botão "Exportar" na tab bar

#### `app/analyze/[videoId]/page.tsx` (MODIFICADO)
- State `isExportOpen`
- Coletar `ExportData` dos states existentes

---

## Fase 6 — Google Drive Upload

### Objetivo
Upload direto do arquivo exportado para o Google Drive do usuário (sem servidor intermediário).

### OAuth
| Item | Detalhe |
|---|---|
| Biblioteca | Google Identity Services (GIS) |
| Escopo | `https://www.googleapis.com/auth/drive.file` |
| Fluxo | `google.accounts.oauth2.initTokenClient()` |

### Upload
```
POST https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart
Authorization: Bearer {accessToken}
```

### Arquivos

#### `lib/export/google-drive.ts` (NOVO)
```typescript
export async function uploadToGoogleDrive(
  accessToken: string,
  content: string,
  filename: string,
  mimeType: string
): Promise<{ fileId: string; webViewLink: string }>
```

#### `lib/hooks/use-google-drive.ts` (NOVO)
```typescript
export function useGoogleDrive(): UseGoogleDriveResult
// 1. Verifica NEXT_PUBLIC_GOOGLE_CLIENT_ID
// 2. Carrega GIS dinamicamente
// 3. initTokenClient() → pop-up OAuth
// 4. uploadToGoogleDrive()
// 5. Toast + link
```

#### `components/export-dialog.tsx` (MODIFICADO)
- Botão "Salvar no Drive" condicional (`isAvailable`)
- Loading + erro + link de sucesso

### Configuração
```
NEXT_PUBLIC_GOOGLE_CLIENT_ID=your_client_id.apps.googleusercontent.com
```

### Segurança
- Escopo `drive.file`: app só acessa arquivos que criar
- Token nunca passa pelo backend
- CSP: adicionar `https://accounts.google.com` aos `script-src`

---

## Fase 7 — Setup Final

### Arquivos

#### `middleware.ts` (MODIFICADO)
- Adicionar `https://accounts.google.com` ao CSP `script-src`

#### `scripts/validate-env.ts` (MODIFICADO)
- Validar `DEEPSEEK_API_KEY` se `AI_PROVIDER=deepseek`

#### `.env.local` (NOVO — não commitado)
```bash
# Provider primário (grátis)
AI_PROVIDER=gemini
NEXT_PUBLIC_AI_PROVIDER=gemini
GEMINI_API_KEY=...

# Fallback
DEEPSEEK_API_KEY=...
XAI_API_KEY=...

# Supabase
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...

# YouTube
YOUTUBE_API_KEY=...

# Segurança
CSRF_SALT=...

# Google Drive (opcional)
NEXT_PUBLIC_GOOGLE_CLIENT_ID=...

# Stripe (opcional)
STRIPE_SECRET_KEY=...
```

---

## Arquivos Novo vs Modificado

### NOVOS (9 arquivos)
```
lib/export/types.ts                  ← Tipos de exportação
lib/export/formatters.ts             ← buildMarkdown(), buildText()
lib/export/google-drive.ts           ← uploadToGoogleDrive()
lib/hooks/use-google-drive.ts        ← Hook OAuth + upload
lib/hooks/use-thumbnail.ts           ← Hook thumbnail URL
lib/youtube-comment-provider.ts      ← fetchPinnedComment()
lib/ai-providers/deepseek-adapter.ts ← Provider DeepSeek
components/export-dialog.tsx         ← Modal de exportação
components/export-button.tsx         ← Botão de exportar
```

### MODIFICADOS (15 arquivos)
```
lib/types.ts                       ← NoteMetadata.thumbnailUrl, VideoInfo.pinnedComment
lib/ai-providers/types.ts          ← ProviderKey +deepseek
lib/ai-providers/provider-config.ts ← +DeepSeek config, hasLongContext
lib/ai-providers/registry.ts       ← +DeepSeek factory
lib/ai-processing.ts               ← Otimizar chunking p/ Gemini
lib/video-info-provider.ts         ← +pinnedComment
components/note-editor.tsx         ← +thumbnail na nota
components/notes-panel.tsx         ← +thumbnail no card
components/right-column-tabs.tsx   ← +botão exportar
components/export-dialog.tsx       ← +Google Drive button
app/analyze/[videoId]/page.tsx     ← Export + pinnedComment + thumbnail
app/api/video-info/route.ts        ← +pinnedComment na resposta
middleware.ts                      ← CSP Google Identity
.env.local.example                 ← +DEEPSEEK_API_KEY, +GOOGLE_CLIENT_ID
scripts/validate-env.ts            ← +deepseek validation
```

---

## Workflow de Desenvolvimento

### Iniciar Implementação de uma Fase
```bash
git checkout -b feat/fase-1-deepseek
# implementar...
git add -A
git commit -m "feat: add DeepSeek provider adapter"
git push -u origin feat/fase-1-deepseek
```

### Restaurar .orig se Necessário
```bash
# Restaurar um único arquivo
cp .orig/lib/types.ts lib/types.ts

# Verificar diff com original
diff .orig/lib/ai-providers/provider-config.ts lib/ai-providers/provider-config.ts
```

### Publicar no GitHub
```bash
git push -u origin main
```

---

## Licença

Este projeto é um fork de [SamuelZ12/longcut](https://github.com/SamuelZ12/longcut), distribuído sob a licença **AGPL-3.0**.

> Copyright (C) 2026 rodrigorez
>
> Este programa é software livre: você pode redistribuí-lo e/ou modificá-lo sob os termos da GNU Affero General Public License publicada pela Free Software Foundation, na versão 3 da Licença.
>
> Este programa é distribuído na esperança de que seja útil, mas SEM QUALQUER GARANTIA; sem mesmo a garantia implícita de COMERCIALIDADE ou ADEQUAÇÃO A UM DETERMINADO FIM. Veja a GNU Affero General Public License para mais detalhes.
