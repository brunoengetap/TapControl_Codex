# PROMPT CODEX — TapControl: Fotos no Drive + Geração de PDF (Sprint Fotos/PDF)

> **Contexto:** Sistema ENGETAP — TapControl = GAS backend + TCM (Manager, app admin) + TCF (Field, app inspetor).  
> Stack atual: **GAS S22.0**, **TCM S22.0**, **TCF S22.0**.  
> Este prompt implementa upload de fotos no Google Drive e geração de PDF de Relatório NR-13 com fotos.  
> A referência de implementação é o **ColectTap** (GAS v1.17, Manager v1.19), que já faz isso com a mesma lógica.

---

## 1. DIAGNÓSTICO PRECISO DO ESTADO ATUAL (S22)

### O que JÁ funciona no S22 (não alterar):

**TCF S22 — captura de fotos completamente funcional:**
- `resizeImageFileToBase64(file, maxDim=800, quality=0.7)` — redimensiona e converte para base64
- `abrirModalFoto({titulo, subtitulo, onConfirm, onSkip})` — modal com câmera + galeria + preview
- `solicitarFotoEquipamento(callback)` — salva em `SES.fotoEquipamento[id_equipamento]`
- `solicitarFotoPontoInstalacao(tipo, callback)` — salva em `SES.fotoPontoInstalacao[id_equipamento + '_' + tipo]`
- Draft e snapshot já incluem `fotos_equipamentos` e `fotos_pontos_instalacao`
- Payload de `enviarLevantamento` já envia `fotos_equipamentos` e `fotos_pontos_instalacao` no body
- Timeout do gasPost já é 60000ms para envio

**GAS S22 — recebe fotos mas as grava errado:**
- `enviarLevantamento(body)` lê `body.fotos_equipamentos` e `body.fotos_pontos_instalacao`
- Atualmente salva o base64 bruto nas colunas `foto_equipamento_b64` / `foto_ponto_instalacao_b64` da aba **ITENS** — **ISSO ESTÁ ERRADO** e precisa ser substituído
- Retorna `{ status: 'ok', id_levantamento, itens_criados }` — sem informação de fotos

**GAS S22 — funções/endpoints que NÃO existem (criar do zero):**
- `salvarFotosDriveTapControl(body)` — não existe
- `getFotoBase64(params)` — não existe
- `getLevantamentosComFotos(params)` — não existe
- Helpers: `getOrCreateDrivePath`, `dataUrlToBlob`, `sanitizeFileName`, `_setColIfExists` — não existem
- Constante `DRIVE_LINK_PUBLICO` — não existe

**GAS S22 — schema:**
- Aba LEVANTAMENTOS: colunas `drive_folder_id`, `drive_folder_url`, `quantidade_fotos_equip`, `quantidade_fotos_ponto`, `fotos_equip_json`, `fotos_ponto_json` — **não existem** ainda
- Aba ITENS: colunas `foto_equipamento_b64` e `foto_ponto_instalacao_b64` — **já existem** (foram o workaround; manter por compatibilidade, mas parar de usar para novos levantamentos)
- Função `_garantirColunas` existe e é idempotente — usar para adicionar colunas

**TCM S22 — o que existe:**
- `gasGet(action, params={})` — aceita objeto params, serializa na querystring ✅ (já correto)
- `gerarPDFCotacao()` — existe mas usa print/CSS (não jsPDF); gera PDF de cotação, **não** o relatório NR-13 com fotos
- View `view-levantamentos` com `renderLevantamentos()` — existe; os cards de levantamento têm botão "👁 Ver itens"
- `syncLevantamentos()` → chama `getLevantamentosByCliente` → popula `DB.levantamentos`

**TCM S22 — o que NÃO existe (criar):**
- jsPDF não está carregado
- `LOGO_ENGETAP_B64` — não existe
- `view-fotos` e nav-item de fotos — não existem
- `gerarPDFLevantamento(idLevantamento)` — não existe
- `_buscarFotosLevantamento(levantamento)` — não existe
- `carregarViewFotos()` / `renderViewFotos()` — não existem
- Barra de progresso de download de fotos — não existe
- Botão PDF nos cards de levantamento — não existe

---

## 2. IMPLEMENTAÇÕES NECESSÁRIAS

### 2.1 GAS — Adicionar ao `inicializarPlanilha` / `_garantirColunas`

No bloco de chamadas `_garantirColunas` existente (após a linha que garante colunas de ITENS), adicionar:

```javascript
_garantirColunas(ss, 'LEVANTAMENTOS', [
  'drive_folder_id', 'drive_folder_url',
  'quantidade_fotos_equip', 'quantidade_fotos_ponto',
  'fotos_equip_json', 'fotos_ponto_json'
]);
```

### 2.2 GAS — Constante de configuração

No topo do arquivo, junto às outras constantes globais:

```javascript
var DRIVE_LINK_PUBLICO = false; // false = acesso restrito (recomendado para contexto industrial)
```

### 2.3 GAS — Funções auxiliares (inserir antes de `salvarFotosDriveTapControl`)

