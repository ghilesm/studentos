<!DOCTYPE html>
<html lang="fr"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Mon Pilote Étudiant</title>
<style>
:root{--bg:#f7f8fa;--card:#fff;--txt:#1f2937;--mut:#6b7280;--acc:#2563eb;--ok:#16a34a;--warn:#d97706;--bad:#dc2626}
*{box-sizing:border-box}body{margin:0;font-family:system-ui,sans-serif;background:var(--bg);color:var(--txt)}
header{background:#fff;border-bottom:1px solid #e5e7eb;padding:10px 16px;display:flex;gap:6px;flex-wrap:wrap;position:sticky;top:0}
header button{border:0;background:none;padding:8px 12px;border-radius:8px;cursor:pointer;color:var(--mut);font-size:1em}
header button.on{background:#eff6ff;color:var(--acc);font-weight:600}
main{max-width:900px;margin:auto;padding:16px}
.card{background:var(--card);border:1px solid #e5e7eb;border-radius:12px;padding:16px;margin-bottom:12px}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(190px,1fr));gap:12px}
.big{font-size:1.5em;font-weight:700}.mut{color:var(--mut);font-size:.9em}
input,select,textarea{padding:6px;border:1px solid #d1d5db;border-radius:6px;margin:2px;font:inherit}
.btn{background:var(--acc);color:#fff;border:0;padding:6px 11px;border-radius:6px;cursor:pointer;margin:1px}
.btn.g{background:#e5e7eb;color:var(--txt)}
table{width:100%;border-collapse:collapse}td,th{padding:6px;border-bottom:1px solid #f0f0f0;text-align:left;font-size:.92em}
.ok{color:var(--ok)}.warn{color:var(--warn)}.bad{color:var(--bad)}
.tab{display:none}.tab.on{display:block}h3{margin-top:0}
</style></head><body>
<header id="nav"></header>
<main>
<section class="tab" id="dash"></section><section class="tab" id="budget"></section>
<section class="tab" id="planning"></section><section class="tab" id="reglages"></section>
</main>
<script>
const KEY='pilote-etudiant-v1';
const DEF={settings:{tauxNet:9.40,jourPaie:3,bourse:0,jourBourse:5,moisBourse:[9,10,11,12,1,2,3,4,5,6],maxH:15,
  aides:[{nom:'APL',montant:0,jour:5}],
  charges:[{nom:'Loyer',montant:0,jour:5},{nom:'Forfait téléphone',montant:0,jour:10},{nom:'Abonnement transport',montant:0,jour:1}]},
  shifts:[],tx:[]};
let S=JSON.parse(localStorage.getItem(KEY)||'null')||structuredClone(DEF);
const $=id=>document.getElementById(id);
const save=()=>{localStorage.setItem(KEY,JSON.stringify(S));render()};
const eur=n=>n.toLocaleString('fr-FR',{style:'currency',currency:'EUR'});
const esc=s=>String(s).replace(/[&<>"]/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'})[c]);
const iso=d=>{d=new Date(d);return d.getFullYear()+'-'+String(d.getMonth()+1).padStart(2,'0')+'-'+String(d.getDate()).padStart(2,'0')};
const ym=s=>s.slice(0,7);
const uid=()=>Math.random().toString(36).slice(2,9);
const h=s=>{const[a,b]=s.debut.split(':').map(Number),[c,d]=s.fin.split(':').map(Number);return(c*60+d-a*60-b)/60};
const monday=d=>{d=new Date(typeof d=='string'?d+'T12:00':d);d.setDate(d.getDate()-(d.getDay()+6)%7);return iso(d)};
const cmp=(a,b)=>(a.date+a.debut)<(b.date+b.debut)?-1:1;
const fmtD=d=>new Date(d+'T12:00').toLocaleDateString('fr-FR',{weekday:'long',day:'numeric',month:'long'});
const conflict=s=>S.shifts.some(o=>o.id!=s.id&&o.date==s.date&&o.debut<s.fin&&s.debut<o.fin);
const TYPES=['Drive','Cours','Révision','Autre'],ICO={Drive:'🛒',Cours:'🎓',Révision:'📚',Autre:'•'};
const CATS=['Courses','Transport','Loisirs','Études','Santé','Fixe','Salaire','Bourse','Aides','Autre'];
const TABS=[['dash','Tableau de bord',vDash],['budget','Budget',vBudget],['planning','Planning',vPlan],['reglages','Réglages',vReg]];
let cur='dash',bm=ym(iso(new Date()));

function render(){
  $('nav').innerHTML=TABS.map(([k,l])=>`<button class="${k==cur?'on':''}" onclick="cur='${k}';render()">${l}</button>`).join('');
  TABS.forEach(([k,,f])=>{$(k).classList.toggle('on',k==cur);if(k==cur)f()});
}

/* ---------- Semi-automatisation : génère les récurrents "à valider" ---------- */
function genMonth(){
  const m=bm,[Y,M]=m.split('-').map(Number),st=S.settings;
  const add=(key,lib,montant,jour,cat)=>{if(S.tx.some(t=>t.key==key))return;
    S.tx.push({id:uid(),key,date:`${m}-${String(Math.min(jour,28)).padStart(2,'0')}`,lib,montant,cat,pending:true})};
  if(st.bourse>0&&st.moisBourse.includes(M))add('b'+m,'Bourse CROUS',st.bourse,st.jourBourse,'Bourse');
  st.aides.forEach(a=>a.montant>0&&add('a'+a.nom+m,a.nom,a.montant,a.jour,'Aides'));
  st.charges.forEach(c=>c.montant>0&&add('c'+c.nom+m,c.nom,-c.montant,c.jour,'Fixe'));
  const prev=ym(iso(new Date(Y,M-2,1)));
  const hp=S.shifts.filter(s=>s.type=='Drive'&&ym(s.date)==prev).reduce((a,s)=>a+h(s),0);
  if(hp>0)add('s'+m,`Salaire Leclerc Drive (estimé : ${hp} h en ${prev})`,Math.round(hp*st.tauxNet*100)/100,st.jourPaie,'Salaire');
  save();
}

/* ---------- Tableau de bord ---------- */
function vDash(){
  const d=new Date(),m=ym(iso(d)),tx=S.tx.filter(t=>ym(t.date)==m),st=S.settings;
  const sum=f=>tx.filter(f).reduce((a,t)=>a+t.montant,0);
  const inV=sum(t=>!t.pending&&t.montant>0),outV=-sum(t=>!t.pending&&t.montant<0);
  const inP=sum(t=>t.pending&&t.montant>0),outP=-sum(t=>t.pending&&t.montant<0);
  const reste=inV+inP-outV-outP,jr=new Date(d.getFullYear(),d.getMonth()+1,0).getDate()-d.getDate()+1;
  const wk=monday(d),hT=t=>S.shifts.filter(s=>s.type==t&&monday(s.date)==wk).reduce((a,s)=>a+h(s),0);
  const hD=hT('Drive'),hC=hT('Cours'),hR=hT('Révision');
  const cls=hD>st.maxH?'bad':hD>st.maxH*.8?'warn':'ok';
  const up=S.shifts.filter(s=>s.date>=iso(d)).sort(cmp);
  const al=[];
  if(!tx.some(t=>t.key))al.push('⚡ Les récurrents du mois ne sont pas générés (onglet Budget).');
  const p=S.tx.filter(t=>t.pending).length;if(p)al.push(`🕐 ${p} opération(s) à valider.`);
  if(hD>st.maxH)al.push(`🛒 Tu dépasses ton plafond Drive (${hD} h > ${st.maxH} h).`);
  if(hD>0&&hR<hD/2)al.push('📚 Révisions < moitié de tes heures Drive cette semaine.');
  if(up.some(conflict))al.push('⚠ Chevauchement Drive / cours dans ton planning.');
  $('dash').innerHTML=`<div class="grid">
   <div class="card"><div class="mut">Reste à vivre (mois)</div><div class="big ${reste<0?'bad':'ok'}">${eur(reste)}</div><div class="mut">≈ ${eur(Math.max(reste,0)/jr)} / jour (${jr} j)</div></div>
   <div class="card"><div class="mut">Encaissé / dépensé</div><div class="big">${eur(inV)} / ${eur(outV)}</div><div class="mut">Prévu : +${eur(inP)} / −${eur(outP)}</div></div>
   <div class="card"><div class="mut">Drive cette semaine</div><div class="big ${cls}">${hD} h / ${st.maxH} h</div></div>
   <div class="card"><div class="mut">Études cette semaine</div><div class="big">${hC+hR} h</div><div class="mut">dont ${hR} h de révisions</div></div></div>
   ${al.length?`<div class="card">${al.map(a=>`<div>${a}</div>`).join('')}</div>`:''}
   <div class="card"><h3>Prochains créneaux</h3>${up.slice(0,6).map(s=>`<div class="${conflict(s)?'bad':''}">${ICO[s.type]} ${fmtD(s.date)} · ${s.debut}–${s.fin} · ${s.type}</div>`).join('')||'<span class="mut">Rien de prévu</span>'}</div>`;
}

/* ---------- Budget ---------- */
function vBudget(){
  const tx=S.tx.filter(t=>ym(t.date)==bm).sort((a,b)=>a.date<b.date?-1:1),cats={};
  tx.filter(t=>t.montant<0&&!t.pending).forEach(t=>cats[t.cat]=(cats[t.cat]||0)-t.montant);
  $('budget').innerHTML=`<div class="card"><input type="month" value="${bm}" onchange="bm=this.value;render()">
   <button class="btn" onclick="genMonth()">⚡ Générer les récurrents du mois</button></div>
   <div class="card"><h3>Ajouter</h3><input type="date" id="td" value="${iso(new Date())}"><input id="tl" placeholder="Libellé">
   <input type="number" step="0.01" id="tm" placeholder="Montant" style="width:100px"><select id="ts"><option value="d">Dépense</option><option value="r">Revenu</option></select>
   <select id="tc">${CATS.map(c=>`<option>${c}</option>`).join('')}</select><button class="btn" onclick="addTx()">+</button></div>
   <div class="card"><table><tr><th>Date</th><th>Libellé</th><th>Cat.</th><th>Montant</th><th></th></tr>
   ${tx.map(t=>`<tr style="${t.pending?'opacity:.6':''}"><td>${t.date.slice(8)}/${t.date.slice(5,7)}</td><td>${esc(t.lib)}${t.pending?' <span class="warn">(à valider)</span>':''}</td><td>${t.cat}</td>
   <td class="${t.montant<0?'bad':'ok'}">${eur(t.montant)}</td><td>${t.pending?`<button class="btn" onclick="valider('${t.id}')">✓</button>`:''}<button class="btn g" onclick="delTx('${t.id}')">✕</button></td></tr>`).join('')||'<tr><td colspan=5 class="mut">Aucune opération</td></tr>'}</table></div>
   <div class="card"><h3>Dépenses par catégorie</h3>${Object.entries(cats).sort((a,b)=>b[1]-a[1]).map(([c,v])=>`<div>${c} : <b>${eur(v)}</b></div>`).join('')||'<span class="mut">—</span>'}</div>`;
}
function addTx(){const m=+$('tm').value;if(!m||!$('tl').value)return;
  S.tx.push({id:uid(),date:$('td').value,lib:$('tl').value,montant:$('ts').value=='d'?-Math.abs(m):Math.abs(m),cat:$('tc').value,pending:false});save()}
function valider(id){const t=S.tx.find(x=>x.id==id),v=prompt('Montant réel (€) :',Math.abs(t.montant));if(v===null)return;
  const n=parseFloat(v.replace(',','.'));if(isNaN(n))return;t.montant=Math.sign(t.montant)*n;t.pending=false;save()}
function delTx(id){S.tx=S.tx.filter(t=>t.id!=id);save()}

/* ---------- Planning ---------- */
function vPlan(){
  const t=iso(new Date()),e=new Date();e.setDate(e.getDate()+21);
  const list=S.shifts.filter(s=>s.date>=t&&s.date<=iso(e)).sort(cmp),byDay={};
  list.forEach(s=>(byDay[s.date]=byDay[s.date]||[]).push(s));
  const hm=S.shifts.filter(s=>s.type=='Drive'&&ym(s.date)==ym(t)).reduce((a,s)=>a+h(s),0);
  $('planning').innerHTML=`<div class="card"><h3>Ajouter un créneau</h3><input type="date" id="sd" value="${t}">
   <input type="time" id="sa" value="08:00"><input type="time" id="sb" value="12:00"><select id="sy">${TYPES.map(x=>`<option>${x}</option>`).join('')}</select>
   répéter <input type="number" id="sn" value="1" min="1" style="width:60px"> sem. <button class="btn" onclick="addShift()">+</button></div>
   <div class="card"><b>Drive ce mois :</b> ${hm} h → paie estimée ${eur(hm*S.settings.tauxNet)} <span class="mut">(versée le mois suivant)</span></div>
   <div class="card"><h3>3 prochaines semaines</h3>${Object.entries(byDay).map(([d,ss])=>`<div style="margin:8px 0"><b>${fmtD(d)}</b>
   ${ss.map(s=>`<div style="margin-left:12px" class="${conflict(s)?'bad':''}">${ICO[s.type]} ${s.debut}–${s.fin} ${s.type} ${conflict(s)?'⚠ chevauchement':''}
   <button class="btn g" onclick="delShift('${s.id}')">✕</button></div>`).join('')}</div>`).join('')||'<span class="mut">Rien de prévu</span>'}</div>`;
}
function addShift(){const d=$('sd').value,a=$('sa').value,b=$('sb').value,ty=$('sy').value,n=+$('sn').value||1;
  if(!d||!a||!b||b<=a)return alert('Créneau invalide');
  for(let i=0;i<n;i++){const x=new Date(d+'T12:00');x.setDate(x.getDate()+7*i);S.shifts.push({id:uid(),date:iso(x),debut:a,fin:b,type:ty})}save()}
function delShift(id){S.shifts=S.shifts.filter(s=>s.id!=id);save()}

/* ---------- Réglages ---------- */
function vReg(){const st=S.settings,L=a=>a.map(x=>`${x.nom};${x.montant};${x.jour}`).join('\n');
  $('reglages').innerHTML=`<div class="card"><h3>Revenus</h3>
   <label>Taux horaire NET Leclerc (€) <input type="number" step="0.01" id="r1" value="${st.tauxNet}"></label>
   <label>Jour de paie <input type="number" id="r2" value="${st.jourPaie}" style="width:60px"></label><br>
   <label>Bourse mensuelle (€) <input type="number" id="r3" value="${st.bourse}"></label>
   <label>Jour de versement <input type="number" id="r4" value="${st.jourBourse}" style="width:60px"></label><br>
   <label>Mois de versement <input id="r5" value="${st.moisBourse.join(',')}"></label><br>
   <label>Plafond heures Drive / semaine <input type="number" id="r6" value="${st.maxH}" style="width:60px"></label></div>
   <div class="card"><h3>Aides récurrentes <span class="mut">(Nom;montant;jour)</span></h3><textarea id="r7" rows="3" style="width:100%">${L(st.aides)}</textarea>
   <h3>Charges fixes</h3><textarea id="r8" rows="5" style="width:100%">${L(st.charges)}</textarea>
   <button class="btn" onclick="saveReg()">Enregistrer</button></div>
   <div class="card"><h3>Sauvegarde</h3><button class="btn g" onclick="exp()">Exporter (.json)</button> <input type="file" accept=".json" onchange="imp(this)"></div>`}
const parseL=v=>v.split('\n').map(l=>l.split(';')).filter(p=>p[0]&&p[1]).map(([n,m,j])=>({nom:n.trim(),montant:+m.replace(',','.')||0,jour:+j||1}));
function saveReg(){const st=S.settings;st.tauxNet=+$('r1').value;st.jourPaie=+$('r2').value;st.bourse=+$('r3').value;st.jourBourse=+$('r4').value;
  st.moisBourse=$('r5').value.split(',').map(Number).filter(Boolean);st.maxH=+$('r6').value;st.aides=parseL($('r7').value);st.charges=parseL($('r8').value);save();alert('Enregistré ✔')}
function exp(){const a=document.createElement('a');a.href=URL.createObjectURL(new Blob([JSON.stringify(S,null,2)],{type:'application/json'}));a.download='pilote-etudiant.json';a.click()}
function imp(i){const r=new FileReader();r.onload=()=>{S=JSON.parse(r.result);save()};r.readAsText(i.files[0])}

render();
</script></body></html>
