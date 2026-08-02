---
name: ebay-daily-digest
description: Crée et envoie automatiquement un digest HTML quotidien à partir des alertes eBay reçues par Gmail. Déclencher ce skill dès que l'utilisateur mentionne eBay, digest eBay, alertes eBay, résultats eBay, annonces eBay, ou demande à consulter, fusionner ou résumer ses emails eBay du jour. Également déclencher s'il dit « regarde mes alertes eBay », « crée le digest », « fais le digest eBay », ou toute formulation similaire. Le skill enchaîne recherche Gmail, extraction HTML en un seul passage JS, fusion en digest, composition d'un email avec pièce jointe, envoi automatique sans confirmation, puis fermeture de tous les onglets navigateur qu'il a ouverts.
---

# eBay Daily Digest

Crée un digest HTML quotidien à partir des alertes eBay reçues par Gmail, l'envoie par email à
l'utilisateur, puis referme proprement tout ce qui a été ouvert dans le navigateur.

Ce skill est conçu pour tourner sans supervision (y compris en tâche planifiée) : il ne demande
aucune validation intermédiaire et ne laisse pas de trace dans le navigateur.

## Configuration

```
GMAIL_MAILBOX_URL       = https://mail.google.com/mail/u/0/#inbox
SEARCH_QUERY            = from:ebay subject:("Nouvelle Annonce" OR "Nouvelles Annonces") newer_than:1d
RECIPIENT               = ${DIGEST_RECIPIENT_EMAIL}   # destinataire du digest (soi-même)
EMAIL_SUBJECT           = eBay daily digest
ATTACHMENT_FILENAME     = ebay-daily-digest.html

RESULTS_TITLE_PATTERN   = "Chaque jour, de nouveaux résultats"
SUGGESTED_TITLE_PATTERN = "Vous aimerez peut-être aussi"

EXACT_ITEM_WIDTH_IN_PX     = 178
SUGGESTED_ITEM_WIDTH_IN_PX = 130
EBAY_BLUE               = #0654ba
```

## ⚠️ Notes de robustesse (pièges rencontrés — lire avant de commencer)

Ces quatre points font gagner l'essentiel du temps, ils évitent de redécouvrir les blocages :

1. **Espaces insécables** : les objets d'emails Gmail utilisent des `\u00A0` autour des `:` et chiffres.
   Toujours normaliser (`s.replace(/\u00A0/g,' ').trim()`) avant toute comparaison de texte de sujet.
2. **Trusted Types** : Gmail bloque `innerHTML` et `DOMParser`. Pour insérer le corps du brouillon,
   construire le DOM avec `document.createElement` / `setAttribute` / `appendChild` uniquement.
3. **Image réelle** : dans les `<img src>`, l'URL passe par un proxy `googleusercontent`.
   L'URL eBay réelle se trouve après le `#` → `src.slice(src.indexOf('#')+1)`.
4. **URL item** : les liens sont des redirections de tracking. Extraire l'ID et reconstruire
   `https://www.ebay.fr/itm/{id}` (regex sur `href` brut puis sur `decodeURIComponent(href)`).

Les références DOM (`ref_*`) deviennent obsolètes en changeant de vue Gmail : privilégier le JS.

## Étape 0 — Inventaire des onglets (à faire en tout premier)

L'utilisateur travaille dans son propre navigateur : fermer un onglet qui lui appartenait serait
intrusif, mais en laisser traîner un que le skill a créé est du bruit dont il devra s'occuper. Pour
distinguer les deux, il faut photographier l'état **avant** de toucher à quoi que ce soit — après
coup, c'est indécidable.

1. Appeler `tabs_context_mcp` et **noter les `tabId` déjà ouverts** : c'est la liste blanche, ces
   onglets ne doivent jamais être fermés.
2. Créer **un onglet dédié** pour le travail (`tabs_create_mcp`) et noter son `tabId`. Toute la
   navigation Gmail se faisant par changement de hash, un seul onglet suffit dans le cas normal.
3. Tenir à jour la liste des `tabId` créés par le skill au fil de l'exécution (Gmail peut en ouvrir un
   pour une composition détachée, un clic accidentel peut en ouvrir un vers eBay). Cette liste est ce
   qu'on referme à l'étape 5.

## Étape 1 — Recherche des alertes Gmail

Naviguer vers l'URL de recherche (query encodée, guillemets conservés) :
`…/mail/u/0/#search/from%3Aebay+subject%3A(%22Nouvelle+Annonce%22+OR+%22Nouvelles+Annonces%22)+newer_than%3A1d`

- Si aucun email trouvé → le dire à l'utilisateur, **enchaîner directement sur l'étape 5** (fermeture
  des onglets) et s'arrêter là. Pas d'email vide.
- Sinon → noter le nombre d'emails et continuer.
- ⚠️ Ne jamais modifier, archiver, supprimer ni mettre à la corbeille les emails (laisser tels quels).

