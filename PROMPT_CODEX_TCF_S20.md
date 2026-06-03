# Prompt para Claude Codex — TapControl Field S20.0

## Contexto do projeto

O TapControl Field (TCF) é uma SPA HTML/JS single-file hospedada no GitHub Pages. Não usa frameworks — é HTML puro com CSS vars e JS vanilla. O backend é um Google Apps Script (GAS). O arquivo a modificar é `TCF_S19_1.html` (3465 linhas).

Versão atual: `S19.1`. A versão resultante deve ser `S20.0`.

---

## LISTA DE MUDANÇAS — implementar todas na ordem abaixo

---

### MUDANÇA 1 — Branding: remover resquícios de "FieldTap" da interface visível

**Arquivo:** `TCF_S19_1.html`

**O que mudar:**

1. Na tela de login (`scr-t1`), o elemento `.pl-name` exibe `"Tap"` e `.pl-sub` exibe `"Engetap · S13.1"`. Mude para:
   - `.pl-badge` → `"FIELD"`  *(já correto, manter)*
   - `.pl-name` → `"TapControl"`
   - `.pl-sub` → `"Engetap · S20.0"` *(atualizar versão junto)*

2. Atualizar a constante de versão no JS:
   ```js
   const APP_VERSION = 'S19.1-...';
   ```
   → `const APP_VERSION = 'S20.0';`

3. No título HTML (`<title>`): já está `"TapControl Field — S19.1 — Engetap"`. Atualizar para `"TapControl Field — S20.0 — Engetap"`.

4. **Funções internas com "FieldTap" no nome NÃO devem ser renomeadas** — são IDs de localStorage e chaves de API GAS que, se mudados, quebram dados existentes. Apenas texto visível ao usuário deve mudar.

---

### MUDANÇA 2 — Faixa do manômetro: exibição simplificada na tela de revisão (T8/scr-revisao)

**Arquivo:** `TCF_S19_1.html`

**Situação atual:** na tela de resumo/lista de itens da OS (scr-t8 ou onde os itens são listados com tag "Qtd"), a faixa de escala aparece duplicada: uma vez no nome do item e outra como subtexto separado abaixo da quantidade.

**O que fazer:** manter apenas uma exibição da faixa — após a quantidade, no mesmo subtexto. Formato desejado:

```
Manômetro Seco 63 mm · 1/4" BSP · Angular · Pressure
Qtd: 1 · 0-28 kgf/cm²
```

Isso significa: remover a faixa que aparece embutida no `hdr-title` / `nome_item` quando for redundante com o subtexto de quantidade. Na prática, a lógica de geração do `desc_curta` já inclui a escala — então o subtexto `Qtd: X · ESCALA` é suficiente. Garantir que a escala não apareça duas vezes no mesmo card de item.

**Localizar** a função que renderiza os cards de itens na revisão (provavelmente `showRevision()` ou `renderRevisionCard()`) e ajustar o template HTML para seguir esse padrão.

---

### MUDANÇA 3 — T6b (Conexões de Montagem): salvar ao pressionar o botão "‹" (voltar)

**Arquivo:** `TCF_S19_1.html`

**Bug atual:** quando o usuário volta da tela T6b usando o botão `‹` do header, a função `goBack('t6b')` limpa `SES.conexoesMontagem = []`, descartando as conexões já adicionadas.

**Linha problemática:**
```js
function goBack(from) {
  if (!SES.history.length) return;
  if (from === 't6b') SES.conexoesMontagem = []; // ← REMOVER esta linha
  ...
}
```

**O que fazer:**
1. Remover a linha `if (from === 't6b') SES.conexoesMontagem = [];` da função `goBack`.
2. O botão `‹` do header de T6b (`goBack('t6b')`) deve chamar `t6bConfirmarConexoes()` em vez de `goBack('t6b')` diretamente — ou seja, tratar o back como um "confirmar e voltar" silencioso.

**Implementação sugerida:** alterar o `onclick` do hdr-back em `scr-t6b`:
```html
<!-- DE: -->
<div class="hdr-back" onclick="goBack('t6b')">‹</div>
<!-- PARA: -->
<div class="hdr-back" onclick="t6bConfirmarConexoes()">‹</div>
```

E garantir que `t6bConfirmarConexoes()` simplesmente avance sem exigir conexões (conexões são opcionais — o usuário pode ter zerado de propósito). A função já trata lista vazia, então isso deve funcionar sem outras mudanças.

---

