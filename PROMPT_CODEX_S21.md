# PROMPT CODEX — TapControl S21 (TCF + TCM + GAS)

> **PRIMEIRA AÇÃO OBRIGATÓRIA:** Antes de qualquer modificação, leia este arquivo no repositório para garantir que está trabalhando com as instruções mais atuais:
> ```
> cat PROMPT_CODEX_S21.md
> ```
> Arquivos a modificar nesta sessão: `TCF_S20_0.html` → salvar como `TCF_S21_0.html` · `TCM_S19_1.html` → salvar como `TCM_S21_0.html` · `GAS_Code_S20_0.js` → salvar como `GAS_Code_S21_0.js`

---

## CONTEXTO DO SISTEMA

**TapControl** é um sistema de inspeção e pipeline de acessórios NR-13 com três camadas:

- **TCF (TapControl Field)** — SPA HTML para inspetores em campo (mobile). Arquivo atual: `TCF_S20_0.html`
- **TCM (TapControl Manager)** — SPA HTML para administração/escritório (desktop). Arquivo atual: `TCM_S19_1.html`
- **GAS (Google Apps Script)** — Backend com Google Sheets como banco. Arquivo atual: `GAS_Code_S20_0.js`

**Arquitetura de dados relevante:**
- Aba `ITENS` no GAS tem colunas: `id_item`, `tipo_acessorio`, `descricao_curta`, `quantidade`, `conexoes_montagem_json`, `itens_adicionais_json`, `payload_item_json`, `status`, `id_os`, `tag_equipamento`, `id_equipamento`, etc.
- Aba `LEVANTAMENTOS` armazena metadados do envio. Fotos são enviadas pelo TCF dentro do payload do levantamento (`fotos_equipamentos`, `fotos_pontos_instalacao`), mas **o GAS atual não persiste fotos em colunas próprias** — elas ficam apenas dentro de `payload_item_json` ou no payload do levantamento.
- IDs seguem padrão: `OS-YYYY-XXX` ou número como `2062026`, equipamentos `EQ-XXX`, itens `ACE-XXXX-XXX`.

---

## BLOCO 1 — CORREÇÕES NO TCF (`TCF_S20_0.html`)

### 1.1 — Bug: Botão "Voltar" na tela T6b vai para T7 em vez de salvar e voltar para T6

**Problema identificado no código (linha ~914):**
```html
<div class="hdr-back" onclick="t6bConfirmarConexoes()">‹</div>
```
O `onclick` do botão voltar (‹) na tela T6b chama `t6bConfirmarConexoes()`, que internamente chama `goToT7()` — ou seja, avança para T7. Isso está errado: o usuário esperava voltar para a tela anterior (T6), mas o sistema avança.

**O que deve acontecer ao clicar em "voltar" (‹) no T6b:**
1. Salvar o estado atual das conexões em `SES.conexoesMontagem` (equivalente ao que `t6bConfirmarConexoes` já faz em termos de persistência no SES).
2. **Não** avançar para T7.
3. Navegar de volta para a tela anterior **usando a pilha de histórico (`SES.history`)**, igual a `goBack('t6b')`.

**Correção exata:**
- Criar uma nova função `t6bSalvarEVoltar()`:
```javascript
function t6bSalvarEVoltar() {
  // Salva conexões no SES sem avançar
  // SES.conexoesMontagem já está atualizado pelo renderT6bLista, não precisa re-parsear
  goBack('t6b');
}
```
- Substituir o `onclick` do `hdr-back` no HTML de T6b:
```html
<!-- ANTES: -->
<div class="hdr-back" onclick="t6bConfirmarConexoes()">‹</div>
<!-- DEPOIS: -->
<div class="hdr-back" onclick="t6bSalvarEVoltar()">‹</div>
```

**Atenção:** `goBack('t6b')` usa a função `goBack(from)` existente que faz `SES.history.pop()` e navega para a tela anterior na pilha. Não alterar `t6bConfirmarConexoes()` — ela continua sendo chamada pelo botão "Confirmar →".

---

### 1.2 — Feature: Adicionar botão "Escolher da galeria" no modal de foto

