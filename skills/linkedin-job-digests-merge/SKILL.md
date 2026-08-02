---
name: linkedin-job-digests-merge
description: "Fusionne plusieurs digests LinkedIn (produits par le skill linkedin-job-digest) en un seul email dédupliqué. Déclencher dès que l'utilisateur demande de fusionner, regrouper, consolider ou dédupliquer ses digests / présélections LinkedIn, dit \"fusionne mes digests\", \"regroupe mes présélections\", \"fais-moi un seul mail avec toutes les offres\", \"dédoublonne les offres\", ou mentionne plusieurs digests LinkedIn à réunir."
---

# Fusion de digests LinkedIn

Fusionne plusieurs digests produits par `linkedin-job-digest` en **un seul email**, dédupliqué, envoyé à `${DIGEST_RECIPIENT_EMAIL}`.

## Règles transversales

- **Seules les offres ✅ (pertinentes) et ⚠️ (à vérifier) sont reprises.** Les ❌ et tout ce que les digests sources ont écarté sont ignorés sans être mentionnés.
- **Aucune justification.** Pas de "pourquoi ça matche", pas de points d'attention, pas de commentaire. Le lien et les faits bruts suffisent.
- **Chaque offre a un lien cliquable.** Une offre sans lien exploitable n'est pas listée.
- **Ne pas ré-évaluer les offres** ni visiter les pages LinkedIn : les digests sources ont déjà fait ce travail. Ce skill ne fait que fusionner.
- **Ne pas improviser en silence** : si un outil manque ou échoue (Gmail indisponible, envoi refusé, aucun digest trouvé), arrêter l'étape et l'expliquer en fin de rapport avec 2-3 alternatives.
- **Refermer les onglets** : si le navigateur a dû être utilisé, fermer (`tabs_close_mcp`) tous les onglets ouverts pour cette exécution avant de terminer.

## Étape 1 — Identifier les digests à fusionner

Si l'utilisateur précise les digests (dates, objets, IDs, période) : utiliser exactement ceux-là.

Sinon, chercher via Gmail dans **les brouillons ET les emails reçus** :
- `list_drafts` puis filtrer sur l'objet contenant `LinkedIn — présélection`
- `search_threads` avec `subject:"LinkedIn — présélection"` (défaut : 7 derniers jours)

Récupérer le contenu complet (`get_message` / `get_thread`). Garder de côté les `id` de brouillons et `threadId` récupérés — ils servent à l'étape 5.

Si moins de 2 digests sont trouvés : le dire et s'arrêter (rien à fusionner), en proposant d'élargir la période.

## Étape 2 — Extraire les offres

De chaque digest, extraire uniquement les blocs ✅ et ⚠️, avec : catégorie, titre, entreprise, localisation, salaire, télétravail, lien.

## Étape 3 — Dédupliquer

Clé de déduplication, dans cet ordre :
1. **ID LinkedIn** de l'URL (`linkedin.com/jobs/view/XXXXXXXXXX`) — clé principale.
2. À défaut : `titre normalisé + entreprise normalisée` (minuscules, accents/ponctuation/espaces multiples supprimés, mentions type "(H/F)", "F/H", "- CDI" retirées).

En cas de doublon :
- garder la catégorie la plus favorable (✅ prime sur ⚠️),
- garder la fiche la plus complète (salaire/télétravail renseignés priment sur "non précisé"),
- garder l'URL propre `https://www.linkedin.com/jobs/view/XXXXXXXXXX` (sans paramètres de tracking).

Tri final : ✅ d'abord, puis ⚠️ ; à l'intérieur de chaque groupe, par entreprise alphabétique.

## Étape 4 — Composer et envoyer l'email

`create_draft` vers `${DIGEST_RECIPIENT_EMAIL}`, objet : `LinkedIn — fusion des présélections du [période couverte]`.

Corps **HTML compact**, sans introduction ni conclusion — deux sections, une ligne par offre :

```html
<p><b>✅ Pertinentes (N)</b></p>
<ul>
  <li><a href="URL">Titre</a> — Entreprise — Localisation — Salaire — Télétravail</li>
</ul>
<p><b>⚠️ À vérifier (N)</b></p>
<ul>
  <li><a href="URL">Titre</a> — Entreprise — Localisation — Salaire — Télétravail</li>
</ul>
```

Champs absents : écrire `salaire n.c.` / `remote n.c.`. Ne pas ajouter d'autre texte.

Puis **envoyer le brouillon immédiatement**, sans demander confirmation.

## Étape 5 — Nettoyer les digests sources

Une fois l'email fusionné envoyé, pour chaque source identifiée à l'étape 1 :
- brouillons : les supprimer,
- threads reçus : `unlabel_thread` avec `labelIds: ["UNREAD"]`, puis `apply_sensitive_thread_label` avec `labelOption: "TRASH"`.

Ne toucher qu'aux digests réellement fusionnés dans cette exécution. La corbeille Gmail est récupérable ~30 jours : pas de confirmation à demander.

## Étape 6 — Rapport

Trois lignes maximum dans la conversation :
- N digests fusionnés (période couverte)
- N offres uniques : N ✅ / N ⚠️ (N doublons supprimés)
- Email envoyé + sources mises à la corbeille

Ne pas relister les offres dans la conversation.