```javascript
function getOrCreateDrivePath(partes) {
  var pasta = DriveApp.getRootFolder();
  partes.forEach(function(nome) {
    var it = pasta.getFoldersByName(nome);
    pasta = it.hasNext() ? it.next() : pasta.createFolder(nome);
  });
  return pasta;
}

function dataUrlToBlob(dataUrl, nome) {
  var arr  = dataUrl.split(',');
  var mime = arr[0].match(/:(.*?);/)[1];
  var ext  = mime.split('/')[1] === 'png' ? '.png' : '.jpg';
  var bytes = Utilities.base64Decode(arr[1]);
  return Utilities.newBlob(bytes, mime, nome + ext);
}

function sanitizeFileName(s) {
  return String(s || '').replace(/[^\w\-_\.]/g, '_').substring(0, 80);
}

function _setColIfExists(aba, rowNum, cab, col, val) {
  var idx = cab.indexOf(col);
  if (idx >= 0) aba.getRange(rowNum, idx + 1).setValue(val);
}
```

### 2.4 GAS — `salvarFotosDriveTapControl(body)`

```javascript
function salvarFotosDriveTapControl(body) {
  /*
    body = {
      id_os, id_levantamento, id_inspetor, numero_os,
      drive_folder_id_existente: (string|'') — reusar pasta se reenvio,
      fotos_equipamentos:      { [id_equipamento]: "data:image/jpeg;base64,..." },
      fotos_pontos_instalacao: { [tipo_key]: "data:image/jpeg;base64,..." }
        // tipo_key ex: "EQ-001_manometro", "EQ-001_valvula"
    }
  */
  var resp = {
    drive_folder_id: '', drive_folder_url: '',
    fotos_salvas_equip: [],  // [{id_equipamento, file_id, file_url}]
    fotos_salvas_ponto: [],  // [{tipo_key, file_id, file_url}]
    fotos_com_erro: [],
    quantidade_fotos_equip: 0,
    quantidade_fotos_ponto: 0
  };

  var folder;
  // Reusar pasta existente se fornecida (caso de reenvio/correção)
  if (body.drive_folder_id_existente) {
    try { folder = DriveApp.getFolderById(body.drive_folder_id_existente); } catch(e) { folder = null; }
  }
  if (!folder) {
    var ym = Utilities.formatDate(new Date(), 'America/Sao_Paulo', 'yyyy-MM');
    folder = getOrCreateDrivePath([
      'TapControl_Fotos', ym,
      '[' + sanitizeFileName(body.id_os || 'SEM_OS') + ']',
      '[' + sanitizeFileName(body.id_levantamento || 'SEM_LEV') + ']'
    ]);
  }
  if (DRIVE_LINK_PUBLICO) {
    try { folder.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW); } catch(e) {}
  }
  resp.drive_folder_id  = folder.getId();
  resp.drive_folder_url = folder.getUrl();

  // Fotos de equipamentos
  Object.keys(body.fotos_equipamentos || {}).forEach(function(idEquip) {
    var dataUrl = body.fotos_equipamentos[idEquip];
    if (!dataUrl || typeof dataUrl !== 'string' || !dataUrl.startsWith('data:')) return;
    try {
      var nome = sanitizeFileName('equip_' + idEquip);
      var existing = folder.getFilesByName(nome + '.jpg');
      var arq = existing.hasNext() ? existing.next() : folder.createFile(dataUrlToBlob(dataUrl, nome));
      if (DRIVE_LINK_PUBLICO) { try { arq.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW); } catch(e) {} }
      resp.fotos_salvas_equip.push({ id_equipamento: idEquip, file_id: arq.getId(), file_url: arq.getUrl() });
    } catch(err) {
      resp.fotos_com_erro.push({ key: 'equip_' + idEquip, erro: err.message });
      logErro('salvarFotosDriveTapControl/equip', err.message, idEquip);
    }
  });

  // Fotos de pontos de instalação
  Object.keys(body.fotos_pontos_instalacao || {}).forEach(function(tipoKey) {
    var dataUrl = body.fotos_pontos_instalacao[tipoKey];
    if (!dataUrl || typeof dataUrl !== 'string' || !dataUrl.startsWith('data:')) return;
    try {
      var nome = sanitizeFileName('ponto_' + tipoKey);
      var existing = folder.getFilesByName(nome + '.jpg');
      var arq = existing.hasNext() ? existing.next() : folder.createFile(dataUrlToBlob(dataUrl, nome));
      if (DRIVE_LINK_PUBLICO) { try { arq.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW); } catch(e) {} }
      resp.fotos_salvas_ponto.push({ tipo_key: tipoKey, file_id: arq.getId(), file_url: arq.getUrl() });
    } catch(err) {
      resp.fotos_com_erro.push({ key: 'ponto_' + tipoKey, erro: err.message });
      logErro('salvarFotosDriveTapControl/ponto', err.message, tipoKey);
    }
  });

  resp.quantidade_fotos_equip = resp.fotos_salvas_equip.length;
  resp.quantidade_fotos_ponto = resp.fotos_salvas_ponto.length;
  return resp;
}
```

### 2.5 GAS — `getFotoBase64(params)`

```javascript
function getFotoBase64(params) {
  var fileId = norm(params.file_id);
  if (!fileId) return { status: 'erro', mensagem: 'file_id obrigatório' };
  try {
    var file = DriveApp.getFileById(fileId);
    var blob = file.getBlob();
    return {
      status: 'ok',
      base64: Utilities.base64Encode(blob.getBytes()),
      mimeType: blob.getContentType() || 'image/jpeg'
    };
  } catch(e) {
    logErro('getFotoBase64', e.message, JSON.stringify({ file_id: fileId }));
    return { status: 'erro', mensagem: 'Erro ao buscar foto: ' + e.message };
  }
}
```