## Étape 2 — Extraction (un seul passage JavaScript, pas d'ouverture visuelle)

Injecter les deux helpers ci-dessous **une fois** (ils persistent à travers la navigation SPA par hash).
`__extract()` lit le corps `.a3s` de l'email actuellement ouvert et renvoie les items.
`__openAndExtract(needle)` ouvre un email par sous-chaîne de sujet (insensible aux `\u00A0`), attend le rendu, extrait et accumule dans `sessionStorage['__ebayDigest']`.

```javascript
window.__extract = function(){
  const bs=document.querySelectorAll('.a3s'); const body=bs[bs.length-1]; if(!body) return null;
  const all=Array.from(body.querySelectorAll('*')); const posOf=new Map(); all.forEach((el,i)=>posOf.set(el,i));
  let sugPos=Infinity; all.forEach(el=>{if(el.children.length===0){const t=(el.textContent||'').trim();
    if(t.includes('Vous aimerez peut-être aussi')&&posOf.get(el)<sugPos)sugPos=posOf.get(el);}});
  const cH=a=>{let h=a.getAttribute('href')||'';let m=h.match(/ebay\.[a-z.]+\/itm\/(\d+)/i);if(m)return 'https://www.ebay.fr/itm/'+m[1];
    let d;try{d=decodeURIComponent(h);}catch(e){d=h;}m=d.match(/ebay\.[a-z.]+\/itm\/(\d+)/i)||d.match(/\/itm\/(\d+)/)||d.match(/[?&]item=(\d+)/i);return m?'https://www.ebay.fr/itm/'+m[1]:null;};
  const cI=img=>{let s=img.getAttribute('src')||'';let h=s.indexOf('#');if(h>=0)s=s.slice(h+1);if(s.startsWith('//'))s='https:'+s;return s;};
  const fP=a=>{let el=a;for(let i=0;i<6&&el.parentElement;i++){let p=el.parentElement;let eb=0;
    p.querySelectorAll('img').forEach(im=>{if(/ebayimg/i.test(im.src))eb++;});if(eb>1)break;el=p;}
    const t=el.innerText||'';let m=t.match(/(\d{1,3}(?:[ .]\d{3})*[.,]\d{2})\s?(?:EUR|€)/);
    if(m)return m[1].replace(/\s/g,'').replace('.',',')+' EUR';m=t.match(/(\d+)\s?(?:EUR|€)/);return m?m[1]+' EUR':'';};
  const anchors=Array.from(body.querySelectorAll('a')).filter(a=>{const im=a.querySelector('img');return im&&/ebayimg/i.test(im.src);});
  const seen=new Set(); const items=[];
  anchors.forEach(a=>{const img=a.querySelector('img');const url=cH(a);if(!url||seen.has(url))return;seen.add(url);
    items.push({type:(posOf.get(a)>=sugPos)?'suggested':'exact',img_src:cI(img),img_alt:img.getAttribute('alt')||'',
      title:(img.getAttribute('alt')||'').replace(/^Image de\s*/,'').trim(),price:fP(a),url});});
  return items;
};

window.__searchHash = location.hash;
window.__openAndExtract = async function(needle){
  const norm=s=>s.replace(/\u00a0/g,' ').trim();
  const rows=Array.from(document.querySelectorAll('tr.zA'));
  let target=null, subj=null;
  for(const tr of rows){const el=tr.querySelector('.bog');if(el&&norm(el.innerText).includes(needle)){target=el;subj=norm(el.innerText);break;}}
  if(!target) return {error:'row not found', needle};
  target.click();
  for(let i=0;i<60;i++){await new Promise(r=>setTimeout(r,250));
    const bs=document.querySelectorAll('.a3s');
    if(bs.length && Array.from(bs[bs.length-1].querySelectorAll('img')).some(im=>/ebayimg/i.test(im.src))) break;}
  const items=window.__extract();
  let acc=JSON.parse(sessionStorage.getItem('__ebayDigest')||'{}'); acc[subj]=items;
  sessionStorage.setItem('__ebayDigest',JSON.stringify(acc));
  location.hash=window.__searchHash; // revenir à la liste
  for(let i=0;i<40;i++){await new Promise(r=>setTimeout(r,200)); if(document.querySelectorAll('tr.zA').length && !document.querySelectorAll('.a3s').length) break;}
  return {subj, count: items?items.length:0};
};
```

Puis, pour chaque email trouvé, appeler `await window.__openAndExtract("<sous-chaîne de sujet unique>")`.
Regrouper 3–4 appels par exécution d'outil pour aller vite. Utiliser une sous-chaîne discriminante
(ex. `"switch : 1"` vs `"switch, Jeux : 1"`).