**Problema:** O modal de foto (`abrirModalFoto`) tem apenas um `<input type="file" accept="image/*" capture="environment">` que força a abertura da câmera. O usuário precisa também conseguir escolher uma foto já existente na galeria.

**Localização no código (linha ~1431):**
```html
<input type="file" id="dlg-foto-input" accept="image/*" capture="environment" style="display:none">
<button class="btn btn-ghost" id="dlg-foto-btn" style="width:100%;margin-bottom:12px">📷 Tirar foto</button>
```

**Correção exata — substituir o trecho acima por dois inputs e dois botões:**
```html
<input type="file" id="dlg-foto-input-camera" accept="image/*" capture="environment" style="display:none">
<input type="file" id="dlg-foto-input-galeria" accept="image/*" style="display:none">
<div style="display:flex;gap:8px;margin-bottom:12px">
  <button class="btn btn-ghost" id="dlg-foto-btn-camera" style="flex:1">📷 Tirar foto</button>
  <button class="btn btn-ghost" id="dlg-foto-btn-galeria" style="flex:1">🖼 Galeria</button>
</div>
```

**Atualizar o JavaScript logo abaixo (onde `input` e `btnFoto` são referenciados):**
- Remover as referências a `dlg-foto-input` e `dlg-foto-btn` (singular).
- Adicionar:
```javascript
const inputCamera  = document.getElementById('dlg-foto-input-camera');
const inputGaleria = document.getElementById('dlg-foto-input-galeria');
const btnCamera    = document.getElementById('dlg-foto-btn-camera');
const btnGaleria   = document.getElementById('dlg-foto-btn-galeria');

const processarArquivo = async (file, btnRef) => {
  if (!file) return;
  try {
    btnRef.textContent = 'Processando…';
    fotoBase64 = await resizeImageFileToBase64(file);
    document.getElementById('dlg-foto-preview').src = fotoBase64;
    document.getElementById('dlg-foto-preview-wrap').style.display = '';
    btnConfirm.disabled = false;
    btnConfirm.style.opacity = '1';
    btnCamera.textContent = '📷 Trocar foto';
    btnGaleria.textContent = '🖼 Trocar da galeria';
  } catch(e) {
    btnRef.textContent = btnRef === btnCamera ? '📷 Tirar foto' : '🖼 Galeria';
    showToast('Não foi possível processar a foto', 'err');
  }
};

btnCamera.onclick  = () => inputCamera.click();
btnGaleria.onclick = () => inputGaleria.click();
inputCamera.onchange  = async () => processarArquivo(inputCamera.files?.[0], btnCamera);
inputGaleria.onchange = async () => processarArquivo(inputGaleria.files?.[0], btnGaleria);
```

**Importante:** Não alterar `resizeImageFileToBase64`, `dlg-foto-preview`, `dlg-foto-pular` nem `dlg-foto-confirmar` — apenas o par input+botão.

---

## BLOCO 2 — CORREÇÕES NO TCM (`TCM_S19_1.html`)

### 2.1 — Renomear todas as referências visuais de "TapParts" na tela de OS

**Problema:** Na tela "Ordens de Serviço" existem referências ao nome antigo "TapParts" que devem ser substituídas pelo nome atual. Há dois tipos de ocorrências:

**Tipo A — Textos visíveis para o usuário (UI labels):** devem ser substituídos.
**Tipo B — Strings internas de lógica/fluxo** como `origem: 'TapParts'`, `'aguardando_tapparts'`, `'Aguardando TapParts'` (valor de `o.status` do GAS), `'TapParts S4'` em `observacao`: **NÃO alterar** — são chaves de dados que vêm do banco e devem permanecer intactas para não quebrar a lógica.

**Substituições visuais obrigatórias (apenas strings de texto renderizado na UI):**

| Localização | Texto atual | Texto novo |
|---|---|---|
| Linha ~1869 — botão condicional na lista de OS | `📲 TapControl Field` | manter (já correto) |
| Linha ~4673 — `origem: 'TapParts'` em `salvarOS` | Não alterar — é valor interno |  |
| Linha ~4675 — `origem: 'TapParts'` em `avancarFaseOS` | Não alterar — é valor interno |  |
| Linha ~4277 — `'Marcado como Em Calibração pelo TapParts S4'` | Substituir por `'Marcado como Em Calibração pelo TapControl Manager'` |  |
| Linha ~2045 — `origem: 'TapParts'` em `desvincularEquip` | Substituir por `origem: 'TapControl Manager'` |  |
| Linha ~2013 — `origem: 'TapParts'` em `vincularEquipamento` | Substituir por `origem: 'TapControl Manager'` |  |