### MUDANÇA 4 — Remover tela T6c (Itens Adicionais) do fluxo de acessórios

**Arquivo:** `TCF_S19_1.html`

**Contexto:** a tela T6c pergunta itens complementares como Lacre de Segurança, Certificado de Calibração, Suporte/Sela, Fita Veda-rosca, Manopla de bloqueio. Esses são insumos do processo — quando calibração é necessária, lacre e certificado já estão implícitos no processo NR-13. Os demais são irrelevantes para o levantamento.

**O que fazer:**
1. **Remover** a tela `scr-t6c` do fluxo de navegação — ela não deve mais aparecer para nenhum tipo de acessório.
2. No fluxo pós-T6b, pular direto para T6eq (se necessário pela primeira vez no equipamento) e depois T7, **sem passar por T6c**.
3. A função `goToT6c()` e `renderT6c()` podem ser mantidas no código como dead code comentado, mas não devem ser chamadas.
4. Atualizar a função que atualmente chama `goToT6c` após T6b/T6eq:

```js
// Fluxo atual: T6b → goToT6eq_seNecessario(goToT6c) → T6c → T7
// Fluxo novo:  T6b → goToT6eq_seNecessario(goToT7)   → T7
```

Localizar onde `goToT6c` é passado como callback para `goToT6eq_seNecessario` e substituir por `goToT7`.

---

### MUDANÇA 5 — T6eq (Itens do Equipamento) mover para dentro de T4 como botão separado

**Arquivo:** `TCF_S19_1.html`

**Contexto atual:** T6eq aparece automaticamente na primeira vez que um acessório de um equipamento chega até T6b/T6c, perguntando "Placa Indelével" e "Placa de Tag/Categoria".

**Novo comportamento desejado:** T6eq deve ser acessível como um botão dedicado na tela T4, no mesmo nível dos botões de tipo de acessório (Manômetro, Válvula Seg., etc.). O técnico acessa quando quiser, não obrigatoriamente no meio do fluxo de um item.

**Implementação:**

1. **Adicionar botão "Itens do Equipamento"** na tela T4 (`scr-t4`), após o `type-grid`, com destaque visual diferente (usar `.btn-ghost` ou um card de largura total):

```html
<!-- Adicionar após o div.type-grid em scr-t4 -->
<div style="margin-top:16px">
  <button class="btn btn-ghost" style="width:100%" onclick="abrirItensEquipamento()">
    🏷️ Itens do Equipamento
  </button>
</div>
```

2. **Criar função `abrirItensEquipamento()`** que chama `renderT6eq()` diretamente, definindo o callback de retorno para `renderT4()` (voltar para T4 após confirmar):

```js
function abrirItensEquipamento() {
  SES._t6eq_callback = function() { renderT4(); };
  renderT6eq();
}
```

3. **Remover** a chamada automática de T6eq do fluxo pós-T6b. Ou seja, `goToT6eq_seNecessario` não deve mais ser chamada após T6b — substituir pelo fluxo direto para T7:

```js
// No lugar de: goToT6eq_seNecessario(goToT7);
// Usar: goToT7();
```

4. T6eq continua registrando que já foi respondido por equipamento (flag `SES._t6eq_feito`) para evitar duplicidade se o técnico clicar novamente — mas deve permitir re-entrada para edição. Ao re-entrar em T6eq para um equipamento já respondido, pré-selecionar os itens já marcados.

---

### MUDANÇA 6 — T4: adicionar seção "Serviços Necessários" com checkboxes

**Arquivo:** `TCF_S19_1.html`

**Objetivo:** antes de escolher o tipo de acessório para adicionar, o técnico deve poder indicar quais **serviços** precisam ser realizados no equipamento (sem necessariamente cadastrar um novo acessório). Isso serve para registrar inspeções de acessórios já existentes no equipamento.

**Adicionar no body de `scr-t4`**, entre o título e o `type-grid`:

```html
<p class="sec-label" style="margin-top:0">Serviços necessários</p>
<div id="t4-servicos-grid" style="display:flex;flex-direction:column;gap:8px;margin-bottom:20px"></div>
```

**Lista de serviços disponíveis** (renderizar como checkboxes toggleáveis):

