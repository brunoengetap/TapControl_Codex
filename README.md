# TapControl_Codex
Todos os arquivos

## Changelog

### 2026-05-12 — Correção Fase 2 formal
- Bloqueada alteração direta para `em_campo_f2` no TapParts (agora exige envio formal da OS).
- Envio Fase 2 com técnico obrigatório e validações adicionais no modal.
- Validação backend no GAS para recusar transição direta para `em_campo_f2` sem autorização formal.
- FieldTap ajustado para não aceitar fallback indevido de `pronto_campo` sem OS formal em Fase 2 + técnico vinculado.
- Adicionada função de diagnóstico de inconsistências da Fase 2 no TapParts (`diagnosticarInconsistenciasFase2()`).