Adicionar no roteador **GET** (bloco `switch(acao)`):
```javascript
case 'getFotoBase64':              resultado = getFotoBase64(params);              break;
case 'getLevantamentosComFotos':   resultado = getLevantamentosComFotos(params);   break;
```

### 2.6 GAS — `getLevantamentosComFotos(params)`

```javascript
function getLevantamentosComFotos(params) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var dados = ss.getSheetByName('LEVANTAMENTOS').getDataRange().getValues();
  var cab = dados[0];
  var osMap = _carregarMapaOS(ss); // função já existe no S22
  var lista = [];
  for (var i = 1; i < dados.length; i++) {
    var obj = _linhaParaObjeto(cab, dados[i]);
    // Só retorna levantamentos que têm fotos no Drive
    if (obj.drive_folder_id || parseInt(obj.quantidade_fotos_equip) > 0 || parseInt(obj.quantidade_fotos_ponto) > 0) {
      var infoOS = osMap[String(obj.id_os)];
      if (infoOS) {
        obj.cliente   = obj.cliente   || infoOS.cliente   || '';
        obj.numero_os = obj.numero_os || infoOS.numero_os || '';
        obj.id_cliente = obj.id_cliente || infoOS.id_cliente || '';
      }
      lista.push(obj);
    }
  }
  lista.sort(function(a, b) { return String(b.timestamp_envio).localeCompare(String(a.timestamp_envio)); });
  return { status: 'ok', levantamentos: lista };
}
```

### 2.7 GAS — Modificar `enviarLevantamento(body)` para chamar as fotos

**Localização:** logo antes do `return { status: 'ok', ... }` final, antes do `finally { lock.releaseLock(); }`.

**Remover** o bloco atual que lê e grava `foto_equipamento_b64` / `foto_ponto_instalacao_b64` nos itemBase (linhas ~950–959 e ~994–995). Substituir por string vazia nesses campos de ITENS, para não gravar base64 bruto.

**Adicionar** logo antes do `return`:

```javascript
// ── FOTOS NO DRIVE ──
var fotosInfo = {
  drive_folder_id: '', drive_folder_url: '',
  fotos_salvas_equip: [], fotos_salvas_ponto: [],
  fotos_com_erro: [],
  quantidade_fotos_equip: 0, quantidade_fotos_ponto: 0
};
var temFotos = (body.fotos_equipamentos && Object.keys(body.fotos_equipamentos).some(function(k){ return !!body.fotos_equipamentos[k]; })) ||
               (body.fotos_pontos_instalacao && Object.keys(body.fotos_pontos_instalacao).some(function(k){ return !!body.fotos_pontos_instalacao[k]; }));

if (temFotos) {
  try {
    // Verificar se levantamento já tem pasta (caso de reenvio/correção)
    var driveIdExistente = '';
    var abaLevCheck = ss.getSheetByName('LEVANTAMENTOS');
    var dadosLevCheck = abaLevCheck.getDataRange().getValues();
    var cabLevCheck = dadosLevCheck[0];
    for (var lc = 1; lc < dadosLevCheck.length; lc++) {
      if (String(dadosLevCheck[lc][cabLevCheck.indexOf('id_levantamento')] || '') === String(idLevantamento)) {
        driveIdExistente = String(dadosLevCheck[lc][cabLevCheck.indexOf('drive_folder_id')] || '');
        break;
      }
    }
    fotosInfo = salvarFotosDriveTapControl({
      id_os:                   body.id_os,
      id_levantamento:         idLevantamento,
      id_inspetor:             body.id_inspetor,
      numero_os:               body.numero_os || body.id_os,
      drive_folder_id_existente: driveIdExistente,
      fotos_equipamentos:      body.fotos_equipamentos || {},
      fotos_pontos_instalacao: body.fotos_pontos_instalacao || {}
    });
  } catch(errFotos) {
    logErro('enviarLevantamento/fotos', errFotos.message, body.id_os);
  }
}

// Gravar metadados de fotos na linha do LEVANTAMENTO
if (fotosInfo.drive_folder_id) {
  var abaLevF = ss.getSheetByName('LEVANTAMENTOS');
  var dadosLevF = abaLevF.getDataRange().getValues();
  var cabLevF = dadosLevF[0];
  for (var lf = 1; lf < dadosLevF.length; lf++) {
    if (String(dadosLevF[lf][cabLevF.indexOf('id_levantamento')] || '') === String(idLevantamento)) {
      _setColIfExists(abaLevF, lf + 1, cabLevF, 'drive_folder_id',       fotosInfo.drive_folder_id);
      _setColIfExists(abaLevF, lf + 1, cabLevF, 'drive_folder_url',      fotosInfo.drive_folder_url);
      _setColIfExists(abaLevF, lf + 1, cabLevF, 'quantidade_fotos_equip', fotosInfo.quantidade_fotos_equip);
      _setColIfExists(abaLevF, lf + 1, cabLevF, 'quantidade_fotos_ponto', fotosInfo.quantidade_fotos_ponto);
      _setColIfExists(abaLevF, lf + 1, cabLevF, 'fotos_equip_json',      JSON.stringify(fotosInfo.fotos_salvas_equip));
      _setColIfExists(abaLevF, lf + 1, cabLevF, 'fotos_ponto_json',      JSON.stringify(fotosInfo.fotos_salvas_ponto));
      SpreadsheetApp.flush();
      break;
    }
  }
}
```