| ID | Label | Aplicável a |
|----|-------|-------------|
| `calib_manometro` | Calibração de Manômetro existente | Sempre |
| `insp_valv_seguranca` | Inspeção de Válvula de Segurança | Sempre |
| `insp_valv_quebra_vacuo` | Inspeção de Válvula Quebra Vácuo | Sempre |
| `insp_valv_alivio` | Inspeção de Válvula de Alívio | Sempre |
| `insp_valv_alivio_qv` | Inspeção de Válvula Alívio/Quebra Vácuo | Sempre |
| `inst_purgador` | Instalação de Purgador | Sempre |

**Estado:** armazenar em `SES.servicosSelecionados` (array de IDs). Inicializar no `renderT4()`. Persistir no draft/levantamento junto com os itens (adicionar campo `servicos_necessarios` no payload de envio).

**Renderização de cada serviço** (card toggleável, similar ao T6eq):
```html
<div class="srv-opt [sel]" onclick="toggleServico('ID')">
  <div style="flex:1">
    <div style="font-weight:600">LABEL</div>
  </div>
  <div class="srv-check">[○ ou ✓]</div>
</div>
```

Usar CSS similar aos itens de T6eq: borda highlight quando selecionado.

---

### MUDANÇA 7 — T3: solicitar foto geral do equipamento ao selecioná-lo

**Arquivo:** `TCF_S19_1.html`

**Trigger:** ao selecionar um equipamento existente na tela T3 (clique no card do equipamento) OU ao cadastrar novo equipamento (após confirmar dados do novo equipamento), exibir um **modal/dialog** para captura de foto geral do equipamento.

**Comportamento do modal:**

- Título: "📷 Foto do Equipamento"
- Subtítulo: "Tire uma foto que mostre o local de instalação do equipamento e seus acessórios."
- Botões:
  - `[📷 Tirar foto]` → aciona `<input type="file" accept="image/*" capture="environment">`
  - Preview da foto selecionada (thumbnail 100%)
  - `[Pular →]` → avança sem foto
  - `[Confirmar →]` → só habilitado quando foto está selecionada; avança com foto

**Armazenamento:** `SES.fotoEquipamento[id_equip]` = base64 string (ou null se pulado). Converter a imagem para base64 com FileReader antes de armazenar. **Limitar tamanho:** se a imagem tiver mais de 800px em qualquer dimensão, redimensionar via Canvas antes de converter para base64 (manter aspect ratio, qualidade JPEG 0.7).

**Envio:** incluir no payload de `enviarLevantamento` um campo `fotos_equipamentos` (objeto `{ id_equip: base64_string }`). No GAS, salvar em coluna `foto_base64` da aba EQUIPAMENTOS (ou criar coluna se não existir — o TCF não precisa se preocupar com isso, só envia).

---

### MUDANÇA 8 — T4: solicitar foto do ponto de instalação ao clicar em tipo de acessório

**Arquivo:** `TCF_S19_1.html`

**Trigger:** quando o técnico toca em qualquer tipo de acessório na tela T4 (Manômetro, Válvula Seg., Purgador, DCBI, Conexão, Outro), antes de avançar para T5, exibir modal de foto do ponto de instalação.

**Comportamento:** igual ao modal da MUDANÇA 7, mas com:
- Título: "📷 Ponto de Instalação"  
- Subtítulo: "Foto do ponto onde será instalado o {TIPO_ACESSORIO} no {TAG_EQUIP}."

**Armazenamento:** `SES.fotoPontoInstalacao[id_equip + '_' + tipo]` = base64 (ou null).

**Envio:** incluir campo `fotos_pontos_instalacao` no payload de envio do levantamento.

**IMPORTANTE:** o modal aparece toda vez que o usuário escolhe um tipo — não apenas na primeira vez. Isso porque pode haver múltiplos manômetros por equipamento.

---

### MUDANÇA 9 — T5 (DCBI): renomear botão "Não encontrei no catálogo" para "Lançar manualmente" e expandir o formulário manual

**Arquivo:** `TCF_S19_1.html`

**Situação atual:** na tela T5 para DCBIs, existe um card ao final com texto "Não encontrei no catálogo / Registrar manualmente" que leva para um formulário simples.

**O que fazer:**

1. Renomear o texto do card:
   - Título: "Lançar manualmente" (era "Não encontrei no catálogo")
   - Subtítulo: "Preencher dados completos do DCBI" (era "Registrar manualmente")