**Busca completa:** Executar `grep -n "TapParts" TCM_S19_1.html` e avaliar linha por linha:
- Se a string é exibida ao usuário ou gravada como `origem` de auditoria (responsavel) → substituir por `TapControl Manager`.
- Se é um valor de `status` ou `fase_atual` que vem do GAS como `'Aguardando TapParts'` → **não alterar** (é chave do banco).

---

### 2.2 — Corrigir lentidão/falha ao vincular equipamento à OS

**Problema:** `vincularEquipamento()` (linha ~2000) chama `gasPost('vincularEquipamentoOS', {...})` e depois `await syncGAS(false)`. O `syncGAS` faz um re-fetch completo de todos os dados do GAS, o que é lento e causa a impressão de travamento. Às vezes o usuário sai e volta e só aí a UI atualiza.

**Causa raiz:** Após o `gasPost` bem-sucedido, o código faz:
```javascript
eq.id_os = _osEquipContext;
await syncGAS(false);  // <-- isso é lento, fetch completo
```
O `syncGAS(false)` leva vários segundos. Enquanto aguarda, a UI não dá feedback visual.

**Correção em `vincularEquipamento()`:**

1. **Mostrar loading imediato** antes do `gasPost`:
```javascript
async function vincularEquipamento() {
  const eqId = document.getElementById('oseq-sel-equip').value;
  if (!eqId) { showToast('Selecione um equipamento', 'err'); return; }
  const eq = DB.equipamentos.find(e => e.id_equipamento === eqId);
  if (!eq) return;

  // NOVO: feedback visual imediato
  const btnVincular = document.querySelector('[onclick="vincularEquipamento()"]');
  const txtOriginal = btnVincular ? btnVincular.textContent : '';
  if (btnVincular) { btnVincular.disabled = true; btnVincular.textContent = 'Vinculando…'; }

  if (GAS_URL) {
    try {
      const r = await gasPost('vincularEquipamentoOS', {
        id_equipamento: eqId,
        id_os: _osEquipContext,
        responsavel: 'adm',
        origem: 'TapControl Manager'
      });
      if (!r || r.status !== 'ok') throw new Error(r?.mensagem || r?.codigo || 'Falha GAS');
      eq.id_os = _osEquipContext;
      // NOVO: atualizar UI imediatamente, sem aguardar syncGAS
      renderOSEquipList(_osEquipContext);
      renderOS();
      renderEquipamentos();
      renderDashboard();
      showToast('Equipamento vinculado ✓');
      // syncGAS em background (sem await)
      syncGAS(false).catch(() => {});
    } catch(e) {
      showToast('Erro ao vincular equipamento: ' + e.message, 'err');
    }
  } else {
    eq.id_os = _osEquipContext;
    renderOSEquipList(_osEquipContext);
    renderOS();
    renderEquipamentos();
    renderDashboard();
    showToast('Equipamento vinculado ✓');
  }

  if (btnVincular) { btnVincular.disabled = false; btnVincular.textContent = txtOriginal; }
}
```

**Mesma correção em `desvincularEquip()`:** trocar `await syncGAS(false)` por `syncGAS(false).catch(() => {})` e mover o `renderOSEquipList` + `renderOS` para imediatamente após a atualização local do `eq.id_os`.

---

### 2.3 — Adicionar filtro por tipo de acessório no Pipeline

**Objetivo:** Adicionar um `<select>` de filtro por `tipo_acessorio` na barra de filtros do Pipeline, posicionado após o filtro de calibração e antes do filtro de clientes.

**Tipos possíveis** (extraídos dos dados reais):
`Manômetro`, `Válvula de Segurança`, `Purgador`, `DCBI`, `Conexão`, `Item personalizado`