**Modificar o `return`** para incluir info de fotos:
```javascript
// Substituir o return atual por:
return {
  status: 'ok',
  id_levantamento: idLevantamento,
  itens_criados: itensCriados,
  fotos: {
    drive_folder_url:      fotosInfo.drive_folder_url     || '',
    quantidade_fotos_equip: fotosInfo.quantidade_fotos_equip || 0,
    quantidade_fotos_ponto: fotosInfo.quantidade_fotos_ponto || 0,
    erros: (fotosInfo.fotos_com_erro || []).length
  }
};
```

**Nos `itemBase` de cada item** (dentro do `body.itens.forEach`), substituir os campos de foto por strings vazias (não gravar base64 no Sheets):
```javascript
foto_equipamento_b64:      '',  // S23 — base64 não gravado; fotos ficam no Drive
foto_ponto_instalacao_b64: '',  // metadados gravados na aba LEVANTAMENTOS
```

### 2.8 TCF — Feedback de fotos pós-envio

Na função `enviarLevantamento(btn)`, logo após o bloco que define `enviado = true` (dentro do `if (r && r.status === 'ok')`), adicionar:

```javascript
// Feedback de fotos salvas no Drive
if (r.fotos) {
  const totalFotos = (r.fotos.quantidade_fotos_equip || 0) + (r.fotos.quantidade_fotos_ponto || 0);
  if (totalFotos > 0) {
    console.log('[FOTOS] Salvas no Drive:', totalFotos, '— Pasta:', r.fotos.drive_folder_url);
  }
  if (r.fotos.erros > 0) {
    showToast(`⚠️ ${r.fotos.erros} foto(s) não puderam ser salvas no Drive.`, 'warn');
  }
}
```

### 2.9 TCF — Contador de fotos no resumo pré-envio (tela T6/confirmação)

Na função que renderiza o resumo antes do envio (tela `tc` ou tela de confirmação pré-envio), adicionar:

```javascript
const totalFotosEquip = Object.values(SES.fotoEquipamento || {}).filter(Boolean).length;
const totalFotosPonto = Object.values(SES.fotoPontoInstalacao || {}).filter(Boolean).length;
const totalFotos = totalFotosEquip + totalFotosPonto;
// Adicionar ao resumo exibido: 
// totalFotos > 0 ? `📷 ${totalFotos} foto(s) (${totalFotosEquip} equip. + ${totalFotosPonto} ponto)` : '📷 Sem fotos'
```

**Atenção:** localizar a tela de confirmação do TCF (`goTo('tc')` ou equivalente) e inserir a linha de fotos no sumário exibido ao inspetor.

### 2.10 TCM — Adicionar jsPDF e LOGO_ENGETAP_B64

No `<head>` do TCM, após as demais tags `<script>`:
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
```

Logo após, dentro do `<script>` principal (junto às constantes no topo):
```javascript
// Logo Engetap em PNG base64 — copiar string do ColectTap Manager v1.19 (constante LOGO_ENGETAP_B64)
// É uma string longa iniciando com "iVBORw0KGgo..." 
const LOGO_ENGETAP_B64 = ''; // ← preencher com o valor real do ColectTap Manager v1.19
```

> **IMPORTANTE para o Codex:** Buscar a constante `LOGO_ENGETAP_B64` no arquivo ColectTap Manager v1.19 e copiar o valor exato.

### 2.11 TCM — Barra de progresso (HTML + CSS)

Adicionar dentro do `<body>`, antes do fechamento da `</div>` principal (ou no final do body):

```html
<div id="rel-progress-wrap" style="display:none;position:fixed;bottom:20px;left:50%;transform:translateX(-50%);background:var(--nav);color:#fff;border-radius:10px;padding:12px 20px;min-width:300px;z-index:9999;box-shadow:0 4px 24px rgba(0,0,0,.3)">
  <div style="font-size:12px;margin-bottom:6px" id="rel-progress-label">Carregando fotos…</div>
  <div style="background:rgba(255,255,255,.2);border-radius:4px;height:6px">
    <div id="rel-progress-bar" style="background:var(--accent);border-radius:4px;height:6px;width:0%;transition:width .3s"></div>
  </div>
</div>
```

### 2.12 TCM — Nav-item Fotos/Drive

Na sidebar, após o nav-item de Levantamentos e antes de CADASTROS:

```html
<div class="nav-item" onclick="setView('fotos',this)" id="nav-fotos">
  <span class="ni-icon">📷</span> Fotos / Drive
</div>
```

### 2.13 TCM — View `view-fotos`

Adicionar após a `div#view-levantamentos`:

```html
<div class="view" id="view-fotos">
  <div class="filter-bar">
    <div class="search-wrap">
      <input type="text" id="fotos-search" placeholder="Buscar por OS, inspetor, levantamento..." oninput="renderViewFotos()">
    </div>
    <button class="btn btn-ghost btn-sm" onclick="carregarViewFotos()" style="margin-left:auto">🔄 Atualizar</button>
  </div>
  <div id="fotos-container">
    <div class="empty-state"><div class="ei">📷</div>Clique em Atualizar para carregar levantamentos com fotos</div>
  </div>
</div>
```

### 2.14 TCM — Funções `carregarViewFotos` e `renderViewFotos`