2. Ao clicar em "Lançar manualmente" para DCBI, ir para uma tela `scr-t5manual-dcbi` nova (similar à `scr-t5manual-manom`) com os campos:

   | Campo | Tipo | Placeholder |
   |-------|------|-------------|
   | Tipo de válvula principal | select | Válvula tripartida / Gaveta / Esfera / Outro |
   | Bitola da válvula | select | 1/4" / 3/8" / 1/2" / 3/4" / 1" / 1.1/4" / 1.1/2" / 2" |
   | Conexões necessárias | textarea | Ex: 2x niple 1/2" inox, 1x bucha redutora |
   | Lacre de segurança | checkbox (sim/não) | — |
   | Plaqueta de aviso | checkbox (sim/não) | — |
   | Observações | textarea | (opcional) |

3. Botão "Confirmar →" monta o objeto de item com `tipo: 'dcbi'`, `lancamento_manual: true`, e os campos acima, e avança para T6b normalmente.

---

### MUDANÇA 10 — GAS: atualizar nomenclatura nos textos de log/comentários visíveis

**Arquivo:** `GAS_Code_S18_0.js`

**O que mudar:**
- Linha 14: `// 1. Abra a planilha Engetap_FieldTap_TapParts_DB no Google Sheets` → `// 1. Abra a planilha Engetap_TapControl_DB no Google Sheets`
- Linha 296: `app: 'Engetap FieldTap TapParts GAS'` → `app: 'Engetap TapControl GAS'`
- Linha 5: comentário de cabeçalho `// ENGETAP — TapControl Field + TapControl Manager` → já correto, manter.

**NÃO renomear:**
- Nomes de funções como `salvarDraftFieldTap`, `getDraftFieldTap`, etc. — são chamadas pelo TCF pelo nome exato via `gasPost()`.
- Strings como `'aguardando_tapparts'` nos campos de status — são valores de dados no Sheets.
- Qualquer chave de coluna nas abas do Sheets.

---

## Notas gerais de implementação

### Estrutura do arquivo
- O arquivo é um HTML único. Todo CSS está em `<style>`, todo JS em `<script>`, sem módulos externos.
- Telas são `<div class="screen hidden" id="scr-XXX">`. A função `goTo(id)` mostra uma tela e registra no `SES.history`.
- O estado global é o objeto `SES`. Adicionar novos campos ao `SES` no topo do script junto com os campos existentes.

### Padrão de modal/dialog
Para os modais de foto (MUDANÇAS 7 e 8), seguir o mesmo padrão do dialog de conexão em `t6bAddConexao()`:
- Criar div overlay com `position:fixed;inset:0;background:rgba(0,0,0,.5);z-index:9999`
- Panel em `align-items:flex-end` (bottom sheet)
- Remover com `.remove()` ao fechar

### Padrão de cards toggleáveis
Para os serviços (MUDANÇA 6), seguir o mesmo estilo CSS dos cards em T6eq:
```css
/* já existe no arquivo como .kit-opt e .kit-opt.sel */
```
Reutilizar essas classes.

### Draft e payload
O draft (`salvarDraftFieldTap`) e o envio (`enviarLevantamento`) têm estrutura definida no GAS. Ao adicionar campos novos (`servicos_necessarios`, `fotos_equipamentos`, `fotos_pontos_instalacao`), garantir que estejam no payload enviado ao GAS. O GAS atual pode ignorar campos desconhecidos — isso é esperado; a persistência dos novos campos será implementada no GAS em outro momento.

### Versão
Ao final de todas as mudanças, confirmar que `APP_VERSION = 'S20.0'` e `<title>TapControl Field — S20.0 — Engetap</title>`.

---

## Ordem de execução sugerida para o Codex

1. MUDANÇA 1 (branding/versão) — trivial, 3 linhas
2. MUDANÇA 3 (bug T6b voltar) — cirúrgica, 2 linhas
3. MUDANÇA 4 (remover T6c) — roteamento, ~5 linhas
4. MUDANÇA 5 (T6eq em T4) — adicionar botão + função, ~15 linhas
5. MUDANÇA 2 (faixa duplicada) — localizar função de revisão e ajustar template
6. MUDANÇA 6 (serviços em T4) — novo HTML + JS de toggle + payload
7. MUDANÇA 9 (DCBI manual) — nova tela HTML + função de render/confirm
8. MUDANÇA 7 (foto equipamento) — modal + FileReader + Canvas resize
9. MUDANÇA 8 (foto ponto instalação) — modal similar ao 7
10. MUDANÇA 10 (GAS comentários) — 2 linhas no GAS

---

## Entregáveis esperados

- `TCF_S20_0.html` — arquivo completo modificado
- `GAS_Code_S20_0.js` — arquivo GAS com os ajustes de comentários
