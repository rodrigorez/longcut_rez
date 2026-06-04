# MiniMax M3 Default Model Design

## Context

LongCut currently defaults MiniMax text generation to `MiniMax-M2.7`. MiniMax has released `MiniMax-M3`, and the project should use M3 by default while preserving existing environment-variable overrides for rollback or experimentation.

## Scope

- Change source-controlled MiniMax default model strings from `MiniMax-M2.7` to `MiniMax-M3`.
- Update provider and validation tests that assert MiniMax defaults.
- Update active environment examples and active documentation.
- Do not change the MiniMax adapter API shape, base URL, provider fallback order, request payload structure, or generated build output.
- Do not edit unrelated existing worktree changes such as `package.json`.

## Approach

Use the existing provider configuration and adapter defaults. `AI_DEFAULT_MODEL`, `AI_FAST_MODEL`, `AI_PRO_MODEL`, and `NEXT_PUBLIC_AI_MODEL` remain supported as overrides. This makes rollback possible by setting `AI_DEFAULT_MODEL=MiniMax-M2.7` without code changes.

## Verification

- Run targeted tests for MiniMax adapter, provider config, and validation defaults.
- Run lint if the targeted tests pass and time permits.
