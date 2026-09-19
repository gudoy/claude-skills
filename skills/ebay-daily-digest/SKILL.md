---
name: "ebay-daily-digest"
description: "Crée et envoie automatiquement un digest quotidien à partir des alertes eBay reçues par Gmail. Déclencher ce skill dès que l'utilisateur mentionne eBay, digest eBay, alertes eBay, résultats eBay, annonces eBay, ou demande à consulter, fusionner ou résumer ses emails eBay du jour (ou d'une date donnée, ex. « sur les mails d'hier »). Également déclencher s'il dit « regarde mes alertes eBay », « crée le digest », « fais le digest eBay », ou toute formulation similaire. Le skill enchaîne recherche Gmail, extraction en un seul passage JS, fusion, envoi via la fenêtre de composition Gmail (grille illustrée dans le corps + pièce jointe), vérification du message réellement reçu, puis fermeture de tous les onglets navigateur qu'il a ouverts."
---

# eBay Daily Digest

Crée un digest quotidien à partir des alertes eBay reçues par Gmail, l'envoie par email à
l'utilisateur, puis referme proprement tout ce qui a été ouvert dans le navigateur.

Conçu pour tourner sans supervision (tâche planifiée) : aucune validation intermédiaire, aucune
trace laissée dans le navigateur.

## Configuration

```
GMAIL_MAILBOX_URL       = https://mail.google.com/mail/u/0/#inbox
SEARCH_QUERY            = from:ebay subject:(résultat OR résultats OR "Nouvelle Annonce" OR "Nouvelles Annonces") newer_than:1d
RECIPIENT               = doyer.guyllaume@gmail.com   # destinataire du digest (soi-même)
EMAIL_SUBJECT           = eBay daily digest
ATTACHMENT_FILENAME     = ebay-daily-digest.html

SUGGESTED_TITLE_PATTERN = "Vous aimerez peut-être aussi"

EXACT_ITEM_WIDTH_IN_PX     = 178
SUGGESTED_ITEM_WIDTH_IN_PX = 130
EBAY_BLUE               = #0654ba
```

**Période demandée.** Par défaut `newer_than:1d`. Si l'utilisateur vise un autre jour (« les mails
d'hier », « ceux du 18 »), remplacer par `after:AAAA/MM/JJ before:AAAA/MM/JJ+1` (bornes Gmail :
`after` inclus, `before` exclu) et mettre cette date dans le titre du digest.

## ⚠️ Format des objets eBay (changé le 18/09/2026)

Deux formats coexistent, la requête doit couvrir les deux :

- ancien : `<recherche> : N NOUVELLES ANNONCES !` / `: 1 NOUVELLE ANNONCE !`
- nouveau (depuis le 18/09/2026) : `<recherche> : N résultats` / `: 1 seul résultat`

Regex de nettoyage du nom de recherche (pour le regroupement) :
`/\s*:\s*(\d+\s+résultats?|1\s+seul\s+résultat|\d+\s+NOUVELLES?\s+ANNONCES?\s*!?)\s*$/i`.

Le corps des emails n'a pas changé : mêmes ancres `<a><img src="…ebayimg…"></a>`, même section
« Vous aimerez peut-être aussi ». Les titres `alt` sont tronqués par eBay avec `…` : c'est la
source, il n'existe pas de titre plus long dans l'email.

## ⚠️ Canal d'envoi : la fenêtre de composition Gmail, JAMAIS l'API du connecteur

Mesuré sur les envois du 13/09 (OK), 17/09 et 19/09 (KO) :

