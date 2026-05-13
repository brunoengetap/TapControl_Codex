# TapControl_Codex
Todos os arquivos

## Changelog

### 2026-05-12 — Correção Fase 2 formal
- Bloqueada alteração direta para `em_campo_f2` no TapParts (agora exige envio formal da OS).
- Envio Fase 2 com técnico obrigatório e validações adicionais no modal.
- Validação backend no GAS para recusar transição direta para `em_campo_f2` sem autorização formal.
- FieldTap ajustado para não aceitar fallback indevido de `pronto_campo` sem OS formal em Fase 2 + técnico vinculado.
- Adicionada função de diagnóstico de inconsistências da Fase 2 no TapParts (`diagnosticarInconsistenciasFase2()`).


## Changelog S16/S15
- FieldTap S16: OS enviadas/concluídas permanecem visíveis por 3 dias, opacas, com cancelamento.
- TapParts S15: reabertura/reenvio administrativo ao FieldTap.
- GAS S15: endpoints de cancelamento, reabertura e reenvio.


### 2026-05-13 — S17 / GAS S16 — Correções cirúrgicas
1. Cancelamento de envio FieldTap preserva estado do levantamento para correção.
2. Reenvio após cancelamento não duplica levantamento/OS.
3. Fase 2 enviada pelo TapParts aparece imediatamente no FieldTap via Atualizar.
4. Correção do tipo de equipamento em modal de edição.
5. Dashboard não marca OS em Fase 2 como 100% concluída.
6. Proteção contra duplicidade de OS por número.
7. Normalização de status/fase FieldTap/TapParts/GAS.