**Passo 1 — Adicionar o select no HTML da pipeline-filter-bar (linha ~828, após `pipe-calib-filter`):**
```html
<select class="filter-select" id="pipe-tipo-filter" onchange="renderPipeline()" style="min-width:170px">
  <option value="">Todos os tipos</option>
  <option value="Manômetro">Manômetros</option>
  <option value="Válvula de Segurança">Válvulas de Segurança</option>
  <option value="Purgador">Purgadores</option>
  <option value="DCBI">DCBIs</option>
  <option value="Conexão">Conexões</option>
  <option value="Item personalizado">Itens personalizados</option>
</select>
```

**Passo 2 — Adicionar leitura e filtro em `renderPipeline()` (linha ~3667):**
Após a linha que lê `cliFilter`, adicionar:
```javascript
const tipoFilter = document.getElementById('pipe-tipo-filter')?.value || '';
```
E na seção de filtros, após o filtro de `calibFilter`:
```javascript
if (tipoFilter) data = data.filter(i => (i.tipo_acessorio || '') === tipoFilter);
```

---

### 2.4 — Geração de PDF de cotação a partir do Pipeline

**Objetivo:** Adicionar um botão "📄 Gerar PDF de Cotação" no header do Pipeline que gera um PDF com os itens atualmente filtrados na tela, formatado como pedido de cotação para fornecedor.

#### 2.4.1 — Botão no header

No HTML da seção pipeline, dentro de `.table-header` (linha ~834), adicionar ao lado do contador:
```html
<div style="display:flex;gap:8px;align-items:center">
  <span class="th-meta" id="pipe-count">0 itens</span>
  <button class="btn btn-ghost btn-sm" onclick="gerarPDFCotacao()" title="Gerar PDF com itens filtrados para cotação" style="font-size:12px">📄 Gerar PDF de Cotação</button>
</div>
```
(remover o `<span id="pipe-count">` isolado e substituir pelo bloco acima)

#### 2.4.2 — Função `gerarPDFCotacao()`

Esta função deve:
1. Ler os mesmos filtros ativos em `renderPipeline()` e aplicar ao `DB.itens` — **exatamente a mesma lógica de filtro**, para que o PDF reflita o que está na tela.
2. Agrupar itens por `tipo_acessorio` para facilitar leitura do fornecedor.
3. Gerar HTML estilizado e abrir em nova janela com `window.print()`.