| Voie | Résultat |
|---|---|
| **API Gmail via le connecteur MCP** (`send_message` + `htmlBody` + `attachments`) | Le connecteur **supprime toutes les balises `<img>`** du corps, et **ré-encode la pièce jointe en ASCII** (`Résultats` → `R?sultats`, `—` → `?`). Corps sans images + accents cassés. **Interdit.** |
| **Fenêtre de composition Gmail, mode texte riche** + injection DOM (`createElement`) | HTML, images, liens et accents **conservés** (MIME `multipart/alternative` avec `text/html`, corps enveloppé dans le `<div dir="ltr">` de Gmail). **C'est la voie du skill.** |
| Fenêtre de composition Gmail avec **« Mode Texte brut » coché** | Le brouillon affiche les images à l'écran, mais le message part en `text/plain` uniquement. **Gmail mémorise ce réglage pour tous les nouveaux messages** : c'est ce qui a fait croire, du 16 au 19/09, que l'injection DOM « ne marchait plus ». |
| Collage : `ClipboardEvent('paste')` synthétique avec `text/html` | Gmail n'insère que le texte. Inutile. |

**Conséquence :**

- **le corps** = la grille illustrée complète (mêmes vignettes que la pièce jointe), injectée en DOM
  dans la fenêtre de composition ;
- **la pièce jointe** `ebay-daily-digest.html` = page autonome avec la même grille, **100 % ASCII**
  (accents en entités HTML `&eacute;`, `&mdash;`, `&#8230;`…) pour être insensible à tout
  ré-encodage ;
- **avant chaque envoi, vérifier dans le menu ⋮ de la fenêtre de composition que « Mode Texte brut »
  n'est PAS coché** (étape 4, point 3). C'est le contrôle qui manquait.

## ⚠️ Autres pièges (lire avant de commencer)

1. **Espaces insécables** : les objets Gmail utilisent des ` `. Toujours normaliser
   (`s.replace(/ /g,' ').trim()`) avant toute comparaison de sujet.
2. **`innerText` vide sur `.bog`** : dans la liste Gmail, `innerText` renvoie `""` pour les sujets.
   **Utiliser `textContent`.**
3. **Lignes fantômes** : `document.querySelectorAll('tr.zA')` renvoie aussi des lignes d'autres vues
   gardées en cache. **Toujours filtrer `tr.offsetParent !== null`.**
4. **Trusted Types** : Gmail bloque `innerHTML`, `DOMParser` et `execCommand('insertHTML')`.
   Construire le DOM avec `createElement` / `setAttribute` / `appendChild` uniquement.
5. **URL d'image** : trois formats coexistent (voir `resolve()` à l'étape 2) :
   - proxy Gmail → l'URL réelle est après le `#` : `src.slice(src.indexOf('#')+1)` ;
   - moderne → `https://i.ebayimg.com/images/g/{KEY}/s-l300/p.jpg` ;
   - service de rendu → `.../imageser/v1/image/render?...&imageUrl={URL encodée}` : décoder le
     paramètre `imageUrl` ;
   - legacy → `https://i.ebayimg.com/00/s/{DIMS}/z/{KEY}/$_57.JPG` (garder le chemin tel quel).
6. **URL item** : liens de tracking. Extraire l'ID et reconstruire `https://www.ebay.fr/itm/{id}`.
7. **Filtre de l'extension navigateur** : les valeurs retournées par `javascript_tool` qui
   ressemblent à une URL avec query string, à un JWT ou à du base64 sont remplacées par
   `[BLOCKED: ...]`. **Ne jamais faire transiter les URLs brutes ni du base64 par la valeur de
   retour.** Ne renvoyer que des composants inoffensifs (`id` numérique, `k` = clé image, `L` =
   chemin legacy) et reconstruire les URLs côté script. Les URLs peuvent en revanche être passées
   **en entrée** d'un `javascript_tool` sans problème.
8. **Prix manquants** : le template « 1 seul résultat » n'inclut souvent pas le prix dans le HTML.
   Le récupérer dans l'extrait (`snippet`) du connecteur Gmail : `Objet mis en vente à X EUR`.
   Prix ≥ 1 000 : l'espace des milliers est une espace fine (U+202F) — la regex prix doit accepter
   `[\s\u202f\u00a0]`.