```javascript
async function carregarViewFotos() {
  const el = document.getElementById('fotos-container');
  if (el) el.innerHTML = '<div class="empty-state"><div class="ei">⏳</div>Carregando…</div>';
  try {
    const r = await gasGet('getLevantamentosComFotos');
    CACHE.levantamentosComFotos = r.levantamentos || [];
    renderViewFotos();
  } catch(e) {
    if (el) el.innerHTML = '<div class="empty-state"><div class="ei">⚠️</div>Erro ao carregar fotos: ' + e.message + '</div>';
  }
}

function renderViewFotos() {
  const lista = CACHE.levantamentosComFotos || [];
  const q = (document.getElementById('fotos-search')?.value || '').toLowerCase();
  const filtrada = lista.filter(l =>
    !q ||
    (l.id_os || '').toLowerCase().includes(q) ||
    (l.id_levantamento || '').toLowerCase().includes(q) ||
    (l.nome_inspetor || '').toLowerCase().includes(q) ||
    (l.cliente || '').toLowerCase().includes(q)
  );
  const el = document.getElementById('fotos-container');
  if (!el) return;
  if (!filtrada.length) {
    el.innerHTML = '<div class="empty-state"><div class="ei">📷</div>Nenhum levantamento com fotos encontrado</div>';
    return;
  }
  el.innerHTML = `
    <div style="font-size:12px;color:var(--text2);margin-bottom:12px">${filtrada.length} levantamento${filtrada.length!==1?'s':''} com fotos</div>
    <table class="data-table">
      <thead>
        <tr>
          <th>Levantamento</th><th>OS</th><th>Cliente</th><th>Inspetor</th><th>Fase</th>
          <th style="text-align:center">📷 Equip.</th><th style="text-align:center">📷 Ponto</th>
          <th>Pasta Drive</th><th>Ações</th>
        </tr>
      </thead>
      <tbody>
        ${filtrada.map(l => {
          const nEquip = parseInt(l.quantidade_fotos_equip) || 0;
          const nPonto = parseInt(l.quantidade_fotos_ponto) || 0;
          const driveBtn = l.drive_folder_url
            ? `<a href="${s(l.drive_folder_url)}" target="_blank" class="btn btn-ghost btn-sm" style="font-size:11px;text-decoration:none">📁 Abrir</a>`
            : '—';
          const pdfBtn = (nEquip + nPonto) > 0
            ? `<button class="btn btn-primary btn-sm" onclick="gerarPDFLevantamento('${s(l.id_levantamento)}')">📄 PDF</button>`
            : `<button class="btn btn-ghost btn-sm" disabled title="Sem fotos">📄 PDF</button>`;
          return `<tr>
            <td><code style="font-size:11px">${s(l.id_levantamento)}</code></td>
            <td>${s(l.numero_os || l.id_os)}</td>
            <td>${s(l.cliente)}</td>
            <td>${s(l.nome_inspetor)}</td>
            <td>${String(l.fase)==='2'?'<span class="phase-badge phase-2">F2</span>':'<span class="phase-badge phase-1">F1</span>'}</td>
            <td style="text-align:center">${nEquip || '—'}</td>
            <td style="text-align:center">${nPonto || '—'}</td>
            <td>${driveBtn}</td>
            <td style="display:flex;gap:4px">${pdfBtn}</td>
          </tr>`;
        }).join('')}
      </tbody>
    </table>`;
}
```

Chamar `carregarViewFotos()` dentro da função `setView()` quando a view for `'fotos'`:
```javascript
// Dentro de setView(), no bloco de ações por view (onde já há "if (id === 'pipeline') {...}"):
if (id === 'fotos') carregarViewFotos();
```

### 2.15 TCM — `_buscarFotosLevantamento(levantamento)`

```javascript
async function _buscarFotosLevantamento(levantamento) {
  const wrap  = document.getElementById('rel-progress-wrap');
  const bar   = document.getElementById('rel-progress-bar');
  const label = document.getElementById('rel-progress-label');

  const tarefasEquip = [];
  const tarefasPonto = [];

  try {
    const fotosEquip = JSON.parse(s(levantamento.fotos_equip_json) || '[]');
    fotosEquip.forEach(f => { if (f.file_id) tarefasEquip.push(f); });
  } catch(e) {}
  try {
    const fotosPonto = JSON.parse(s(levantamento.fotos_ponto_json) || '[]');
    fotosPonto.forEach(f => { if (f.file_id) tarefasPonto.push(f); });
  } catch(e) {}

  const todas = [...tarefasEquip, ...tarefasPonto];
  const resultadoEquip = new Map();
  const resultadoPonto = new Map();

  if (!todas.length) return { equip: resultadoEquip, ponto: resultadoPonto };

  if (wrap) wrap.style.display = 'block';
  if (bar) bar.style.width = '0%';

  for (let i = 0; i < todas.length; i++) {
    const t = todas[i];
    const labelTxt = t.id_equipamento ? `Equip. ${t.id_equipamento}` : `Ponto ${t.tipo_key}`;
    if (label) label.textContent = `Carregando foto ${i+1}/${todas.length}: ${labelTxt}`;
    if (bar) bar.style.width = Math.round(((i+1)/todas.length)*100) + '%';
    try {
      const r = await gasGet('getFotoBase64', { file_id: t.file_id });
      if (r.status === 'ok' && r.base64) {
        if (t.id_equipamento) resultadoEquip.set(t.id_equipamento, { base64: r.base64, mimeType: r.mimeType || 'image/jpeg' });
        if (t.tipo_key)       resultadoPonto.set(t.tipo_key,       { base64: r.base64, mimeType: r.mimeType || 'image/jpeg' });
      }
    } catch(e) { console.warn('[PDF] Foto não carregada:', t.file_id, e.message); }
  }

  if (label) label.textContent = '✅ Fotos carregadas!';
  setTimeout(() => { if (wrap) wrap.style.display = 'none'; }, 1500);
  return { equip: resultadoEquip, ponto: resultadoPonto };
}
```

