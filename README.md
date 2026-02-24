<!DOCTYPE html>
<html lang="pt-br">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Gerador de Keys + Painel Admin HWID</title>
  <style>
    body {
      font-family: system-ui, sans-serif;
      background: #0d1117;
      color: #c9d1d9;
      padding: 20px;
      max-width: 1000px;
      margin: auto;
    }
    h1, h2 { color: #58a6ff; text-align: center; }
    .container {
      background: #161b22;
      padding: 20px;
      border-radius: 10px;
      border: 1px solid #30363d;
      margin: 20px 0;
    }
    input, select, textarea, button {
      padding: 10px 14px;
      margin: 6px 0;
      border-radius: 6px;
      border: 1px solid #444;
      background: #0d1117;
      color: #e6edf3;
      font-family: 'Courier New', monospace;
      width: 100%;
      box-sizing: border-box;
    }
    button {
      background: #238636;
      border: none;
      cursor: pointer;
      font-weight: bold;
    }
    button:hover { background: #2ea043; }
    button.danger { background: #b91c1c; }
    button.danger:hover { background: #dc2626; }
    button.addtime { background: #7c3aed; }
    button.addtime:hover { background: #9f7aea; }
    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 16px;
      font-size: 0.92rem;
    }
    th, td {
      padding: 10px;
      text-align: left;
      border-bottom: 1px solid #30363d;
      vertical-align: top;
    }
    th { background: #1f2937; }
    .status {
      padding: 14px;
      border-radius: 8px;
      margin: 12px 0;
      white-space: pre-wrap;
    }
    .success { background: #064e3b; color: #a7f3d0; }
    .error   { background: #7f1d1d; color: #fecaca; }
    .info    { background: #1e40af; color: #bfdbfe; }
    .small   { font-size: 0.85rem; color: #8b949e; word-break: break-all; }
    .key-full { 
      font-size: 0.9rem; 
      word-break: break-all; 
      color: #a5d6ff; 
      max-width: 300px; 
    }
  </style>
</head>
<body>

<h1>Gerador de Keys + Painel de Controle</h1>

<div class="container">
  <h2>1. Gerar Key</h2>
  <label>Prefixo / Nome do produto:</label>
  <input type="text" id="prefixo" value="GHOST" maxlength="12">

  <label>Tipo de licença:</label>
  <select id="tipo">
    <option value="diaria">Diária (1 dia)</option>
    <option value="semanal" selected>Semanal (7 dias)</option>
    <option value="mensal">Mensal (30 dias)</option>
    <option value="vitalicia">Vitalícia (∞)</option>
  </select>

  <button onclick="gerarKey()">Gerar Key</button>
  <textarea id="keyGerada" rows="2" readonly placeholder="Key gerada aparece aqui..."></textarea>
</div>

<div class="container">
  <h2>2. Ativar Licença</h2>
  <textarea id="keyInput" rows="2" placeholder="Cole a key aqui..."></textarea>
  <button onclick="ativarKey()">Ativar</button>

  <div id="statusAtivacao" class="status info">Nenhuma ação realizada.</div>
  <div class="small">HWID atual: <span id="hwidAtual"></span></div>
</div>

<div class="container">
  <h2>3. Painel Admin (licenças ativas neste navegador)</h2>
  <p class="small">Mostra apenas as ativações salvas neste navegador (localStorage)</p>

  <table id="tabelaLicencas">
    <thead>
      <tr>
        <th>Key completa</th>
        <th>HWID (resumido)</th>
        <th>Ativado em</th>
        <th>Validade restante</th>
        <th>Ações</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>

  <div style="margin-top: 16px;">
    <button onclick="atualizarPainel()">Atualizar Painel</button>
  </div>
</div>

<script>
// =================================================================
//   APENAS ESTUDO / PROTÓTIPO — NÃO É SEGURO PARA PRODUTO REAL
// =================================================================

const SECRET = "sistema-licenca-2025-xx-secreto";
const STORAGE_KEY = "__licencas_ativas__";

function simpleHash(str) {
  let h = 0;
  for (let i = 0; i < str.length; i++) {
    h = Math.imul(31, h) + str.charCodeAt(i) | 0;
  }
  return (h >>> 0).toString(36).toUpperCase();
}

function getHWID() {
  const c = document.createElement('canvas');
  const ctx = c.getContext('2d');
  ctx.fillStyle = '#f60'; ctx.fillRect(125,1,62,20);
  ctx.fillStyle = '#069'; ctx.font = '14px Arial';
  ctx.fillText("HWID-2025", 2, 15);
  ctx.fillStyle = 'rgba(102,204,0,0.7)'; ctx.fillText("HWID-2025", 4, 17);

  const data = [
    navigator.userAgent,
    navigator.language,
    screen.width + 'x' + screen.height,
    new Date().getTimezoneOffset(),
    c.toDataURL().slice(-80)
  ].join('###');

  return simpleHash(data + SECRET).slice(0, 32);
}

function diasPorTipo(tipo) {
  const map = { diaria: 1, semanal: 7, mensal: 30, vitalicia: 999999 };
  return map[tipo] || 7;
}

function gerarKey() {
  const prefixo = (document.getElementById('prefixo').value || 'LIC').trim().toUpperCase().replace(/[^A-Z0-9]/g,'');
  const tipo = document.getElementById('tipo').value;
  const dias = diasPorTipo(tipo);

  const payload = { t: Date.now(), d: dias, p: prefixo };
  const json = JSON.stringify(payload);
  const hash = simpleHash(json + SECRET).slice(0,10);

  let b64 = btoa(json).replace(/=/g,'').replace(/\+/g,'-').replace(/\//g,'_');
  const key = `${prefixo}-${b64}-${hash}`;

  document.getElementById('keyGerada').value = key;
}

function lerLicencas() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? JSON.parse(raw) : [];
  } catch {
    return [];
  }
}

function salvarLicencas(lista) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(lista));
}

function ativarKey() {
  const key = document.getElementById('keyInput').value.trim();
  const status = document.getElementById('statusAtivacao');
  const hwid = getHWID();
  document.getElementById('hwidAtual').textContent = hwid;

  if (!key) return statusErro("Cole uma key.");

  const partes = key.split('-');
  if (partes.length !== 3) return statusErro("Formato inválido.");

  const [prefixo, b64, hashFornecido] = partes;
  let json;
  try {
    json = atob(b64.replace(/-/g,'+').replace(/_/g,'/') + '==');
  } catch {
    return statusErro("Base64 inválido.");
  }

  if (simpleHash(json + SECRET).slice(0,10) !== hashFornecido) {
    return statusErro("Assinatura inválida.");
  }

  let payload;
  try { payload = JSON.parse(json); } catch { return statusErro("Dados corrompidos."); }

  let licencas = lerLicencas();

  // Verifica se esta key já está ativada neste HWID
  const jaExiste = licencas.find(l => l.key === key && l.hwid === hwid);
  if (jaExiste) {
    const exp = jaExiste.ativadoEm + jaExiste.dias * 86400000;
    const restam = Math.max(0, Math.ceil((exp - Date.now()) / 86400000));
    return statusSucesso(`Já ativada.\nRestam ≈ ${restam} dias`);
  }

  // Adiciona nova ativação
  licencas.push({
    key,
    hwid,
    ativadoEm: Date.now(),
    dias: payload.d,
    prefixo: payload.p
  });

  salvarLicencas(licencas);
  statusSucesso(`Ativação OK!\nProduto: ${payload.p}\nValidade: ${payload.d >= 999999 ? 'Vitalícia' : payload.d + ' dias'}`);
  atualizarPainel();
}

function statusSucesso(msg) { 
  const el = document.getElementById('statusAtivacao');
  el.textContent = msg;
  el.className = 'status success';
}
function statusErro(msg) { 
  const el = document.getElementById('statusAtivacao');
  el.textContent = msg;
  el.className = 'status error';
}

function revogarLicenca(index) {
  let licencas = lerLicencas();
  if (index < 0 || index >= licencas.length) return;
  licencas.splice(index, 1);
  salvarLicencas(licencas);
  atualizarPainel();
}

function adicionarDias(index, diasExtra) {
  let licencas = lerLicencas();
  if (index < 0 || index >= licencas.length) return;
  if (licencas[index].dias >= 999999) return; // já vitalícia

  licencas[index].dias += diasExtra;
  salvarLicencas(licencas);
  atualizarPainel();
}

function formatarData(ts) {
  return new Date(ts).toLocaleString('pt-BR', {dateStyle: 'short', timeStyle: 'short'});
}

function atualizarPainel() {
  const tbody = document.querySelector('#tabelaLicencas tbody');
  tbody.innerHTML = '';

  const licencas = lerLicencas();
  if (licencas.length === 0) {
    tbody.innerHTML = '<tr><td colspan="5" style="text-align:center; padding:20px;">Nenhuma licença ativa neste navegador</td></tr>';
    return;
  }

  licencas.forEach((lic, index) => {
    const expiraEm = lic.ativadoEm + lic.dias * 86400000;
    const restam = lic.dias >= 999999 
      ? 'Vitalícia' 
      : Math.max(0, Math.ceil((expiraEm - Date.now()) / 86400000)) + ' dias';

    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td><div class="key-full">${lic.key}</div></td>
      <td class="small">${lic.hwid.slice(0,8)}...${lic.hwid.slice(-6)}</td>
      <td>${formatarData(lic.ativadoEm)}</td>
      <td>${restam}</td>
      <td>
        <button class="danger" onclick="revogarLicenca(${index})">Revogar</button>
        <button class="addtime" onclick="adicionarDias(${index}, 7)">+7 dias</button>
        <button class="addtime" onclick="adicionarDias(${index}, 30)">+30 dias</button>
      </td>
    `;
    tbody.appendChild(tr);
  });
}

// Inicializa
document.addEventListener('DOMContentLoaded', () => {
  document.getElementById('hwidAtual').textContent = getHWID();
  atualizarPainel();
});
</script>

</body>
</html>