9. **Champ destinataire Gmail** : après avoir tapé l'adresse, appuyer sur `Return` **puis attendre
   1 s** avant de cliquer sur « Objet ». Sinon la frappe suivante atterrit dans le champ « À ».
   Si la fenêtre se replie (clic malheureux sur son bandeau), la ré-agrandir en cliquant sur le
   bandeau en bas à droite ; le contenu tapé avant le repli peut être perdu : re-vérifier.
10. **Plusieurs fenêtres de composition** : `document.querySelector('input[name=subjectbox]')`
    peut cibler une autre fenêtre. Ne garder qu'une seule fenêtre ouverte à la fois.

Les références DOM (`ref_*`) deviennent obsolètes en changeant de vue Gmail : privilégier le JS.

## Étape 0 — Inventaire des onglets (à faire en tout premier)

1. `tabs_context_mcp` et **noter les `tabId` déjà ouverts** : liste blanche, à ne jamais fermer.
   (Si aucun groupe n'existe, `createIfEmpty: true` crée l'onglet de travail : il est **à nous**.)
2. Noter le `tabId` de l'onglet de travail. Un seul suffit.
3. Tenir à jour la liste des `tabId` créés par le skill. C'est ce qu'on referme à l'étape 5.

## Étape 1 — Recherche des alertes Gmail

Vérifier d'abord par l'API (`search_threads` du connecteur Gmail, `pageSize: 50`) avec
`SEARCH_QUERY` : rapide, et donne les `snippet` (prix de secours, piège 8). Si `{}` → aucune
alerte : le signaler, **enchaîner directement sur l'étape 5** et s'arrêter. Pas d'email vide.

Sinon, naviguer vers l'URL de recherche Gmail (query encodée, guillemets conservés), ex. :
`…/mail/u/0/#search/from%3Aebay+subject%3A(r%C3%A9sultat+OR+r%C3%A9sultats+OR+%22Nouvelle+Annonce%22+OR+%22Nouvelles+Annonces%22)+newer_than%3A1d`

- Attendre le rendu (la liste peut être vide pendant 5–7 s), puis compter les lignes **visibles**
  (`offsetParent !== null`) et lire les sujets via `textContent`. Le nombre doit correspondre à
  l'API.
- ⚠️ Ne jamais modifier, archiver, supprimer ni mettre à la corbeille les alertes eBay.

## Étape 2 — Extraction (un seul passage JavaScript)

Injecter les helpers **une fois** (ils persistent à travers la navigation SPA par hash).