```javascript
function gerarPDFCotacao() {
  // Reaplica os mesmos filtros de renderPipeline
  const q          = (document.getElementById('pipe-search')?.value || '').toLowerCase();
  const stFilter   = document.getElementById('pipe-status-filter')?.value || '';
  const osFilter   = document.getElementById('pipe-os-filter')?.value || '';
  const calibFilter= document.getElementById('pipe-calib-filter')?.value || '';
  const cliFilter  = document.getElementById('pipe-cli-filter')?.value || '';
  const tipoFilter = document.getElementById('pipe-tipo-filter')?.value || '';

  let data = DB.itens || [];
  if (q) data = data.filter(i =>
    i.descricao_curta?.toLowerCase().includes(q) ||
    i.tag_equipamento?.toLowerCase().includes(q) ||
    i.cliente?.toLowerCase().includes(q) ||
    i.id_os?.toLowerCase().includes(q) ||
    i.id_item?.toLowerCase().includes(q));
  if (stFilter) data = data.filter(i => i.status === stFilter);
  if (osFilter) data = data.filter(i => i.id_os === osFilter);
  if (cliFilter) {
    data = data.filter(i => {
      if (i.id_cliente && i.id_cliente === cliFilter) return true;
      const os = DB.os.find(o => o.id_os === i.id_os);
      if (os && os.id_cliente === cliFilter) return true;
      const cli = DB.clientes.find(c => c.id_cliente === cliFilter);
      if (cli && i.cliente && i.cliente.toLowerCase().includes(cli.nome.toLowerCase())) return true;
      return false;
    });
  }
  if (calibFilter === 'sim') data = data.filter(i => i.requer_calibracao);
  if (calibFilter === 'nao') data = data.filter(i => !i.requer_calibracao);
  if (tipoFilter) data = data.filter(i => (i.tipo_acessorio || '') === tipoFilter);

  if (!data.length) { showToast('Nenhum item para gerar PDF com os filtros atuais', 'err'); return; }

  // Agrupa por tipo_acessorio
  const grupos = {};
  data.forEach(i => {
    const tipo = i.tipo_acessorio || 'Outros';
    if (!grupos[tipo]) grupos[tipo] = [];
    grupos[tipo].push(i);
  });

  const hoje = new Date().toLocaleDateString('pt-BR');
  const empresa = 'Engetap — Serviços de Inspeção NR-13';

  // Monta título dinâmico baseado nos filtros ativos
  const partesFiltro = [];
  if (tipoFilter) partesFiltro.push(tipoFilter + 's');
  if (osFilter) { const os = DB.os.find(o => o.id_os === osFilter); partesFiltro.push('OS ' + (os?.numero_os || osFilter)); }
  if (cliFilter) { const cli = DB.clientes.find(c => c.id_cliente === cliFilter); if (cli) partesFiltro.push(cli.nome); }
  if (stFilter) partesFiltro.push(STATUS_LABEL[stFilter] || stFilter);
  const tituloFiltro = partesFiltro.length ? ' — ' + partesFiltro.join(' · ') : ' — Todos os itens filtrados';

  let rowsHtml = '';
  Object.entries(grupos).forEach(([tipo, itens]) => {
    rowsHtml += `<tr class="grupo-header"><td colspan="6">${tipo}</td></tr>`;
    itens.forEach((i, idx) => {
      const os = DB.os.find(o => o.id_os === i.id_os);
      const osLabel = os?.numero_os ? 'OS ' + os.numero_os : i.id_os;
      const clienteLabel = os?.cliente || i.cliente || '—';
      const calibLabel = i.requer_calibracao ? '⚠ Calibração NR-13' : '—';
      rowsHtml += `<tr>
        <td style="text-align:center">${idx + 1}</td>
        <td>${i.descricao_curta || '—'}</td>
        <td style="text-align:center">${i.quantidade}</td>
        <td>${i.tag_equipamento || '—'} · ${osLabel}</td>
        <td>${clienteLabel}</td>
        <td>${calibLabel}</td>
      </tr>`;
    });
  });

  const html = `<!DOCTYPE html><html lang="pt-BR"><head>
  <meta charset="UTF-8">
  <title>Cotação TapControl</title>
  <style>
    * { margin:0; padding:0; box-sizing:border-box; }
    body { font-family: Arial, sans-serif; font-size: 12px; color: #111; padding: 24px; }
    .header { border-bottom: 2px solid #1a56db; padding-bottom: 12px; margin-bottom: 16px; display: flex; justify-content: space-between; align-items: flex-end; }
    .header-left h1 { font-size: 18px; color: #1a56db; }
    .header-left p { font-size: 11px; color: #555; margin-top: 2px; }
    .header-right { text-align: right; font-size: 11px; color: #555; }
    .info-box { background: #f0f4ff; border: 1px solid #c3d5ff; border-radius: 6px; padding: 10px 14px; margin-bottom: 16px; font-size: 11px; color: #333; }
    table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
    th { background: #1a56db; color: white; padding: 7px 8px; text-align: left; font-size: 11px; }
    td { padding: 6px 8px; border-bottom: 1px solid #e5e7eb; font-size: 11px; vertical-align: top; }
    tr:nth-child(even) td { background: #f9fafb; }
    tr.grupo-header td { background: #e8f0fe; color: #1a56db; font-weight: 700; font-size: 12px; padding: 8px; border-top: 2px solid #c3d5ff; }
    .footer { margin-top: 24px; border-top: 1px solid #ddd; padding-top: 12px; font-size: 10px; color: #888; display: flex; justify-content: space-between; }
    @media print { body { padding: 10px; } button { display: none; } }
  </style>
  </head><body>
  <div class="header">
    <div class="header-left">
      <h1>Pedido de Cotação${tituloFiltro}</h1>
      <p>${empresa}</p>
    </div>
    <div class="header-right">
      <strong>Data:</strong> ${hoje}<br>
      <strong>Total de itens:</strong> ${data.length}
    </div>
  </div>
  <div class="info-box">
    📋 <strong>Instruções:</strong> Favor cotar os itens abaixo conforme especificações. Indicar preço unitário, prazo de entrega e disponibilidade em estoque. Para itens com calibração NR-13, informar se possuem certificado de calibração rastreável à RBC/Inmetro.<br>
    <strong>Enviar proposta para:</strong> ___________________________ &nbsp;&nbsp; <strong>Validade da proposta:</strong> ___________
  </div>
  <table>
    <thead><tr>
      <th style="width:30px">#</th>
      <th>Descrição / Especificação</th>
      <th style="width:50px;text-align:center">Qtd</th>
      <th style="width:120px">TAG · OS</th>
      <th style="width:130px">Cliente</th>
      <th style="width:100px">Obs.</th>
    </tr></thead>
    <tbody>${rowsHtml}</tbody>
  </table>
  <div style="margin-top:16px;border:1px solid #ddd;border-radius:6px;padding:12px">
    <strong style="font-size:12px">Proposta do fornecedor:</strong>
    <table style="margin-top:8px">
      <thead><tr>
        <th>#</th><th>Item</th><th>Qtd</th><th>Preço Unit. (R$)</th><th>Preço Total (R$)</th><th>Prazo Entrega</th><th>Obs.</th>
      </tr></thead>
      <tbody>${data.map((i,idx) => `<tr><td>${idx+1}</td><td>${i.descricao_curta||'—'}</td><td>${i.quantidade}</td><td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</td><td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</td><td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</td><td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</td></tr>`).join('')}</tbody>
    </table>
  </div>
  <div class="footer">
    <span>Gerado por TapControl Manager · ${hoje}</span>
    <span>Documento de uso interno — Engetap</span>
  </div>
  <script>window.onload = () => window.print();<\/script>
  </body></html>`;

  const w = window.open('', '_blank');
  if (w) { w.document.write(html); w.document.close(); }
  else { showToast('Permita pop-ups para gerar o PDF', 'err'); }
}
```

---

### 2.5 — TCM deve receber e exibir fotos enviadas pelo TCF

**Contexto atual:** O TCF envia `fotos_equipamentos` (object: `{ [id_equipamento]: base64 }`) e `fotos_pontos_instalacao` (object: `{ [id_equipamento + '_' + tipo]: base64 }`) dentro do payload do levantamento. No GAS, esses dados ficam dentro de `payload_item_json` de cada item ou no payload do levantamento, mas **não há colunas dedicadas** para fotos na aba `ITENS` ou `LEVANTAMENTOS`.

**Decisão de arquitetura (sugestão implementada):**
- Adicionar colunas `foto_equipamento_b64` e `foto_ponto_instalacao_b64` na aba `ITENS` do GAS.
- No `enviarLevantamento` do GAS: ao gravar cada item, buscar a foto correspondente no payload do levantamento e salvar nas colunas.
- No TCM: ao renderizar o drawer de detalhe do item, exibir as fotos se existirem.
- No TCM: na tela de levantamentos, exibir miniaturas das fotos por equipamento.

#### 2.5.1 — GAS: adicionar colunas de foto e persistir

Em `inicializarPlanilha()` (seção `_garantirColunas`), adicionar:
```javascript
_garantirColunas(ss, 'ITENS', [
  'foto_equipamento_b64',
  'foto_ponto_instalacao_b64'
]);
```

Em `enviarLevantamento(body)`, dentro do bloco onde se monta `itemBase` para cada item (logo após `payload_item_json`), adicionar:
```javascript
// S21 — fotos enviadas pelo TCF
var fotoEquip  = (body.fotos_equipamentos  && item.id_equipamento)
                 ? (body.fotos_equipamentos[item.id_equipamento]  || '')
                 : '';
var fotoPonto  = (body.fotos_pontos_instalacao && item.id_equipamento && item.tipo)
                 ? (body.fotos_pontos_instalacao[item.id_equipamento + '_' + item.tipo] || '')
                 : '';
```
E incluir no objeto `itemBase`:
```javascript
foto_equipamento_b64:      fotoEquip,
foto_ponto_instalacao_b64: fotoPonto,
```

**Atenção:** O campo `tipo` do item (ex: `'manometro'`, `'valvula'`) deve já existir no objeto `item` do payload. Se não existir, usar `item.tipo_acessorio_key` ou derivar de `item.tipo_acessorio` mapeando para lowercase sem acento:
```javascript
var tipoKey = (item.tipo || item.tipo_acessorio || '')
  .toLowerCase()
  .normalize('NFD').replace(/[\u0300-\u036f]/g, '')
  .replace(/\s+/g,'_')
  .replace(/ã/g,'a').replace(/ç/g,'c'); // fallback simples
var fotoPonto = (body.fotos_pontos_instalacao && item.id_equipamento)
  ? (body.fotos_pontos_instalacao[item.id_equipamento + '_' + tipoKey] || '')
  : '';
```

#### 2.5.2 — GAS: garantir que `getItensByOS` e `getItensPipeline` retornam as colunas de foto

Verificar as funções que lêem a aba `ITENS` e montam o objeto de retorno. Certificar que `foto_equipamento_b64` e `foto_ponto_instalacao_b64` estão incluídas no mapeamento colunas → objeto. **Se o mapeamento usa `cabecalho.reduce` ou `Object.fromEntries` de forma dinâmica, as colunas novas serão incluídas automaticamente.** Caso o mapeamento seja manual/explícito, adicionar os campos.

#### 2.5.3 — TCM: exibir fotos no drawer de detalhe do item

No `openDrawer(itemId)`, dentro do bloco `(() => { let extra = ''; ... })()` (linha ~3948), **após** o bloco de `conexoes_montagem` e **antes** do `return extra`, adicionar:

```javascript
// S21 — Foto do equipamento
if (item.foto_equipamento_b64) {
  extra += `<div class="section-hdr">📷 Foto do Equipamento</div>
  <div style="margin-bottom:10px">
    <img src="${item.foto_equipamento_b64}" alt="Foto do equipamento" 
         style="width:100%;max-height:220px;object-fit:cover;border-radius:10px;border:1px solid var(--border);cursor:pointer"
         onclick="abrirFotoFullscreen('${item.id_item}','equip')">
  </div>`;
}
// S21 — Foto do ponto de instalação
if (item.foto_ponto_instalacao_b64) {
  extra += `<div class="section-hdr">📷 Ponto de Instalação</div>
  <div style="margin-bottom:10px">
    <img src="${item.foto_ponto_instalacao_b64}" alt="Foto do ponto de instalação" 
         style="width:100%;max-height:220px;object-fit:cover;border-radius:10px;border:1px solid var(--border);cursor:pointer"
         onclick="abrirFotoFullscreen('${item.id_item}','ponto')">
  </div>`;
}
```

#### 2.5.4 — TCM: função de fullscreen para foto

Adicionar a função auxiliar:
```javascript
function abrirFotoFullscreen(itemId, tipo) {
  const item = DB.itens.find(i => i.id_item === itemId);
  if (!item) return;
  const src = tipo === 'equip' ? item.foto_equipamento_b64 : item.foto_ponto_instalacao_b64;
  if (!src) return;
  const overlay = document.createElement('div');
  overlay.style.cssText = 'position:fixed;inset:0;background:rgba(0,0,0,.9);z-index:99999;display:flex;align-items:center;justify-content:center;cursor:zoom-out';
  overlay.innerHTML = `<img src="${src}" style="max-width:95vw;max-height:95vh;object-fit:contain;border-radius:8px">`;
  overlay.onclick = () => overlay.remove();
  document.body.appendChild(overlay);
}
```

#### 2.5.5 — TCM: exibir miniaturas de foto na tela de Levantamentos

Na função `renderLevantamentos()`, dentro do card de cada levantamento, ao renderizar os itens do levantamento (bloco onde itens são listados), adicionar indicador visual se o item tiver foto:

Localizar o trecho onde itens do levantamento são renderizados (próximo à linha ~2253) e, no innerHTML de cada item, adicionar badge condicional:
```javascript
${it.foto_equipamento_b64 ? '<span title="Foto do equipamento disponível">📷</span>' : ''}
${it.foto_ponto_instalacao_b64 ? '<span title="Foto do ponto de instalação disponível">🔍</span>' : ''}
```

---

## BLOCO 3 — CORREÇÕES NO GAS (`GAS_Code_S20_0.js`)

### 3.1 — Adicionar colunas de foto (já coberto no item 2.5.1 acima)

Resumo das mudanças no GAS:
1. Em `inicializarPlanilha`: adicionar `_garantirColunas(ss, 'ITENS', ['foto_equipamento_b64','foto_ponto_instalacao_b64'])`.
2. Em `enviarLevantamento`: mapear `fotos_equipamentos` e `fotos_pontos_instalacao` do body para os campos de cada item.
3. Verificar `getItensByOS`, `getItens` e qualquer função que retorne itens para garantir que as novas colunas são incluídas.

### 3.2 — Nenhuma outra mudança no GAS nesta sprint

As demais correções (PDF, filtro, renomeação de origem) são client-side no TCM. O GAS não precisa de modificações adicionais além das fotos.

---

## BLOCO 4 — CHECKLIST FINAL ANTES DE COMMITAR

Execute as verificações abaixo no código gerado:

```bash
# 1. Verificar que T6b back button não chama mais t6bConfirmarConexoes
grep -n "hdr-back.*t6bConfirmar\|hdr-back.*onclick.*t6b" TCF_S21_0.html

# 2. Verificar que modal de foto tem dois inputs (camera e galeria)
grep -n "dlg-foto-input-camera\|dlg-foto-input-galeria\|dlg-foto-btn-galeria" TCF_S21_0.html

# 3. Verificar que filtro de tipo existe no Pipeline
grep -n "pipe-tipo-filter" TCM_S21_0.html

# 4. Verificar que função gerarPDFCotacao existe
grep -n "function gerarPDFCotacao" TCM_S21_0.html

# 5. Verificar que vincularEquipamento não tem mais await syncGAS bloqueante
grep -A5 "function vincularEquipamento" TCM_S21_0.html | grep "await syncGAS"
# O resultado deve ser vazio (não deve mais ter await syncGAS bloqueante)

# 6. Verificar que colunas de foto foram adicionadas no GAS
grep -n "foto_equipamento_b64\|foto_ponto_instalacao_b64" GAS_Code_S21_0.js

# 7. Verificar que "TapParts S4" foi substituído por "TapControl Manager"
grep -n "TapParts S4" TCM_S21_0.html
# Deve retornar vazio

# 8. Verificar que origem TapParts foi substituída nas funções de vincular/desvincular
grep -n "origem.*TapParts" TCM_S21_0.html | grep -v "Aguardando\|status\|fase_atual"
```

---

## NOTAS CRÍTICAS PARA O CODEX

1. **Não remover nem refatorar** funções existentes que não estejam explicitamente mencionadas neste prompt. Alterações fora do escopo descrito podem quebrar funcionalidades em uso.

2. **Manter compatibilidade com o GAS S20.0**: O TCM S21 deve continuar funcionando com o GAS S20.0 enquanto o GAS S21.0 não for deployado. As novas colunas de foto são adicionadas via `_garantirColunas` que é idempotente — seguro de rodar mesmo em planilhas já existentes.

3. **Fotos base64 são grandes**: No PDF de cotação (`gerarPDFCotacao`), **não incluir** as fotos base64 — apenas texto tabular. Fotos ficam apenas no drawer do TCM.

4. **O filtro de tipo no PDF**: A função `gerarPDFCotacao` deve obrigatoriamente ler o filtro `pipe-tipo-filter` para que o usuário consiga gerar um PDF apenas com manômetros, por exemplo.

5. **Nomenclatura dos arquivos de saída**: Salvar como `TCF_S21_0.html`, `TCM_S21_0.html` e `GAS_Code_S21_0.js`. Não sobrescrever os arquivos S20/S19.

6. **String `'Aguardando TapParts'`** em comparações de status (ex: `.includes('Aguardando TapParts')`) vem do campo `o.status` gravado pelo GAS. **Não alterar** essas comparações — elas dependem do valor exato gravado no banco. Se o usuário quiser mudar o label visual, usar mapeamento local no TCM, não alterar a chave.

7. **`syncGAS(false)` em background**: O padrão `syncGAS(false).catch(() => {})` (sem await) permite que a UI atualize imediatamente enquanto o sync acontece em background. Se `syncGAS` não existir com esse nome, adaptar para o nome correto da função de sincronização.