### 2.16 TCM — `gerarPDFLevantamento(idLevantamento)`

```javascript
async function gerarPDFLevantamento(idLevantamento) {
  if (!window.jspdf) { mostrarFeedback('⚠️ Biblioteca PDF não carregada ainda. Aguarde e tente novamente.', true); return; }

  // Buscar o levantamento no cache (view-fotos) ou nos levantamentos gerais
  let lev = (CACHE.levantamentosComFotos || []).find(l => l.id_levantamento === idLevantamento);
  if (!lev) lev = (DB.levantamentos || []).find(l => l.id_levantamento === idLevantamento);
  if (!lev) { mostrarFeedback('Levantamento não encontrado. Sincronize os dados primeiro.', true); return; }

  mostrarFeedback('Buscando itens do levantamento…');
  let itens = [];
  try {
    const ri = await gasGet('getItensByLevantamento', { id_levantamento: idLevantamento });
    itens = ri.itens || [];
  } catch(e) { mostrarFeedback('Erro ao buscar itens: ' + e.message, true); return; }

  mostrarFeedback('Carregando fotos do Drive…');
  const fotosMap = await _buscarFotosLevantamento(lev);

  const { jsPDF } = window.jspdf;
  const doc = new jsPDF({ orientation: 'portrait', unit: 'mm', format: 'a4' });
  const PW = 210, PH = 297, ML = 14, MR = 14, CW = PW - ML - MR;

  const ltx = t => (t || '').normalize('NFD').replace(/[\u0300-\u036f]/g, '');
  const sv  = v => String(v || '').trim();

  // ── CAPA ──
  if (LOGO_ENGETAP_B64) {
    try { doc.addImage('data:image/png;base64,' + LOGO_ENGETAP_B64, 'PNG', PW/2 - 40, 18, 80, 27); } catch(e) {}
  }
  doc.setFont('helvetica', 'bold'); doc.setFontSize(16); doc.setTextColor(13, 27, 42);
  doc.text(ltx('RELATÓRIO DE ACESSÓRIOS NR-13'), PW/2, 58, { align: 'center' });

  doc.setFont('helvetica', 'normal'); doc.setFontSize(10); doc.setTextColor(74, 104, 130);
  doc.text(`OS: ${ltx(sv(lev.numero_os || lev.id_os))}`, PW/2, 66, { align: 'center' });
  doc.text(ltx(`Inspetor: ${sv(lev.nome_inspetor)}  |  Fase: ${sv(lev.fase)}`), PW/2, 72, { align: 'center' });
  doc.text(ltx(`Levantamento: ${sv(lev.id_levantamento)}  |  ${sv(lev.cliente)}`), PW/2, 78, { align: 'center' });

  // Foto de capa (primeira foto de equipamento disponível)
  if (fotosMap.equip.size > 0) {
    const primeiraFoto = fotosMap.equip.values().next().value;
    try {
      const mt = (primeiraFoto.mimeType || '').toUpperCase().includes('PNG') ? 'PNG' : 'JPEG';
      const imgSrc = 'data:' + primeiraFoto.mimeType + ';base64,' + primeiraFoto.base64;
      const props = doc.getImageProperties(imgSrc);
      const imgW = CW * 0.75;
      const imgH = Math.min(imgW * (props.height / props.width), 80);
      const imgX = ML + (CW - imgW) / 2;
      doc.addImage(imgSrc, mt, imgX, 86, imgW, imgH);
    } catch(e) {}
  }

  doc.setDrawColor(220, 230, 245); doc.setLineWidth(0.5); doc.line(ML, 175, PW - MR, 175);
  doc.setFont('helvetica', 'italic'); doc.setFontSize(8); doc.setTextColor(154, 181, 204);
  doc.text(ltx('Documento gerado pelo TapControl Manager · Engetap'), PW/2, 180, { align: 'center' });
  doc.text(ltx('Data: ' + new Date().toLocaleDateString('pt-BR')), PW/2, 185, { align: 'center' });

  // ── PÁGINAS DE CONTEÚDO — agrupadas por equipamento ──
  const porEquip = {};
  itens.forEach(it => {
    const id = sv(it.id_equipamento) || 'sem_equip';
    if (!porEquip[id]) porEquip[id] = { tag: sv(it.tag_equipamento), itens: [] };
    porEquip[id].itens.push(it);
  });

  const colW = [64, 30, 14, 30, 44]; // Descrição, Tipo, Qtd, Status, Calibração/Obs
  const headers = ['Descrição', 'Tipo', 'Qtd', 'Status', 'Calibração NR-13'];

  Object.entries(porEquip).forEach(([idEquip, grupo]) => {
    doc.addPage();
    let y = 20;

    // Cabeçalho de equipamento
    doc.setFillColor(13, 27, 42);
    doc.rect(ML, y, CW, 10, 'F');
    doc.setFont('helvetica', 'bold'); doc.setFontSize(11); doc.setTextColor(255, 255, 255);
    doc.text(ltx(`Equipamento: ${grupo.tag} (${idEquip})`), ML + 4, y + 7);
    y += 16;

    // Foto do equipamento
    const fotoEquip = fotosMap.equip.get(idEquip);
    if (fotoEquip) {
      try {
        const mt = (fotoEquip.mimeType || '').toUpperCase().includes('PNG') ? 'PNG' : 'JPEG';
        const imgSrc = 'data:' + fotoEquip.mimeType + ';base64,' + fotoEquip.base64;
        const props = doc.getImageProperties(imgSrc);
        const imgW = CW / 2;
        const imgH = Math.min(imgW * (props.height / props.width), 60);
        doc.addImage(imgSrc, mt, ML, y, imgW, imgH);
        doc.setFont('helvetica', 'italic'); doc.setFontSize(7); doc.setTextColor(74, 104, 130);
        doc.text(ltx('Foto do local de instalação'), ML, y + imgH + 3);
        y += imgH + 8;
      } catch(e) { y += 4; }
    }

    // Cabeçalho da tabela
    doc.setFillColor(220, 230, 245);
    doc.rect(ML, y, CW, 6, 'F');
    doc.setFont('helvetica', 'bold'); doc.setFontSize(7.5); doc.setTextColor(13, 27, 42);
    let xCol = ML;
    headers.forEach((h, i) => { doc.text(ltx(h), xCol + 2, y + 4); xCol += colW[i]; });
    y += 6;

    // Linhas de itens
    doc.setFont('helvetica', 'normal'); doc.setFontSize(7.5);
    grupo.itens.forEach((it, idx) => {
      if (y > PH - 30) { doc.addPage(); y = 20; }
      if (idx % 2 === 0) { doc.setFillColor(247, 250, 253); doc.rect(ML, y, CW, 6, 'F'); }
      doc.setTextColor(13, 27, 42);
      xCol = ML;
      const calibLabel = String(it.requer_calibracao) === 'true'
        ? (it.num_certificado ? `Cert: ${sv(it.num_certificado).substring(0,12)}` : 'Necessária')
        : '—';
      const vals = [
        sv(it.descricao_curta).substring(0, 38),
        sv(it.tipo_acessorio).substring(0, 16),
        sv(it.quantidade),
        ltx(sv(it.status)).substring(0, 16),
        ltx(calibLabel)
      ];
      vals.forEach((v, i) => { doc.text(ltx(v), xCol + 2, y + 4); xCol += colW[i]; });
      y += 6;

      // Foto do ponto de instalação para este item (se houver)
      const fotoKey = sv(it.id_equipamento) + '_' + sv(it.tipo_acessorio)
        .toLowerCase().normalize('NFD').replace(/[\u0300-\u036f]/g,'')
        .replace(/\s+de\s+/g,'_').replace(/\s+/g,'_');
      const fotoPonto = fotosMap.ponto.get(fotoKey);
      if (fotoPonto) {
        try {
          const mt2 = (fotoPonto.mimeType || '').toUpperCase().includes('PNG') ? 'PNG' : 'JPEG';
          const imgSrc2 = 'data:' + fotoPonto.mimeType + ';base64,' + fotoPonto.base64;
          const props2 = doc.getImageProperties(imgSrc2);
          const imgW2 = CW / 3;
          const imgH2 = Math.min(imgW2 * (props2.height / props2.width), 40);
          if (y + imgH2 > PH - 30) { doc.addPage(); y = 20; }
          doc.addImage(imgSrc2, mt2, ML, y, imgW2, imgH2);
          doc.setFont('helvetica', 'italic'); doc.setFontSize(6.5); doc.setTextColor(74, 104, 130);
          doc.text(ltx('Ponto: ' + sv(it.tipo_acessorio)), ML, y + imgH2 + 2);
          doc.setFont('helvetica', 'normal'); doc.setFontSize(7.5); doc.setTextColor(13, 27, 42);
          y += imgH2 + 6;
        } catch(e) {}
      }
    });

    // Rodapé da página de equipamento
    doc.setFont('helvetica', 'italic'); doc.setFontSize(7); doc.setTextColor(154, 181, 204);
    doc.text(ltx(`OS: ${sv(lev.numero_os || lev.id_os)} · Lev: ${sv(lev.id_levantamento)} · Equip: ${idEquip}`), PW/2, PH - 8, { align: 'center' });
  });

  const nomeArq = `Engetap_NR13_${sv(lev.numero_os || lev.id_os)}_${new Date().toISOString().substring(0, 10)}.pdf`;
  doc.save(nomeArq);
  mostrarFeedback('✅ PDF gerado: ' + nomeArq);
}
```