```javascript
window.__extract = function(){
  const bs=document.querySelectorAll('.a3s'); const body=bs[bs.length-1]; if(!body) return null;
  const all=Array.from(body.querySelectorAll('*')); const posOf=new Map(); all.forEach((el,i)=>posOf.set(el,i));
  let sugPos=Infinity; all.forEach(el=>{if(el.children.length===0){const t=(el.textContent||'').trim();
    if(t.includes('Vous aimerez peut-') && posOf.get(el)<sugPos)sugPos=posOf.get(el);}});
  const cH=a=>{let h=a.getAttribute('href')||'';let m=h.match(/ebay\.[a-z.]+\/itm\/(\d+)/i);if(m)return 'https://www.ebay.fr/itm/'+m[1];
    let d;try{d=decodeURIComponent(h);}catch(e){d=h;}m=d.match(/ebay\.[a-z.]+\/itm\/(\d+)/i)||d.match(/\/itm\/(\d+)/)||d.match(/[?&]item=(\d+)/i);return m?'https://www.ebay.fr/itm/'+m[1]:null;};
  const cI=img=>{let s=img.getAttribute('src')||'';let h=s.indexOf('#');if(h>=0)s=s.slice(h+1);if(s.startsWith('//'))s='https:'+s;return s;};
  const fP=a=>{let el=a;for(let i=0;i<6&&el.parentElement;i++){let p=el.parentElement;let eb=0;
    p.querySelectorAll('img').forEach(im=>{if(/ebayimg/i.test(im.src))eb++;});if(eb>1)break;el=p;}
    const t=el.innerText||el.textContent||'';let m=t.match(/(\d{1,3}(?:[\s\u202f\u00a0.]\d{3})*[.,]\d{2})\s?(?:EUR|€)/);
    if(m)return m[1].replace(/[\s\u202f\u00a0]/g,' ').replace('.',',')+' EUR';m=t.match(/(\d+)\s?(?:EUR|€)/);return m?m[1]+' EUR':'';};
  const anchors=Array.from(body.querySelectorAll('a')).filter(a=>{const im=a.querySelector('img');return im&&/ebayimg/i.test(im.src);});
  const seen=new Set(); const items=[];
  anchors.forEach(a=>{const img=a.querySelector('img');const url=cH(a);if(!url||seen.has(url))return;seen.add(url);
    items.push({t:(posOf.get(a)>=sugPos)?'s':'e',i:cI(img),n:(img.getAttribute('alt')||'').replace(/^Image de\s*/,'').trim(),p:fP(a),u:url});});
  return items;
};

window.__searchHash = location.hash;
window.__openAndExtract = async function(needle){
  const norm=s=>(s||'').replace(/ /g,' ').trim();
  const rows=Array.from(document.querySelectorAll('tr.zA')).filter(tr=>tr.offsetParent!==null);
  let target=null, subj=null;
  for(const tr of rows){const el=tr.querySelector('.bog');if(el&&norm(el.textContent).includes(needle)){target=el;subj=norm(el.textContent);break;}}
  if(!target) return {error:'row not found', needle};
  target.click();
  for(let i=0;i<60;i++){await new Promise(r=>setTimeout(r,250));
    const bs=document.querySelectorAll('.a3s');
    if(bs.length && Array.from(bs[bs.length-1].querySelectorAll('img')).some(im=>/ebayimg/i.test(im.src))) break;}
  const items=window.__extract();
  let acc=JSON.parse(sessionStorage.getItem('__ebayDigest')||'{}'); acc[subj]=items;
  sessionStorage.setItem('__ebayDigest',JSON.stringify(acc));
  location.hash=window.__searchHash;
  for(let i=0;i<40;i++){await new Promise(r=>setTimeout(r,200)); if(document.querySelectorAll('tr.zA').length && !document.querySelectorAll('.a3s').length) break;}
  return {subj, count: items?items.length:0};
};
sessionStorage.setItem('__ebayDigest','{}');
```

Appeler `await window.__openAndExtract("<sous-chaîne de sujet unique>")` pour chaque email, par
lots de 6 à 12 dans un même appel d'outil (avec `await sleep(800)` entre deux). Utiliser une
sous-chaîne discriminante (ex. `"metroid, Jeux"` vs `"metroid fusion"`).

Puis **dédupliquer par `u`** et **réduire à des composants transférables** (piège 7) :

```javascript
const acc=JSON.parse(sessionStorage.getItem('__ebayDigest')||'{}');
let raw=0; const seen=new Set(); const items=[];
for(const arr of Object.values(acc)){ (arr||[]).forEach(it=>{ raw++; if(seen.has(it.u))return; seen.add(it.u); items.push(it); }); }
const resolve=s=>{s=String(s);
  let m=s.match(/\/images\/g\/([A-Za-z0-9~_-]+)\//); if(m) return {k:m[1]};
  let inner=null; try{ inner=new URL(s).searchParams.get('imageUrl'); }catch(e){}
  if(inner){ inner=decodeURIComponent(inner);
    m=inner.match(/\/images\/g\/([A-Za-z0-9~_-]+)\//); if(m) return {k:m[1]};
    try{ return {L:new URL(inner).pathname.replace(/^\//,'')}; }catch(e){}
  }
  try{ return {L:new URL(s).pathname.replace(/^\//,'')}; }catch(e){ return {}; }
};
const idOf=s=>{const m=String(s).match(/\/itm\/(\d+)/); return m?m[1]:null;};
const mapped=items.map(x=>Object.assign({n:x.n,p:x.p,t:x.t,id:idOf(x.u)},resolve(x.i)));
const groups=Object.entries(acc).map(([subj,arr])=>({
  s: subj.replace(/\s*:\s*(\d+\s+résultats?|1\s+seul\s+résultat|\d+\s+NOUVELLES?\s+ANNONCES?\s*!?)\s*$/i,'').trim(),
  ids:(arr||[]).map(x=>idOf(x.u))}));
({emails:Object.keys(acc).length, raw, uniq:mapped.length, e:mapped.filter(x=>x.t==='e').length, s:mapped.filter(x=>x.t==='s').length,
  noId:mapped.filter(x=>!x.id).length, noImg:mapped.filter(x=>!x.k&&!x.L).length, noPrice:mapped.filter(x=>!x.p).map(x=>x.id), mapped, groups})
```

