<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>QuickLoan UGX</title>
<style>
*{box-sizing:border-box}
body{margin:0;font-family:system-ui,Arial,sans-serif;background:#f1f5f2;color:#12261b}
header{background:#0f5132;color:#fff;padding:14px 16px;display:flex;justify-content:space-between;align-items:center;position:sticky;top:0;z-index:5}
header h1{font-size:18px;margin:0}
button{font:inherit;cursor:pointer;border:0;border-radius:10px;padding:10px 14px}
.primary{background:#0f5132;color:#fff;font-weight:700;width:100%;margin:12px 0}
.ghost{background:#e3ebe6;color:#0f5132;font-weight:600}
.danger{background:#fde8e8;color:#b42318;font-weight:600}
.wa{background:#25d366;color:#073b1b;font-weight:700;text-decoration:none;border-radius:10px;padding:10px 14px;display:inline-block;font-size:14px}
main{max-width:520px;margin:auto;padding:14px}
.cards{display:grid;grid-template-columns:1fr 1fr;gap:10px}
.stat{background:#fff;border-radius:14px;padding:12px;box-shadow:0 1px 3px #0001}
.stat small{color:#5b7265;display:block;font-size:11px;text-transform:uppercase;letter-spacing:.4px}
.stat b{font-size:16px}
.stat.bad b{color:#b42318}
input,select{width:100%;padding:11px;border:1px solid #c7d6cc;border-radius:10px;font:inherit;margin-top:4px;background:#fff}
label{display:block;font-size:13px;font-weight:600;margin-top:10px}
.loan{background:#fff;border-radius:14px;padding:14px;margin-top:12px;box-shadow:0 1px 3px #0001}
.row{display:flex;justify-content:space-between;align-items:center;gap:8px}
.tag{font-size:11px;font-weight:700;padding:3px 8px;border-radius:99px}
.active{background:#dcf3e4;color:#0f5132}.overdue{background:#fde8e8;color:#b42318}.paid{background:#e2e8f0;color:#334155}
.bar{height:8px;background:#e3ebe6;border-radius:99px;overflow:hidden;margin:10px 0}
.bar i{display:block;height:100%;background:#0f5132}
.btns{display:flex;gap:8px;flex-wrap:wrap;margin-top:10px}
.muted{color:#5b7265;font-size:12px}
details{margin-top:8px;font-size:13px}
dialog{border:0;border-radius:18px;padding:18px;width:min(92vw,440px)}
dialog::backdrop{background:#0008}
.prev{background:#f1f5f2;border-radius:10px;padding:10px;margin-top:12px;font-size:13px}
.empty{text-align:center;color:#5b7265;padding:30px 0}
</style>
</head>
<body>
<header><h1>QuickLoan UGX</h1><button class="ghost" id="bkBtn">Backup</button></header>
<main>
  <div class="cards" id="stats"></div>
  <button class="primary" id="addBtn">+ New Loan</button>
  <input id="q" placeholder="Search name or phone">
  <div id="list"></div>
</main>

<dialog id="dNew">
  <h3 style="margin:0">New Loan</h3>
  <form id="fNew">
    <label>Client name<input id="nName" required></label>
    <label>Phone<input id="nPhone" type="tel" placeholder="0772123456"></label>
    <label>Amount lent (UGX)<input id="nAmt" type="number" min="1" required></label>
    <div class="row">
      <label style="flex:1">Interest % per month<input id="nRate" type="number" min="0" step="0.5" value="10" required></label>
      <label style="flex:1">Months<input id="nMon" type="number" min="1" max="60" value="1" required></label>
    </div>
    <label>Start date<input id="nStart" type="date" required></label>
    <div class="prev" id="nPrev"></div>
    <div class="btns"><button class="primary" style="flex:1;margin:0">Save Loan</button><button type="button" class="ghost" data-close="dNew">Cancel</button></div>
  </form>
</dialog>

<dialog id="dPay">
  <h3 style="margin:0">Record Payment</h3>
  <p class="muted" id="pInfo"></p>
  <form id="fPay">
    <input type="hidden" id="pId">
    <label>Amount (UGX)<input id="pAmt" type="number" min="1" required></label>
    <label>Date<input id="pDate" type="date" required></label>
    <label>Note (optional)<input id="pNote" placeholder="Cash, Mobile Money..."></label>
    <div class="btns"><button class="primary" style="flex:1;margin:0">Save Payment</button><button type="button" class="ghost" data-close="dPay">Cancel</button></div>
  </form>
</dialog>

<dialog id="dBk">
  <h3 style="margin:0">Backup & Restore</h3>
  <p class="muted">Your data is saved only in this browser. Download a backup often.</p>
  <div class="btns"><button class="primary" id="exp" style="margin:0">Download Backup</button></div>
  <label>Restore from backup file<input type="file" id="imp" accept=".json"></label>
  <div class="btns"><button type="button" class="ghost" data-close="dBk">Close</button></div>
</dialog>

<script>
const KEY = 'quickloan_v1';
let loans = [];
try { loans = JSON.parse(localStorage.getItem(KEY) || '[]'); } catch (e) { loans = []; }
const $ = s => document.querySelector(s);
const fmt = n => 'UGX ' + Math.round(n || 0).toLocaleString('en-US');
const ymd = d => d.getFullYear() + '-' + String(d.getMonth() + 1).padStart(2, '0') + '-' + String(d.getDate()).padStart(2, '0');
const today = () => ymd(new Date());
const addMonths = (s, m) => { const d = new Date(s + 'T00:00:00'); d.setMonth(d.getMonth() + m); return ymd(d); };
const esc = s => String(s == null ? '' : s).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
const paid = l => l.pays.reduce((s, p) => s + p.a, 0);
const bal = l => Math.max(0, l.total - paid(l));
const status = l => bal(l) <= 0 ? 'paid' : (l.due < today() ? 'overdue' : 'active');
const fmtDate = s => new Date(s + 'T00:00:00').toLocaleDateString('en-GB', {day:'2-digit', month:'short', year:'numeric'});

function save() {
  try { localStorage.setItem(KEY, JSON.stringify(loans)); } catch (e) { alert('Could not save data in this browser.'); }
  render();
}

function waLink(l) {
  let n = (l.phone || '').replace(/\D/g, '');
  if (!n) return '';
  if (n.startsWith('0')) n = '256' + n.slice(1);
  const msg = `Hello ${l.name}, this is a friendly reminder that your loan balance is ${fmt(bal(l))}, due on ${fmtDate(l.due)}. Thank you.`;
  return 'https://wa.me/' + n + '?text=' + encodeURIComponent(msg);
}

function render() {
  const lent = loans.reduce((s, l) => s + l.principal, 0);
  const expected = loans.reduce((s, l) => s + l.total, 0);
  const collected = loans.reduce((s, l) => s + paid(l), 0);
  const outstanding = loans.reduce((s, l) => s + bal(l), 0);
  const overdue = loans.filter(l => status(l) === 'overdue');
  $('#stats').innerHTML =
    `<div class="stat"><small>Total Lent</small><b>${fmt(lent)}</b></div>
     <div class="stat"><small>Collected</small><b>${fmt(collected)}</b></div>
     <div class="stat"><small>Outstanding</small><b>${fmt(outstanding)}</b></div>
     <div class="stat ${overdue.length ? 'bad' : ''}"><small>Overdue (${overdue.length})</small><b>${fmt(overdue.reduce((s, l) => s + bal(l), 0))}</b></div>
     <div class="stat" style="grid-column:span 2"><small>Expected Profit (interest)</small><b>${fmt(expected - lent)}</b></div>`;

  const q = $('#q').value.toLowerCase().trim();
  const rows = loans.filter(l => !q || l.name.toLowerCase().includes(q) || (l.phone || '').includes(q))
    .sort((a, b) => ({overdue:0, active:1, paid:2})[status(a)] - ({overdue:0, active:1, paid:2})[status(b)]);

  if (!rows.length) { $('#list').innerHTML = '<div class="empty">No loans yet. Tap "+ New Loan".</div>'; return; }

  $('#list').innerHTML = rows.map(l => {
    const st = status(l), pct = Math.min(100, Math.round(paid(l) / l.total * 100)), wa = waLink(l);
    const hist = l.pays.length
      ? l.pays.slice().reverse().map(p => `<div class="row" style="padding:4px 0;border-top:1px solid #eef2ef"><span>${fmtDate(p.d)} ${p.n ? '· ' + esc(p.n) : ''}</span><b>${fmt(p.a)}</b></div>`).join('')
      : '<div class="muted">No payments yet</div>';
    return `<div class="loan">
      <div class="row"><b>${esc(l.name)}</b><span class="tag ${st}">${st.toUpperCase()}</span></div>
      <div class="muted">${esc(l.phone || 'No phone')} · Lent ${fmt(l.principal)} at ${l.rate}%/month × ${l.months}</div>
      <div class="row" style="margin-top:8px"><div><div class="muted">Balance</div><b style="color:#b42318">${fmt(bal(l))}</b></div><div style="text-align:right"><div class="muted">Due</div><b>${fmtDate(l.due)}</b></div></div>
      <div class="bar"><i style="width:${pct}%"></i></div>
      <div class="muted">${pct}% paid of ${fmt(l.total)}</div>
      <details><summary>Payment history (${l.pays.length})</summary>${hist}</details>
      <div class="btns">
        ${st !== 'paid' ? `<button class="primary" style="width:auto;margin:0" data-act="pay" data-id="${l.id}">+ Payment</button>` : ''}
        ${wa && st !== 'paid' ? `<a class="wa" href="${wa}" target="_blank" rel="noopener">WhatsApp</a>` : ''}
        <button class="danger" data-act="del" data-id="${l.id}">Delete</button>
      </div></div>`;
  }).join('');
}

function preview() {
  const a = parseFloat($('#nAmt').value) || 0, r = parseFloat($('#nRate').value) || 0, m = parseInt($('#nMon').value) || 1;
  const i = a * r / 100 * m;
  $('#nPrev').innerHTML = `Interest: <b>${fmt(i)}</b><br>Total to repay: <b>${fmt(a + i)}</b><br>Due date: <b>${$('#nStart').value ? fmtDate(addMonths($('#nStart').value, m)) : '-'}</b>`;
}

$('#addBtn').onclick = () => { $('#fNew').reset(); $('#nRate').value = 10; $('#nMon').value = 1; $('#nStart').value = today(); preview(); $('#dNew').showModal(); };
['nAmt', 'nRate', 'nMon', 'nStart'].forEach(id => $('#' + id).addEventListener('input', preview));

$('#fNew').onsubmit = e => {
  e.preventDefault();
  const a = parseFloat($('#nAmt').value), r = parseFloat($('#nRate').value), m = parseInt($('#nMon').value), s = $('#nStart').value;
  if (!(a > 0) || !s) return;
  loans.push({ id: 'L' + Date.now(), name: $('#nName').value.trim(), phone: $('#nPhone').value.trim(),
    principal: a, rate: r, months: m, start: s, due: addMonths(s, m), total: a + a * r / 100 * m, pays: [] });
  $('#dNew').close(); save();
};

$('#list').onclick = e => {
  const b = e.target.closest('button[data-act]'); if (!b) return;
  const l = loans.find(x => x.id === b.dataset.id); if (!l) return;
  if (b.dataset.act === 'pay') {
    $('#pId').value = l.id; $('#pInfo').textContent = l.name + ' · Balance ' + fmt(bal(l));
    $('#pAmt').value = Math.round(bal(l)); $('#pAmt').max = Math.round(bal(l)); $('#pDate').value = today(); $('#pNote').value = '';
    $('#dPay').showModal();
  } else if (b.dataset.act === 'del') {
    if (confirm('Delete the loan for ' + l.name + '? This cannot be undone.')) { loans = loans.filter(x => x.id !== l.id); save(); }
  }
};

$('#fPay').onsubmit = e => {
  e.preventDefault();
  const l = loans.find(x => x.id === $('#pId').value); if (!l) return;
  const a = Math.min(parseFloat($('#pAmt').value), bal(l));
  if (!(a > 0)) return;
  l.pays.push({ d: $('#pDate').value, a: a, n: $('#pNote').value.trim() });
  $('#dPay').close(); save();
};

document.querySelectorAll('[data-close]').forEach(b => b.onclick = () => $('#' + b.dataset.close).close());
$('#q').oninput = render;
$('#bkBtn').onclick = () => $('#dBk').showModal();

$('#exp').onclick = () => {
  const url = URL.createObjectURL(new Blob([JSON.stringify(loans, null, 2)], {type: 'application/json'}));
  const a = document.createElement('a'); a.href = url; a.download = 'QuickLoan_Backup_' + today() + '.json';
  document.body.appendChild(a); a.click(); a.remove(); URL.revokeObjectURL(url);
};

$('#imp').onchange = e => {
  const f = e.target.files[0]; if (!f) return;
  const r = new FileReader();
  r.onload = () => {
    try {
      const d = JSON.parse(r.result);
      const ok = Array.isArray(d) && d.every(l => l && l.id && l.name && typeof l.total === 'number' && Array.isArray(l.pays));
      if (!ok) throw new Error('bad');
      if (!confirm('Replace all current data with this backup?')) return;
      loans = d; save(); $('#dBk').close();
    } catch (err) { alert('This is not a valid QuickLoan backup file.'); }
    e.target.value = '';
  };
  r.readAsText(f);
};

render();
</script>
</body>
</html>
