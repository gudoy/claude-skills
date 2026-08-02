---
name: add-to-candidatures
description: "Ajouter une entrée à la base Notion \"Candidatures\". Utiliser ce skill dès que l'utilisateur mentionne vouloir sauvegarder, ajouter, enregistrer ou archiver une fiche de poste à laquelle il a candidaté, une URL, un lien ou un contenu dans Notion. Également déclencher si l'utilisateur dit \"ajoute aux candidatures\", ou toute formulation similaire."
---

# Add to Candidatures (Notion)

Ce skill ajoute une candidature à la base Notion "Candidatures".

## Méthode d'insertion — deux chemins selon le contexte

### Chemin A — MCP Notion (prioritaire en session Cowork)
Utiliser l'outil `notion-create-pages` avec :
- `parent` : `{ "type": "database_id", "database_id": "<NOTION_CANDIDATURES_DB_ID>" }`
- `pages`  : tableau d'un objet `{ "properties": { ... } }`

⚠️ Différences de nommage MCP vs API brute :
- Le champ "URL" s'appelle **`userDefined:URL`** via le MCP (mais `URL` via l'API brute).
- Le MCP **refuse les options `Poste` inédites** — utiliser impérativement une valeur de la liste ci-dessous.

### Chemin B — Script Python (fallback si MCP non disponible)
Requiert `NOTION_TOKEN` dans l'environnement (voir section dédiée). Le script peut créer
des options `Poste` inédites car il passe par l'API brute.

## Config / Credentials (Chemin B uniquement)
- DATABASE_ID = lu depuis `NOTION_CANDIDATURES_DB_ID` (voir `.env.example`)
- TOKEN = intégration interne Notion, connectée à la base (menu "•••" → Connexions).
  ⚠️ Ne JAMAIS écrire le token en clair. Il est lu depuis la variable d'environnement
  NOTION_TOKEN au moment de l'exécution.
- Endpoint : POST https://api.notion.com/v1/pages
- Headers  : Authorization: Bearer <TOKEN> · Notion-Version: 2022-06-28 · Content-Type: application/json

## Définir le token (une fois, Chemin B)
- macOS/Linux (zsh/bash) : export NOTION_TOKEN="ta_cle"   (permanent : ligne dans ~/.zshrc puis `source ~/.zshrc`)
- Windows PowerShell     : [Environment]::SetEnvironmentVariable("NOTION_TOKEN","ta_cle","User")  (rouvrir le terminal)
- Windows cmd            : setx NOTION_TOKEN "ta_cle"   (rouvrir le terminal)

## Règles globales
- Au démarrage, confirmer avec : 'Application du skill "Add to Notion Candidatures"'
- Ignorer toute entrée existante en corbeille (in_trash / archived).
- Envoyer les noms d'options de select/status EXACTEMENT tels qu'ils existent dans la
  base (coquilles comprises) pour éviter les doublons.
- Un seul appel crée la candidature complète.

## Schéma réel de la base
| Champ           | Type API   | Nom MCP             | Forme JSON (API brute)                                     |
|-----------------|------------|---------------------|------------------------------------------------------------|
| Entreprise      | title      | Entreprise          | "title": [{ "text": { "content": <str> }}]                 |
| Poste           | select     | Poste               | "select": { "name": <str> }                                |
| Status          | status     | Status              | "status": { "name": <str> }                                |
| Progress        | select     | Progress            | "select": { "name": <str> }                                |
| Activité        | rich_text  | Activité            | "rich_text": [{ "text": { "content": <str> }}]             |
| URL             | rich_text  | **userDefined:URL** | "rich_text": [{ "text": { "content": <str> }}]  <- texte   |
| Via             | rich_text  | Via                 | "rich_text": [{ "text": { "content": <str> }}]             |
| Fiche de poste  | url        | Fiche de poste      | "url": <str>                                               |
| Date            | rich_text  | Date                | "rich_text": [{ "text": { "content": <str> }}]             |
| Steps           | rich_text  | Steps               | laisser vide (ne pas inclure la clé)                       |
| Commentaires    | rich_text  | Commentaires        | "rich_text": [{ "text": { "content": <str> }}]             |

## Valeurs par défaut / options fermées
- Status  -> défaut "En cours".
  Options RÉELLES : "En cours", "Success (non concluding)", "Success (offer)", "Archivée".