**Vérification chiffrée obligatoire** avant de continuer :
`X emails → Y items bruts → Z après déduplication (E exacts + S suggérés)`, et
`0 item sans id, 0 item sans k ni L`. Compléter les `noPrice` avec les `snippet` de l'API
(piège 8). Si un email renvoie `count:0` ou `error`, le relancer.

> Il est normal que Z soit inférieur au total annoncé dans les objets (doublons dans un email, ou
> même item dans deux alertes). Le signaler dans le récapitulatif, ce n'est pas une perte.

- ⚠️ Extraction seulement, aucun email modifié ni déplacé.

## Étape 3 — Construction des données

Écrire les données dans un fichier `data.json` (`{date, items:[{id,k|L,n,p,t}], groups}`) puis
produire avec un script Python un `spec.json` :

```
{ date, n, ne, ns,
  ex: [{i: <url image>, u: <url item>, n: <titre>, p: <prix ou "—">}],   # t === 'e'
  su: [...] }                                                            # t === 's'
```

Reconstruction des URLs : `k` → `https://i.ebayimg.com/images/g/{k}/s-l300/p.jpg` ;
`L` → `https://i.ebayimg.com/{L}` ; `id` → `https://www.ebay.fr/itm/{id}`.

Le corps et la pièce jointe sont **générés dans la page** à partir de `spec` (étape 4) : aucun
HTML ni base64 ne transite par les valeurs de retour.

## Étape 4 — Envoi via la fenêtre de composition Gmail (ne pas demander de confirmation)

Le digest part vers la propre adresse de l'utilisateur, sans destinataire externe, et il est
reproductible : **envoyer directement**. Ne garder qu'une seule fenêtre de composition ouverte.

1. Naviguer vers `https://mail.google.com/mail/u/0/#inbox?compose=new`, attendre
   `input[name="subjectbox"]` + 1,5 s. Prendre une capture pour situer les champs.
2. **Destinataire et objet au clavier** (`computer`) : clic dans « À » → taper `RECIPIENT` →
   `Return` → **attendre 1 s** → clic dans « Objet » → taper `EMAIL_SUBJECT`. Zoomer sur l'en-tête
   pour vérifier : une seule puce destinataire, objet rempli.
3. **Mode texte riche obligatoire** : ouvrir le menu ⋮ (bas de la fenêtre, à gauche de la
   corbeille), capture, et **s'assurer que « Mode Texte brut » n'a pas de coche**. S'il est coché,
   cliquer dessus pour le désactiver (Gmail mémorise le réglage). Fermer le menu (`Escape`).
4. Cliquer dans le corps et taper un caractère (`x`) pour initialiser l'éditeur.
5. **Injection DOM** dans le corps (`div[aria-label="Corps du message"][contenteditable="true"]`),
   à partir de `spec` passé **en entrée** du `javascript_tool` (stocker dans `window.__SPEC`) :

