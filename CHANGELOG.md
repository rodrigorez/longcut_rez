# Changelog — LongCut Rez

> **Projeto original:** [SamuelZ12/longcut](https://github.com/SamuelZ12/longcut) (AGPL-3.0)
> **Fork:** [rodrigorez/longcut_rez](https://github.com/rodrigorez/longcut_rez)

---

## [0.0.6] — 2026-06-05 — Fase 0 Concluída

### Adicionado
- `SPEC.md`: Seção **Fase 0 — Análise Técnica com Skills**
  - Status de saúde do projeto (ESLint 0 erros, TS 0 erros)
  - Stack tecnológica atual (Next 15.5.7, React 19.1.2, Tailwind v4, Zod 4)
  - Análise por skill (React best practices, Clean Code, Tailwind v4, Lint)
  - Recomendações técnicas carry-over para fases futuras

### Commits
| SHA | Descrição |
|-----|-----------|
| `a48e190` | docs: add Phase 0 technical analysis with skills review |

---

## [0.0.5] — 2026-06-05 — Reverse Engineering

### Adicionado
- `ORIGINAL_ARCHITECTURE.md` (~920 linhas): Engenharia reversa completa
  - Provider system (interface, config, registry, fallback, 3 adapters)
  - API routes (27 rotas, security middleware, CSRF, rate limiting)
  - Component architecture (hierarquia, playlist, playback)
  - Database schema (13 tabelas, 5 views, 12 RPCs, RLS)
  - AI processing pipeline (chunking, topic generation, quote matching)
  - Type system completo + conventions

### Modificado
- `SPEC.md`: Link para `ORIGINAL_ARCHITECTURE.md` no cabeçalho

### Commits
| SHA | Descrição |
|-----|-----------|
| `4ee71c4` | docs: add complete reverse engineering of original architecture |

---

## [0.0.4] — 2026-06-05 — README Cleanup

### Modificado
- `README.md`: Removida linha "Código original preservado em: .orig/"

### Commits
| SHA | Descrição |
|-----|-----------|
| `4a5acf2` | docs: remove .orig reference from README |

---

## [0.0.3] — 2026-06-05 — README Rewrite

### Modificado
- `README.md`: Reescrito para identidade visual do fork + instruções de setup + atribuição original

### Commits
| SHA | Descrição |
|-----|-----------|
| `cfe58ef` | docs: rewrite README to reflect fork identity |

---

## [0.0.2] — 2026-06-05 — SPEC + .orig

### Adicionado
- `.orig/` (253 arquivos): Cópia fiel do fork original (gitignorado, local-only)
- `SPEC.md`: Especificação técnica com roadmap de 8 fases
  - Fase 0: YouTube Pinned Comment
  - Fase 1: DeepSeek Provider
  - Fase 2: Otimizar Chunking (Gemini 1M)
  - Fase 3: Screenshot nas Notas (thumbnail YouTube)
  - Fase 4: Export .md/.txt
  - Fase 5: Export Dialog + Download
  - Fase 6: Google Drive Upload
  - Fase 7: Setup Final

### Commits
| SHA | Descrição |
|-----|-----------|
| `233f961` | docs: update SPEC.md with .orig structure and refined feature specs |
| `ff98c31` | chore: initial fork setup - rodrigorez/longcut_rez |

---

## [0.0.1] — 2026-06-05 — Fork Inicial

### Ações
- Clone de `SamuelZ12/longcut` para `rodrigorez/longcut_rez`
- Remote alterado para `https://github.com/rodrigorez/longcut_rez.git`
- Branch `main` preservada

---

## Legenda

| Marcação | Significado |
|----------|-------------|
| `[0.0.x]` | Pré-lançamento — fases de implementação |
| **Adicionado** | Novos arquivos |
| **Modificado** | Arquivos existentes alterados (nunca do `.orig/`) |
| **Removido** | Arquivos deletados |
| **Commits** | Referências SHA associadas |
