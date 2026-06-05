# LongCut — Especificação Técnica

> **Projeto original:** [SamuelZ12/longcut](https://github.com/SamuelZ12/longcut) (AGPL-3.0)
> **Fork:** [rodrigorez/longcut_rez](https://github.com/rodrigorez/longcut_rez)
> **Modificações por:** rodrigorez
> **Data da spec:** Junho 2026

Este documento descreve as modificações planejadas para o LongCut, respeitando a licença AGPL-3.0 do projeto original. Todo o histórico original (483 commits) está preservado no git deste fork. O código original do fork encontra-se preservado em `.orig/`.

---

## Sumário das Modificações

| # | Feature | Status | Fase |
|---|---------|--------|------|
| 0 | YouTube Pinned Comment | Planejado | 0 |
| 1 | DeepSeek como Provider de IA | Planejado | 1 |
| 2 | Otimizar chunking p/ Gemini 1M tokens | Planejado | 2 |
| 3 | Screenshot nas Notas (thumbnail YouTube) | Planejado | 3 |
| 4 | Exportar Análise (.md / .txt) | Planejado | 4 |
| 5 | Export Dialog + Download | Planejado | 5 |
| 6 | Google Drive Upload | Planejado | 6 |
| 7 | Setup final + validação | Planejado | 7 |

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

- Faz a chamada à YouTube Data API
- Retorna `null` se não houver comentário fixado ou se a API key não estiver configurada
- Trata erros de cota e rate limit silenciosamente (graceful degradation)

#### `lib/types.ts` (MODIFICADO)
Adicionar ao `VideoInfo`:
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
- Design discreto: ícone de "📌" + texto do comentário (máx 3 linhas, expandível)

### Cache
O pinned comment é salvo junto com a análise no Supabase (`video_analyses.pinned_comment`), evitando chamadas repetidas à API.

---

## Fase 1 — DeepSeek como Provider

### Objetivo
Adicionar DeepSeek como provider de texto via API compatível com OpenAI.

### Provider Interface
O sistema já define o contrato em `lib/ai-providers/types.ts`:
```typescript
export interface ProviderAdapter {
  readonly name: string;
  readonly defaultModel: string;
  generate(params: ProviderGenerateParams): Promise<ProviderGenerateResult>;
}
```

DeepSeek usa o mesmo padrão de API que Grok (OpenAI-compatible), então criamos o adapter copiando `grok-adapter.ts` e alterando constantes.

### Arquivos

#### `lib/ai-providers/deepseek-adapter.ts` (NOVO)
```typescript
export function createDeepSeekAdapter(): ProviderAdapter
```

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
Adicionar nas tabelas:
```typescript
const PROVIDER_ORDER: ProviderKey[] = ['grok', 'gemini', 'minimax', 'deepseek'];

const PROVIDER_DEFAULT_MODELS: Record<ProviderKey, string> = {
  // ...
  deepseek: 'deepseek-chat',
};

const PROVIDER_BEHAVIORS: Record<ProviderKey, ProviderBehavior> = {
  // ...
  deepseek: {
    forceFullTranscriptTopicGeneration: false,
    forceSmartModeOnClient: false,
  },
};

const PROVIDER_ENV_KEYS: Record<ProviderKey, string> = {
  // ...
  deepseek: 'DEEPSEEK_API_KEY',
};
```

#### `lib/ai-providers/registry.ts` (MODIFICADO)
```typescript
import { createDeepSeekAdapter } from './deepseek-adapter';

const providerFactories: Record<ProviderKey, ProviderFactory> = {
  // ...
  deepseek: createDeepSeekAdapter,
};

const providerEnvGuards: Record<ProviderKey, () => string | undefined> = {
  // ...
  deepseek: () => process.env.DEEPSEEK_API_KEY,
};
```

### Configuração Recomendada

```bash
AI_PROVIDER=gemini           # Primário: Gemini 2.5 Flash (grátis)
NEXT_PUBLIC_AI_PROVIDER=gemini
DEEPSEEK_API_KEY=sk-...      # Fallback automático via registry
```

O sistema de fallback já existente no `registry.ts` tentará DeepSeek automaticamente se o provider primário falhar (rate limit, timeout, 5xx).

---

## Fase 2 — Otimizar Chunking p/ Gemini 1M Tokens

### Objetivo
Aproveitar a janela de contexto de **1M tokens** do Gemini 2.5 Flash para eliminar o chunking desnecessário no modo `smart` de geração de tópicos.

### Estado Atual
Em `lib/ai-processing.ts`, o modo `smart`:
1. Se vídeo > 30min: tenta enviar transcrição completa (1 chamada)
2. Se falha ou provider não suporta: faz chunking em janelas de ~5min (~14 chamadas para um vídeo de 60min)
3. Depois faz redução/seleção (2 chamadas adicionais)

### Mudança
Adicionar uma flag `hasLongContext` ao `ProviderBehavior`:
```typescript
export interface ProviderBehavior {
  forceFullTranscriptTopicGeneration: boolean;
  forceSmartModeOnClient: boolean;
  hasLongContext: boolean;  // NOVO: indica janela de contexto >= 1M tokens
}
```

Para o Gemini:
```typescript
gemini: {
  forceFullTranscriptTopicGeneration: false,
  forceSmartModeOnClient: false,
  hasLongContext: true,  // Gemini 2.5 Flash tem 1M tokens
}
```

Em `lib/ai-processing.ts`, quando `providerBehavior.hasLongContext === true`:
- Pular chunking completamente
- Enviar transcrição completa em 1 única chamada
- Aumentar `maxOutputTokens` para comportar mais resultados

### Impacto
| Cenário | Antes | Depois |
|---------|-------|--------|
| Vídeo 60min, modo smart | ~16 chamadas | **1 chamada** |
| Latência | 30-60s | **5-10s** |
| Custo (Gemini Flash) | ~$0.10 | **~$0.01** |

---

## Fase 3 — Screenshot nas Notas

### Objetivo
Ao criar uma nota em um timestamp específico, capturar automaticamente o frame correspondente do vídeo usando o endpoint de thumbnail do YouTube.

### Técnica
O YouTube disponibiliza thumbnails em timestamps arbitrários (não oficial, mas confiável):
```
https://img.youtube.com/vi/{videoId}/{segundos}.jpg
```
Qualidade: 320×180 (MQ). A CSP já permite `img-src` para `img.youtube.com`.

### Por que não "print" real?
O iframe do YouTube é cross-origin — o navegador bloqueia captura de canvas. Thumbnail por timestamp é a solução padrão da indústria.

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
- No `handleSave`, quando `capturedTimestamp` estiver setado:
  1. Gerar URL da thumbnail via `getThumbnailUrl(timestamp)`
  2. Incluir `thumbnailUrl` no `NoteMetadata`
- **Não fazer fetch** da imagem no momento do save (URL é gerada lazy)

#### `components/notes-panel.tsx` (MODIFICADO)
No card da nota (`sourceNotes.map`):
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
- Layout: thumbnail à esquerda, texto à direita (ou topo se mobile)
- Fallback: se thumbnail não carregar, mostrar placeholder

### Exportação
No `.md` com minutagem:
```markdown
![Frame em 03:45](https://img.youtube.com/vi/dQw4w9WgXcQ/225.jpg)
```

No `.txt`: apenas o timestamp.

---

## Fase 4 — Exportar Análise (.md / .txt)

### Objetivo
Exportar a análise completa do vídeo como arquivo Markdown ou TXT, com opção de incluir ou remover timestamps.

### Dados Incluídos

```
CABEÇALHO (sempre incluído):
  ─ Título do vídeo
  ─ Autor/Canal
  ─ URL do YouTube
  ─ Duração
  ─ Descrição completa do vídeo
  ─ Comentário fixado pelo autor (se houver)
  ─ Data de upload do vídeo
  ─ Data de exportação

CORPO (selecionável pelo usuário):
  ☑ Destaques (tópicos)
    ─ Título, descrição, quote com timestamp
    ─ Palavras-chave
  ☑ Resumo (takeaways)
  ☑ Transcrição completa
  ☑ Notas pessoais
    ─ Texto selecionado + anotação + thumbnail (apenas .md)
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

export interface ExportResult {
  content: string;
  filename: string;
  mimeType: string;
}
```

#### `lib/export/formatters.ts` (NOVO)

```typescript
export function buildMarkdown(data: ExportData, options: ExportOptions): string;
export function buildText(data: ExportData, options: ExportOptions): string;
```

**Regras de formatação (.md com minutagem):**
```markdown
# Título do Vídeo

**Autor:** Nome do Canal
**URL:** https://youtube.com/watch?v=XXXX
**Duração:** 1h 30min
**Data de upload:** 15 de março de 2026
**Data de exportação:** 5 de junho de 2026

---

## Descrição

Lorem ipsum…

---

## Destaques

### 1. Introdução à IA [00:15–05:30]
Descrição…

> "Citação relevante" — [02:30]

**Palavras-chave:** keyword1, keyword2
```

**Regras de formatação (.txt sem minutagem):**
```
TÍTULO DO VÍDEO
Autor: Nome do Canal
...

DESCRIÇÃO
...

DESTAQUES
1. Introdução à IA
Descrição…
Citação: "Citação relevante"
```

**Regras gerais:**
- Timestamps: `[MM:SS]` inline no markdown, omitidos no modo `without`
- URLs: sempre incluídas como links no .md, texto puro no .txt
- Thumbnails: incluídas como `![descrição](url)` no .md, ignoradas no .txt
- Comentário fixado: aparece como blockquote no .md, citação no .txt

---

## Fase 5 — Export Dialog + Download

### Objetivo
UI para o usuário configurar e disparar a exportação.

### Gate: Apenas Usuários Logados
- Botão "Exportar" só aparece se `isAuthenticated === true`
- Diferente do transcript export (que é Pro-only), este é gratuito para usuários logados

### Arquivos

#### `components/export-dialog.tsx` (NOVO)

```tsx
interface ExportDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  data: ExportData;
  videoId: string;
  isAuthenticated: boolean;
}
```

**Layout:**
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
│  ☑ Destaques (tópicos)               │
│  ☑ Resumo                            │
│  ☑ Transcrição completa              │
│  ☑ Notas pessoais                    │
│                                      │
│  ──────────────────────────          │
│                                      │
│  [  Baixar  ]  [ Salvar no Drive ]   │
│                                      │
│  O arquivo será salvo como           │
│  "Video Title - LongCut.md"          │
└──────────────────────────────────────┘
```

**Fluxo de download:**
1. Usuário clica "Baixar"
2. Chama `buildMarkdown()` ou `buildText()` com as opções selecionadas
3. Cria `Blob` com o conteúdo
4. `URL.createObjectURL()` + link `<a>` clicável
5. `URL.revokeObjectURL()` após download
6. Toast de sucesso

#### `components/export-button.tsx` (NOVO)
- Botão com ícone de download no cabeçalho da página
- Se não logado: tooltip "Faça login para exportar"
- Se logado: abre `ExportDialog`

#### `components/right-column-tabs.tsx` (MODIFICADO)
- Adicionar botão "Exportar" na tab bar
- Reutilizar `onRequestExport` prop já existente

#### `app/analyze/[videoId]/page.tsx` (MODIFICADO)
- Adicionar state `isExportOpen`
- Coletar `ExportData` de todos os states existentes
- Passar `onRequestExport` para `RightColumnTabs`

---

## Fase 6 — Google Drive Upload

### Objetivo
Upload do arquivo exportado diretamente para o Google Drive do usuário, sem passar por servidor intermediário.

### OAuth Flow
Usar **Google Identity Services (GIS)** — biblioteca oficial.

| Item | Detalhe |
|---|---|
| Cliente | OAuth 2.0 Web Application (Google Cloud Console) |
| Escopo | `https://www.googleapis.com/auth/drive.file` |
| Fluxo | `TokenClient` via `google.accounts.oauth2.initTokenClient()` |
| Pop-up | Abre janela de autorização do Google |

### Upload API
```
POST https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart
Authorization: Bearer {accessToken}
Content-Type: multipart/related; boundary=...

--boundary
Content-Type: application/json; charset=UTF-8

{ "name": "Video Title - LongCut.md", "mimeType": "text/markdown" }

--boundary
Content-Type: text/markdown

{conteúdo do arquivo}
--boundary--
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
interface UseGoogleDriveResult {
  uploadToDrive: (content: string, filename: string, mimeType: string) => Promise<void>;
  isUploading: boolean;
  error: string | null;
  isAvailable: boolean;  // false se GOOGLE_CLIENT_ID não configurado
}

export function useGoogleDrive(): UseGoogleDriveResult
```

**Funcionamento:**
1. Verifica se `NEXT_PUBLIC_GOOGLE_CLIENT_ID` está definido
2. Carrega script GIS dinamicamente
3. `initTokenClient({ client_id, scope, callback })`
4. `tokenClient.requestAccessToken()` → abre pop-up OAuth
5. De posse do token, chama `uploadToGoogleDrive()`
6. Toast de sucesso com link "Ver no Drive"

#### `components/export-dialog.tsx` (MODIFICADO)
- Botão "Salvar no Drive" só aparece se `isAvailable === true`
- Estado de loading durante upload
- Mensagem de erro se falhar
- Link "Ver no Drive" após sucesso

#### `.env.local.example` (MODIFICADO)
```
NEXT_PUBLIC_GOOGLE_CLIENT_ID=your_client_id.apps.googleusercontent.com
```

### Considerações de Segurança
- Escopo `drive.file`: o app só vê/cria arquivos que ele próprio criar
- Sem servidor intermediário: token nunca passa pelo backend
- Sem refresh token: usuário precisa autorizar a cada export
- CSP: adicionar `https://accounts.google.com` aos `script-src`

---

## Fase 7 — Setup Final

### Objetivo
Configurar ambiente, validar variáveis e preparar para deploy.

### Arquivos

#### `middleware.ts` (MODIFICADO)
Adicionar `https://accounts.google.com` ao CSP `script-src` (Google Identity Services).

#### `scripts/validate-env.ts` (MODIFICADO)
Validar também `DEEPSEEK_API_KEY` se `AI_PROVIDER=deepseek`.

---

## Resumo de Arquivos

### NOVOS
```
lib/export/types.ts
lib/export/formatters.ts
lib/export/google-drive.ts
lib/hooks/use-google-drive.ts
lib/hooks/use-thumbnail.ts
lib/youtube-comment-provider.ts
lib/ai-providers/deepseek-adapter.ts
components/export-dialog.tsx
components/export-button.tsx
```

### MODIFICADOS
```
lib/types.ts                       ← NoteMetadata.thumbnailUrl, VideoInfo.pinnedComment
lib/ai-providers/types.ts          ← ProviderKey +deepseek
lib/ai-providers/provider-config.ts ← +DeepSeek, hasLongContext
lib/ai-providers/registry.ts       ← +DeepSeek factory
lib/ai-processing.ts               ← Otimizar chunking
lib/video-info-provider.ts         ← +pinnedComment
components/note-editor.tsx         ← +thumbnail na nota
components/notes-panel.tsx         ← +thumbnail no card
components/right-column-tabs.tsx   ← +botão exportar
components/export-dialog.tsx       ← +drive button
app/analyze/[videoId]/page.tsx     ← Integrar export + pinned comment + thumbnail
app/api/video-info/route.ts        ← +pinnedComment
middleware.ts                      ← CSP p/ Google Identity
.env.local.example                 ← +DEEPSEEK_API_KEY, +GOOGLE_CLIENT_ID
scripts/validate-env.ts            ← +deepseek validation
```

### NENHUM ARQUIVO ORIGINAL SERÁ ALTERADO
O histórico original (483 commits) está preservado no git. O código original do fork está preservado em `.orig/`. Todas as modificações são aditivas.

---

## Licença

Este projeto é um fork de [SamuelZ12/longcut](https://github.com/SamuelZ12/longcut), distribuído sob a licença **AGPL-3.0**.

> Copyright (C) 2026 rodrigorez
>
> Este programa é software livre: você pode redistribuí-lo e/ou modificá-lo sob os termos da GNU Affero General Public License publicada pela Free Software Foundation, na versão 3 da Licença.
>
> Este programa é distribuído na esperança de que seja útil, mas SEM QUALQUER GARANTIA; sem mesmo a garantia implícita de COMERCIALIDADE ou ADEQUAÇÃO A UM DETERMINADO FIM. Veja a GNU Affero General Public License para mais detalhes.

---

## Estrutura Final do Projeto

```
longcut/
├── .orig/                    ← Cópia do fork original (intocável)
├── lib/
│   ├── export/               ← NOVO: core de exportação
│   ├── hooks/                ← NOVO: use-google-drive, use-thumbnail
│   ├── ai-providers/
│   │   └── deepseek-adapter.ts ← NOVO
│   └── youtube-comment-provider.ts ← NOVO
├── components/
│   ├── export-dialog.tsx     ← NOVO
│   └── export-button.tsx     ← NOVO
├── SPEC.md                   ← Este arquivo
└── ... demais arquivos originais do fork (intocados) ...
```