- Progress -> défaut "Postullé".  ⚠️ Orthographe exacte (deux L) dans la base — envoyer tel quel.

## Champ "Poste" — liste complète des options existantes
Choisir l'option la plus proche. Via MCP (Chemin A), toute valeur hors liste génère une erreur.
Via API brute (Chemin B), une valeur inédite est créée automatiquement.

"Senior Fullstack Engineer", "Lead Dev / CTO", "Lead JS Engineer", "Backend",
"Principal Frontend", "Software Architect", "Senior JS Engineer", "Tech Lead",
"Staff Software Engineer", "Senior Frontend Engineer", "Lead Frontend Engineer",
"Senior/Lead Dev", "Lead Backend Engineer", "Fouding Engineer",
"Staff/Lead Software Engineer", "Senior Software Engineer", "Enginering Lead Manager",
"Senior/Staff/Principal Software Engineer", "Lead Dev", "First Engineer",
"Staff/Senior Frontend Engineer", "Full Stack Engineer", "Senior/Staff Software Engineer",
"Fullstack Product Engineer", "Senior Fullstack Architect & AI Enabler",
"Staff Fullstack Engineer", "Founding Software Engineer", "Principal Fullstack Engineer",
"Senior Product Engineer", "Fullstack Software Engineer", "Frontend Engineer",
"Senior/Lead Fullstack Developer", "Senior Backend Engineer", "Software Engineer",
"Staff Frontend Engineer", "Senior Full Stack Engineer", "Software Engineer III Frontend",
"Staff Engineer", "Lead Fullstack Engineer", "Head of Engineering",
"Software Engineer (Integrations)", "Senior Software Developer", "Junior Full-Stack Engineer",
"Senior Platform Engineer", "Team Lead", "Frontend Developer (React)",
"Développeur Front-end", "Front End Tech Lead", "Product Engineer AI",
"Senior Full-Stack Software Engineer", "Senior Next.js Developer",
"Senior Customer Engineer", "Engineering Manager", "Software Engineer Platform",
"Fullstack Engineer"

## Champ "Entreprise" (titre)
- Détecter le nom dans la fiche. Tiers recruteur sans client nommé -> "??? via " + nom du tiers.
- Non précisé mais devinable -> tenter ; si incertain -> "???". Si deviné, l'indiquer dans "Commentaires".

## Champ "Activité" (texte)
- Déduire l'activité depuis la fiche et/ou le site, en s'inspirant des valeurs existantes.
- Tiers sans client nommé et non devinable -> laisser vide.

## Champ "URL" (texte — site entreprise)
- Site officiel via recherche web simple. Vide si entreprise non définie/devinée.
- Ne JAMAIS mettre l'URL du dépositaire tiers.