### 2.17 TCM — Botão PDF nos cards de levantamento (view-levantamentos)

Na função `renderLevantamentos()`, dentro do template de cada card (`.lev-card`), adicionar ao bloco de botões de ação junto ao botão "👁 Ver itens":

```javascript
// Dentro do map de data.map(l => ...), no bloco de botões (div com display:flex;gap:6px):
const nFotosLev = (parseInt(l.quantidade_fotos_equip) || 0) + (parseInt(l.quantidade_fotos_ponto) || 0);
const btnPDFLev = nFotosLev > 0
  ? `<button class="btn btn-primary btn-sm" onclick="gerarPDFLevantamento('${l.id_levantamento}')" title="${nFotosLev} foto(s) disponível/eis">📄 PDF</button>`
  : `<button class="btn btn-ghost btn-sm" style="opacity:.45" disabled title="Sem fotos para PDF">📄</button>`;
```

Adicionar esse `btnPDFLev` ao lado do botão existente "👁 Ver itens" na linha do card.

---

## 3. PREMISSAS CRÍTICAS

1. **Nunca salvar base64 no Sheets.** O GAS salva apenas `file_id` e `file_url` na planilha LEVANTAMENTOS. As colunas `foto_equipamento_b64` e `foto_ponto_instalacao_b64` de ITENS ficam com string vazia para novos levantamentos (manter as colunas por compatibilidade, mas não preencher).