```javascript
const el=(t,st,attrs)=>{const e=document.createElement(t); if(st) e.setAttribute('style',st); if(attrs) for(const k in attrs) e.setAttribute(k,attrs[k]); return e;};
const txt=(e,s)=>{e.appendChild(document.createTextNode(s)); return e;};
const root=el('div','font-family:Arial,Helvetica,sans-serif;color:#222');
root.appendChild(txt(el('div','font-size:20px;font-weight:bold;margin:0 0 4px'),'eBay daily digest — '+SPEC.date));
root.appendChild(txt(el('div','font-size:13px;color:#666;margin:0 0 16px'),SPEC.n+' annonces uniques ('+SPEC.ne+' exactes, '+SPEC.ns+' suggérées)'));
const grid=(title,list,w)=>{
  root.appendChild(txt(el('div','font-size:15px;font-weight:bold;margin:18px 0 12px;color:#555'),title));
  const g=el('div','');
  list.forEach(x=>{
    const c=el('div','display:inline-block;vertical-align:top;width:'+w+'px;min-width:'+w+'px;margin:0 14px 26px 0');
    const a=el('a','text-decoration:none',{href:x.u,target:'_blank'});
    a.appendChild(el('img','width:'+w+'px;height:'+w+'px;object-fit:contain;display:block;border:1px solid #e5e5e5;background:#f6f6f6',{src:x.i,alt:x.n,width:String(w),height:String(w)}));
    c.appendChild(a);
    c.appendChild(txt(el('a','display:block;margin-top:6px;font-size:13px;line-height:1.3;color:#0654ba;text-decoration:none',{href:x.u,target:'_blank'}),x.n));
    c.appendChild(txt(el('div','margin-top:4px;font-size:13px;font-weight:bold;color:#111'),x.p));
    g.appendChild(c);
  });
  root.appendChild(g);
};
grid('Résultats exacts ('+SPEC.ne+')',SPEC.ex,178);
if(SPEC.su.length) grid('Vous aimerez peut-être aussi ('+SPEC.ns+')',SPEC.su,130);
body.focus();
let wrap=body.querySelector('div[dir="ltr"]');
if(wrap){ while(wrap.firstChild) wrap.removeChild(wrap.firstChild); wrap.appendChild(root); }
else { while(body.firstChild) body.removeChild(body.firstChild); body.appendChild(root); }
body.dispatchEvent(new InputEvent('input',{bubbles:true,inputType:'insertFromPaste'}));
```

   La grille en `inline-block` (pas `flex`) est ce que Gmail rend en grille fluide pleine largeur
   à la réception.

6. **Pièce jointe**, générée dans la page en **ASCII pur** (échapper tout caractère > 0x7F en
   `&#NNNN;`), puis attachée via `input[type="file"]` :

```javascript
const esc=s=>s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;').replace(/[^\x00-\x7F]/g,ch=>'&#'+ch.codePointAt(0)+';');
const card=(x,w)=>'<div class="c" style="width:'+w+'px"><a href="'+x.u+'"><img src="'+x.i+'" alt=""></a><a class="t" href="'+x.u+'">'+esc(x.n)+'</a><div class="p">'+esc(x.p)+'</div></div>';
let ATT='<!DOCTYPE html><html lang="fr"><head><meta charset="utf-8"><title>eBay daily digest &mdash; '+SPEC.date+'</title>\n<style>body{font:13px Arial,sans-serif;color:#222;margin:20px}h1{font-size:20px}h2{font-size:15px;margin:28px 0 10px;color:#555}\n.g{display:flex;flex-wrap:wrap;gap:12px;row-gap:26px}.c{display:flex;flex-direction:column}.c img{width:100%;aspect-ratio:1;object-fit:contain;background:#f6f6f6;border:1px solid #e5e5e5}\n.t{color:#0654ba;text-decoration:none;margin-top:6px;line-height:1.3}.t:hover{text-decoration:underline}.p{font-weight:bold;margin-top:4px}</style></head><body>\n<h1>eBay daily digest &mdash; '+SPEC.date+'</h1><p>'+SPEC.n+' annonces uniques ('+SPEC.ne+' exactes, '+SPEC.ns+' sugg&eacute;r&eacute;es).</p>\n<h2>R&eacute;sultats exacts ('+SPEC.ne+')</h2><div class="g">'+SPEC.ex.map(x=>card(x,178)).join('')+'</div>';
if(SPEC.su.length) ATT+='<h2>Vous aimerez peut-&ecirc;tre aussi ('+SPEC.ns+')</h2><div class="g">'+SPEC.su.map(x=>card(x,130)).join('')+'</div>';
ATT+='\n</body></html>';
const fi=document.querySelector('input[type="file"]');
const dt=new DataTransfer(); dt.items.add(new File([ATT],'ebay-daily-digest.html',{type:'text/html'})); fi.files=dt.files; fi.dispatchEvent(new Event('change',{bubbles:true}));
// attendre que "ebay-daily-digest.html" apparaisse dans document.body.innerText (≤ 15 s), puis 2 s
```

   Retourner seulement des compteurs : `{imgs, links, attShown, attAscii}`. Attendu :
   `imgs == Z`, `links == 2·Z`, `attShown == true`, `attAscii == true`.