## Champ "Via" (texte)
- Entreprise nommée (par l'annonceur OU identifiée par recherche externe) -> nom du job board.
- Tiers pour entreprise non identifiée -> "NomDuTiers via NomDuJobBoard".
- Job boards : LinkedIn, WTTJ (Welcome to the Jungle), Collective.work, Jobgether, BAO...

## Champ "Fiche de poste" (url)
- URL de l'annonce, nettoyée des paramètres de tracking.
  - LinkedIn : garder uniquement https://www.linkedin.com/jobs/view/<ID>/

## Champ "Date" (texte)
- Date du jour, format "DD mois AAAA", mois FR minuscule (ex : 06 juillet 2026). Pas un date-picker.

## Champ "Steps" — laisser vide (ne pas inclure la clé).

## Champ "Commentaires" (texte)
- Fourchette salariale en K€ (ou TJM en €/j pour freelance) si précisée (sinon "non précisée").
- Rythme de télétravail si précisé. Toute déduction faite sur l'entreprise.

## Flux d'exécution
1. **Lire la fiche de poste.**
   - Utiliser `mcp__claude-in-chrome__get_page_text` sur l'onglet actif.
   - Si le contenu retourné est vide ou minimal (page LinkedIn sans session, SPA non rendue) :
     s'appuyer sur les informations disponibles dans le titre de page et l'URL, sans tenter de
     navigation supplémentaire vers la plateforme d'origine.

2. **Identifier l'entreprise cliente.**
   - Si l'annonceur est un job board / plateforme (Free-Work, WTTJ, Jobgether, etc.) ET que le
     nom du client final n'apparaît nulle part dans le texte extrait -> poser directement
     `Entreprise = "??? via <NomPlateforme>"` **sans lancer de recherche web**.
   - Si l'entreprise est nommée ou clairement devinable -> recherche web pour le site officiel.

3. **Choisir le champ "Poste"** parmi la liste des options existantes (section ci-dessus).

4. **Créer l'entrée** :
   - Chemin A (Cowork) : appeler `notion-create-pages` avec `database_id`.
   - Chemin B (fallback) : lancer le script Python ci-dessous.

5. Vérifier la réponse (id + url). En cas d'erreur d'option select -> corriger et relancer.

## Notes / pièges
- Coquilles à respecter : "Postullé" (Progress) ; "Fouding Engineer", "Enginering Lead Manager" (Poste).
- "URL" est du texte (rich_text), pas un type url. Via MCP, son nom est "userDefined:URL".
- "Status" est de type `status`, pas `select`.
- LinkedIn sans session connectée retourne un contenu minimal (titre + salaire + lieu).
  C'est suffisant pour remplir la fiche — ne pas tenter de naviguer vers Free-Work ou autre
  plateforme pour retrouver l'annonce originale.

## Script — add_candidature.py (Chemin B — portable Mac/PC, stdlib uniquement)

### Utilisation
    python add_candidature.py --entreprise "Fabulous" --poste "Senior Backend Engineer" \
      --date "06 juillet 2026" --via "LinkedIn" \
      --activite "App de bien-être & coaching self-care" \
      --url "https://www.thefabulous.co" \
      --fiche "https://www.linkedin.com/jobs/view/4413418313/" \
      --commentaires "Full remote, Paris. Salaire non précisé."
  (Sur macOS, utiliser `python3` si `python` n'est pas trouvé.)

```python
#!/usr/bin/env python3
"""add_candidature.py — Ajoute une candidature à la base Notion "Candidatures"."""
import os
import sys
import json
import argparse
import urllib.request
import urllib.error

DATABASE_ID = os.environ.get("NOTION_CANDIDATURES_DB_ID")
NOTION_VERSION = "2022-06-28"

def rich(txt):
    return {"rich_text": [{"text": {"content": txt}}]} if txt else {"rich_text": []}

def main():
    token = os.environ.get("NOTION_TOKEN")
    if not token:
        sys.exit("Erreur : variable d'environnement NOTION_TOKEN non définie.")
    if not DATABASE_ID:
        sys.exit("Erreur : variable d'environnement NOTION_CANDIDATURES_DB_ID non définie.")

    p = argparse.ArgumentParser()
    p.add_argument("--entreprise", required=True)
    p.add_argument("--poste", required=True)
    p.add_argument("--date", required=True, help='Format "DD mois AAAA", ex: 06 juillet 2026')
    p.add_argument("--via", default="")
    p.add_argument("--activite", default="")
    p.add_argument("--url", default="")            # site entreprise (champ texte)
    p.add_argument("--fiche", default="")          # URL annonce (champ url)
    p.add_argument("--commentaires", default="")
    p.add_argument("--status", default="En cours")
    p.add_argument("--progress", default="Postullé")  # orthographe exacte de la base
    args = p.parse_args()

    props = {
        "Entreprise": {"title": [{"text": {"content": args.entreprise}}]},
        "Poste":      {"select": {"name": args.poste}},
        "Status":     {"status": {"name": args.status}},
        "Progress":   {"select": {"name": args.progress}},
        "Activité":   rich(args.activite),
        "URL":        rich(args.url),
        "Via":        rich(args.via),
        "Date":       rich(args.date),
        "Commentaires": rich(args.commentaires),
    }
    if args.fiche:
        props["Fiche de poste"] = {"url": args.fiche}

    body = json.dumps({
        "parent": {"database_id": DATABASE_ID},
        "properties": props,
    }).encode("utf-8")

    req = urllib.request.Request(
        "https://api.notion.com/v1/pages",
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {token}",
            "Notion-Version": NOTION_VERSION,
            "Content-Type": "application/json",
        },
    )
    try:
        with urllib.request.urlopen(req) as resp:
            data = json.load(resp)
            print(f"OK — candidature créée : {data.get('id')}")
            print(f"URL : {data.get('url')}")
    except urllib.error.HTTPError as e:
        print(f"Erreur {e.code} : {e.read().decode('utf-8')}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    main()
```