2. **Reenvio/correção.** A função `salvarFotosDriveTapControl` verifica se já há `drive_folder_id` gravado na linha do levantamento e reusa a pasta existente, não criando nova.

3. **Timeout do TCF.** O `gasPost('enviarLevantamento', payload, 60000)` já tem 60s — manter. Upload de muitas fotos pode ser lento.

4. **`_garantirColunas` é idempotente.** Pode ser chamada múltiplas vezes sem efeito colateral. Adicionar as colunas novas sem remover as existentes.

5. **Compatibilidade retroativa.** Levantamentos antigos (sem fotos) terão os campos de Drive vazios/zero. O TCM deve tratar `undefined`/`''`/`0` como "sem fotos" e não quebrar.

6. **`CACHE.levantamentosComFotos`** é um array do cache local do TCM. Verificar se o objeto `CACHE` já existe no TCM S22 — se sim, apenas adicionar a chave; se não existir como objeto global, declarar: `if (!window.CACHE) window.CACHE = {};`.

7. **Função `s(val)`** é um helper já existente no TCM S22 que sanitiza strings para uso em HTML (retorna `String(val||'')`). Usar onde necessário para evitar XSS.

8. **`mostrarFeedback(msg, isErro)`** — verificar o nome exato da função de feedback no TCM S22. Pode ser `showToast(msg, 'err')` ou similar. Adaptar conforme o que existir.

9. **`norm(val)`** no GAS — função utilitária já existente que faz `String(val||'').trim()`. Usar em `getFotoBase64`.

10. **`logErro(contexto, msg, detalhe)`** — função já existente no GAS S22 para registro de erros no LOG_OPERACIONAL.

---

## 4. ARQUIVOS A MODIFICAR

| Arquivo | Modificações |
|---|---|
| `GAS_Code_S22_0.js` | + constante `DRIVE_LINK_PUBLICO`; + helpers `getOrCreateDrivePath`, `dataUrlToBlob`, `sanitizeFileName`, `_setColIfExists`; + `salvarFotosDriveTapControl`, `getFotoBase64`, `getLevantamentosComFotos`; modificar `enviarLevantamento` (bloco de fotos + return); adicionar casos no roteador GET; adicionar `_garantirColunas` para LEVANTAMENTOS em `inicializarPlanilha` |
| `TCM_S22_0.html` | + `<script jsPDF>`; + `LOGO_ENGETAP_B64`; + barra de progresso HTML; + nav-item fotos; + `view-fotos`; + `carregarViewFotos`, `renderViewFotos`, `_buscarFotosLevantamento`, `gerarPDFLevantamento`; + botão PDF em `renderLevantamentos`; + trigger em `setView` |
| `TCF_S22_0.html` | Feedback de fotos em `enviarLevantamento` (pós `r.status === 'ok'`); contador de fotos no resumo pré-envio |

---

## 5. ORDEM DE IMPLEMENTAÇÃO

1. **GAS:** constante `DRIVE_LINK_PUBLICO` + 4 funções auxiliares
2. **GAS:** `salvarFotosDriveTapControl`
3. **GAS:** `getFotoBase64` + `getLevantamentosComFotos` + roteador GET
4. **GAS:** modificar `enviarLevantamento` (bloco fotos + return + itemBase)
5. **GAS:** `_garantirColunas` LEVANTAMENTOS em `inicializarPlanilha`
6. **TCF:** feedback pós-envio + contador no resumo
7. **TCM:** `<script jsPDF>` + `LOGO_ENGETAP_B64` + barra de progresso HTML
8. **TCM:** `_buscarFotosLevantamento`
9. **TCM:** `gerarPDFLevantamento`
10. **TCM:** nav-item + `view-fotos` + `carregarViewFotos` + `renderViewFotos`
11. **TCM:** botão PDF em `renderLevantamentos` + trigger em `setView`

---

## 6. RESULTADO ESPERADO

1. Inspetor abre TCF → tira fotos de equipamentos e pontos → envia levantamento
2. GAS cria pasta `TapControl_Fotos/YYYY-MM/[OS]/[LEV]` → salva arquivos → grava apenas `file_id` e `file_url` em LEVANTAMENTOS
3. Admin abre TCM → view Fotos/Drive → vê levantamento com 2 fotos equip + 3 fotos ponto → clica 📁 → abre pasta no Drive diretamente
4. Admin clica 📄 PDF → barra de progresso aparece → TCM busca itens via `getItensByLevantamento` → busca fotos via `getFotoBase64` → gera PDF com: capa Engetap + foto principal + páginas por equipamento com tabela de acessórios e fotos inline → download automático
5. Reenvio: GAS detecta `drive_folder_id` existente → reusa pasta → não duplica fotos