7. Capture de contrôle (destinataire, objet, vignettes visibles, pièce jointe listée en bas), puis
   cliquer sur **Envoyer**. Attendre 5 s : la fenêtre se ferme et « Message envoyé » s'affiche.

**Ne pas envoyer un digest vide ou sans pièce jointe.**

## Étape 4bis — Vérification du message **reçu** (obligatoire)

Une capture du brouillon ne prouve rien : **c'est le message arrivé qu'il faut relire.**

1. `search_threads` (connecteur Gmail) avec `subject:"eBay daily digest" newer_than:1h`,
   `THREAD_VIEW_METADATA_ONLY` : récupérer l'`id` du message le plus récent. Sa taille doit être
   de l'ordre de 80–120 Ko pour ~60 annonces (≈ 30 Ko = texte brut → échec).
2. `get_message` en `RAW` → le résultat est écrit dans un fichier : le décoder en Python
   (`base64.urlsafe_b64decode(raw+'==')`, module `email`) et vérifier :

```
partie text/html (hors pièce jointe) présente
  nb de <img> == Z, nb de <a  == 2·Z, "inline-block" == Z
  accents présents ("Résultats", "suggérées", "—")
pièce jointe ebay-daily-digest.html : nb de <img> == Z, "R&eacute;sultats" présent, 100 % ASCII
```

3. Ouvrir `#all/<id>` dans l'onglet de travail (par id, jamais en cherchant l'objet dans la liste),
   attendre `.a3s`, capture : grille pleine largeur, vignettes chargées, nom de la pièce jointe.

Si un seul contrôle échoue : **ne pas renvoyer en boucle**. Premier réflexe : ré-ouvrir une
composition et vérifier « Mode Texte brut » (étape 4, point 3). Mettre le message raté à la
corbeille (`trash_message`, réversible), signaler l'échec avec les comptes obtenus, et s'arrêter
après l'étape 5.

## Étape 5 — Fermeture des onglets (dans tous les cas)

S'exécute **quelle que soit l'issue** : envoi réussi, aucun email, ou erreur en cours de route.

1. Supprimer tout brouillon créé par le skill et non envoyé (Ctrl+Maj+D dans la fenêtre de
   composition, ou `list_drafts` + `delete_draft`).
2. `tabs_context_mcp` pour lister l'état courant.
3. `tabs_close_mcp` sur **tous les `tabId` de la liste tenue à l'étape 0**.
4. Ne fermer aucun onglet de la liste blanche, et ne pas fermer le navigateur.
5. Re-vérifier avec `tabs_context_mcp` qu'il ne reste aucun onglet créé par le skill.

## Récapitulatif final

`X emails traités → Z annonces (E exactes + S suggérées) → digest envoyé à RECIPIENT
(reçu vérifié : Z images, 2·Z liens, pièce jointe OK) → N onglets fermés.`

Mentionner les écarts entre le nombre annoncé dans les objets et le nombre d'items uniques, et
les prix complétés depuis les extraits API.
