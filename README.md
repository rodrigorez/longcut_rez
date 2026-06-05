# LongCut Rez

> Fork de [SamuelZ12/longcut](https://github.com/SamuelZ12/longcut) com modificações próprias.

LongCut transforma vídeos longos do YouTube em um workspace de aprendizado estruturado. Cole uma URL e o app gera destaques, respostas com IA com timestamps, e um local para capturar suas anotações — absorva vídeos de uma hora em minutos.

**Projeto original:** [SamuelZ12/longcut](https://github.com/SamuelZ12/longcut) (AGPL-3.0)  
**Código original preservado em:** `.orig/`  
**Roadmap de modificações:** [`SPEC.md`](SPEC.md)

## Visão Geral

Next.js 15 + React 19 com roteamento de geração de texto através de adaptadores de provedores. Atualmente suporta MiniMax, xAI Grok e Google Gemini como provedores de texto, com DeepSeek planejado como fallback econômico. A extração de transcrições do YouTube é gratuita (sem API key). Supabase fornece autenticação, persistência e rate limiting.

## Funcionalidades

- Destaques por IA com modos Smart (qualidade) e Fast (velocidade), reprodução contínua ("Play All") e regeração por tema.
- Preview rápido, resumo estruturado, perguntas sugeridas e quotes memoráveis gerados em paralelo.
- Chat com IA contextualizado na transcrição com respostas JSON estruturadas e citações com timestamp.
- Visualizador de transcrição sincronizado com o player do YouTube.
- Sistema de anotações pessoais com fontes da transcrição, chat e takeaways.
- Biblioteca de vídeos salvos, favoritos e limite de geração por usuário.
- Middleware de segurança com CSP, CSRF e rate limiting.

## Arquitetura

- **Frontend:** Next.js 15 App Router, React 19, TypeScript, Tailwind CSS v4, shadcn/ui, lucide-react
- **Backend:** Route handlers do Next.js com middleware `withSecurity` (CSRF, validação Zod, rate limiting)
- **AI Pipeline:** `lib/ai-processing.ts` + `lib/ai-client.ts` orquestram prompts agnósticos a provedores via adaptadores em `lib/ai-providers/`
- **Transcrição:** Extração direta de legendas públicas do YouTube (gratuita)
- **Persistência:** Supabase (video_analyses, user_notes, rate_limits, perfis)
- **Autenticação:** Supabase Auth com sessão via middleware

## Roadmap

Consulte [`SPEC.md`](SPEC.md) para o plano completo de implementação:

- Fase 0: YouTube Pinned Comment
- Fase 1: DeepSeek como provider de IA
- Fase 2: Otimização de chunking p/ Gemini 1M tokens
- Fase 3: Screenshot nas notas (thumbnail do YouTube)
- Fase 4: Exportar análise (.md / .txt)
- Fase 5: Export Dialog + Download
- Fase 6: Google Drive Upload
- Fase 7: Setup final + validação

## Primeiros Passos

### Pré-requisitos

- Node.js 18+ (Next.js 15 requer 18.18 ou superior)
- npm ou pnpm
- Projeto Supabase (gratuito)
- Chave de API de um provedor de IA (recomendado: Gemini 2.5 Flash — grátis)

### Instalação

```bash
git clone https://github.com/rodrigorez/longcut_rez.git
cd longcut_rez
npm install
```

### Configuração de Ambiente

Crie `.env.local` na raiz:

| Variável | Obrigatório | Descrição |
|---|---|---|
| `AI_PROVIDER` | recomendado | `gemini`, `minimax`, `grok` ou `deepseek` |
| `NEXT_PUBLIC_AI_PROVIDER` | recomendado | Deve coincidir com `AI_PROVIDER` |
| `GEMINI_API_KEY` | sim* | Google Gemini (texto + imagem) |
| `DEEPSEEK_API_KEY` | opcional | Fallback via DeepSeek |
| `XAI_API_KEY` | opcional | Fallback via Grok |
| `MINIMAX_API_KEY` | opcional | Fallback via MiniMax |
| `NEXT_PUBLIC_SUPABASE_URL` | sim | URL do projeto Supabase |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | sim | Chave anônima Supabase |
| `SUPABASE_SERVICE_ROLE_KEY` | sim | Chave de serviço Supabase |
| `CSRF_SALT` | sim | String aleatória para tokens CSRF |
| `YOUTUBE_API_KEY` | opcional | Habilita metadados extras + pinned comment |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | opcional | Habilita upload p/ Google Drive |
| `NEXT_PUBLIC_APP_URL` | opcional | URL canônica do app |

<sup>\*</sup> `GEMINI_API_KEY` é necessária para geração de texto se `AI_PROVIDER=gemini`, e para geração de imagem via `/api/generate-image`.

> Setup recomendado: `AI_PROVIDER=gemini`, `GEMINI_API_KEY=...`, `DEEPSEEK_API_KEY=...` (fallback). Gemini 2.5 Flash tem faixa gratuita generosa (60 req/min, 1M tokens de contexto).

### Supabase

1. Execute as migrations em `supabase/migrations/` no SQL editor do Supabase
2. Habilite os provedores de autenticação desejados
3. Configure as URLs de redirect para seu domínio

### Executar

```bash
npm run dev        # Inicia com Turbopack em http://localhost:3000
npm run lint       # ESLint
```

## Estrutura do Projeto

```
longcut_rez/
├── .orig/                     ← Cópia local do fork original (não versionada)
├── app/
│   ├── api/                   × Route handlers
│   ├── analyze/[videoId]/     × Página de análise
│   ├── all-notes/             × Dashboard de notas
│   ├── my-videos/             × Biblioteca de vídeos
│   └── settings/              × Configurações
├── components/
│   ├── ui/                    × Primitivas shadcn/ui
│   ├── ai-chat.tsx            × Chat com IA
│   ├── highlights-panel.tsx   × Painel de destaques
│   └── notes-panel.tsx        × Painel de notas
├── contexts/
│   └── auth-context.tsx       × Provedor de autenticação
├── lib/
│   ├── ai-providers/          × Adaptadores: grok, gemini, minimax (+ deepseek em breve)
│   ├── supabase/              × Clientes browser/server
│   ├── ai-client.ts           × Ponto de entrada da IA
│   └── utils.ts               × Utilitários diversos
├── public/                    × Assets estáticos
├── supabase/migrations/       × Migrations SQL
├── SPEC.md                    × Roadmap de modificações
├── CLAUDE.md                  × Guia original do projeto
├── LICENSE                    × AGPL-3.0
└── next.config.ts             × Config Next.js
```

## Notas do Desenvolvedor

- Requisições com mudança de estado devem usar `csrfFetch` (valida CSRF automática)
- Rate limiting usa tabela `rate_limits` no Supabase
- Modo de geração de tópicos (`smart` vs `fast`) é persistido por perfil
- Consulte [`SPEC.md`](SPEC.md) para o plano de implementação das próximas features
- O código original do fork está preservado em `.orig/` para referência

## Contribuição

Este é um fork pessoal com modificações próprias. Pull requests não são esperados, mas issues são bem-vindas.

## Licença

Distribuído sob [GNU Affero General Public License v3.0](LICENSE).

```
Copyright (C) 2026 rodrigorez

Este programa é software livre: você pode redistribuí-lo e/ou modificá-lo 
sob os termos da GNU Affero General Public License publicada pela Free 
Software Foundation, na versão 3 da Licença.

Este programa é distribuído na esperança de que seja útil, mas SEM QUALQUER 
GARANTIA; sem mesmo a garantia implícita de COMERCIALIDADE ou ADEQUAÇÃO A 
UM DETERMINADO FIM.
```

Projeto original: [SamuelZ12/longcut](https://github.com/SamuelZ12/longcut) — licensiado sob AGPL-3.0.