**Vérification chiffrée obligatoire** avant de continuer :
`X emails → Y items bruts → Z items après déduplication (E exacts + S suggérés)`.
Si un email renvoie `count:0` ou `error`, le relancer avant de passer à la suite.

- Déduplication finale par `url` sur l'ensemble des emails.
- ⚠️ Extraction seulement, aucun email modifié ni déplacé.

## Étape 3a — Construction des deux versions HTML

À partir des items dédupliqués, générer **deux** HTML et les stocker en `sessionStorage` :

- `__ebayHtml` : page autonome complète (flexbox `flex-wrap`, `<style>` en tête) → sert de **pièce jointe**.
- `__ebayEmailBody` : version **compatible Gmail** → sert de **corps d'email**.

Règles communes aux deux grilles :
- Grille 1 « exact » à `EXACT_ITEM_WIDTH_IN_PX`, grille 2 « suggested » à `SUGGESTED_ITEM_WIDTH_IN_PX`.
- Image + titre = liens `target="_blank" rel="noopener"` vers l'URL eBay de l'item.
- Titres en `EBAY_BLUE`, non tronqués ; prix affiché sous le titre.
- ~26px d'espace vertical entre les lignes pour aérer.

Spécifique corps Gmail (pas de flexbox) : émuler la grille avec
`display:inline-block; vertical-align:top; width:{w}px; min-width:{w}px; margin:0 14px 26px 0;`
et **tous les styles en inline** (Gmail supprime les blocs `<style>`).

## Étape 3b — Composition du brouillon

1. Ouvrir une fenêtre de composition (`#inbox?compose=new`).
2. Renseigner **To** = `RECIPIENT`, **Objet** = `EMAIL_SUBJECT`.
3. Insérer le corps : récupérer la zone éditable
   (`div[aria-label="Corps du message"][contenteditable="true"]`), vider ses enfants, puis
   **reconstruire le DOM via `createElement`** (Trusted Types interdit `innerHTML`), et
   déclencher `dispatchEvent(new InputEvent('input',{bubbles:true}))`.
4. Attacher le fichier : créer `new File([__ebayHtml], ATTACHMENT_FILENAME, {type:'text/html'})`,
   l'ajouter à un `DataTransfer`, l'assigner à `input[type="file"][name="Filedata"].files`,
   puis `dispatchEvent(new Event('change',{bubbles:true}))`.
5. Prendre une capture d'écran du brouillon — elle sert de contrôle avant envoi à l'étape 4.

## Étape 4 — Envoi automatique (ne pas demander de confirmation)

Le digest part vers la propre adresse de l'utilisateur, sans destinataire externe, et il est
reproductible : demander un feu vert n'apporte aucune sécurité et empêche le skill de tourner en
tâche planifiée. **Envoyer directement, sans poser de question dans le chat et sans attendre de
« oui ».**

1. Contrôle silencieux sur la capture de l'étape 3b : destinataire = `RECIPIENT`, objet =
   `EMAIL_SUBJECT`, chip de pièce jointe présent, au moins un item dans le corps.
   Si un de ces points manque, corriger le brouillon puis refaire le contrôle. **Ne pas envoyer un
   digest vide ou sans pièce jointe** : c'est le seul cas où il vaut mieux s'arrêter et le signaler,
   parce qu'un email inutilisable coûte plus à l'utilisateur qu'une exécution interrompue.
2. Cliquer sur **Envoyer** (ou `Ctrl+Entrée` dans la fenêtre de composition).
3. Confirmer l'envoi : la fenêtre de composition se ferme et Gmail affiche le toast « Message envoyé ».
   Si le toast n'apparaît pas au bout de quelques secondes, vérifier la présence du message dans
   `#sent` avant de conclure — ne jamais recliquer à l'aveugle, ça crée des doublons.

## Étape 5 — Fermeture des onglets (dans tous les cas)

Le skill doit rendre le navigateur dans l'état où il l'a trouvé. Cette étape s'exécute **quel que soit
l'issue** : envoi réussi, aucun email trouvé, ou erreur en cours de route. C'est justement quand ça se
passe mal qu'on oublie de nettoyer, donc y penser avant de rédiger le message final.

1. Fermer la fenêtre de composition si elle est encore ouverte (Gmail la ferme normalement après envoi).
2. Appeler `tabs_context_mcp` pour lister l'état courant.
3. Fermer avec `tabs_close_mcp` **tous les onglets dont le `tabId` figure dans la liste tenue à
   l'étape 0** — onglet de travail Gmail, composition détachée, onglets eBay ouverts par mégarde.
4. Ne fermer aucun onglet de la liste blanche, et ne pas fermer le navigateur lui-même.
5. Re-vérifier avec `tabs_context_mcp` qu'il ne reste aucun onglet créé par le skill.

## Récapitulatif final

Terminer par un message court :
`X emails traités → Z items (E exacts + S suggérés) → digest envoyé à RECIPIENT → N onglets fermés.`
