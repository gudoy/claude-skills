---
name: linkedin-job-digest
description: "Filtre les alertes d'offres d'emploi LinkedIn reçues par email et produit une présélection personnalisée. Déclencher ce skill dès que l'utilisateur mentionne : parcourir ses alertes LinkedIn, filtrer des offres d'emploi, faire une sélection d'offres, \"regarde mes alertes LinkedIn\", \"quelles offres j'ai reçues\", \"trie mes offres\", ou toute demande liée à la recherche d'emploi et aux emails de job alerts. Utiliser même si l'utilisateur dit simplement \"montre-moi les offres du jour\" ou \"qu'est-ce que j'ai reçu comme offres\"."
---

# Filtre d'alertes LinkedIn

Ce skill parcourt les emails d'alertes LinkedIn dans Gmail, extrait les offres d'emploi, et produit une présélection basée sur le profil et les critères définis ci-dessous.

## Principes transversaux

Ces règles s'appliquent à toutes les étapes ci-dessous, pas seulement à celle où elles sont mentionnées.

**Chaque offre doit être visitée avant d'être classée.** Les infos de l'email seul (titre, extrait, parfois un salaire approximatif) ne suffisent jamais à décider — voir étape 3. Ce n'est vrai que si la page reste inaccessible (login LinkedIn requis, page supprimée), auquel cas c'est signalé explicitement comme une limite plutôt que traité comme le cas normal.

**Toujours un lien vers l'offre.** Une offre présentée (✅ ou ⚠️) sans lien cliquable ne sert à rien — L'utilisateur ne peut ni la consulter ni postuler dessus. Le lien doit être présent dans le rapport texte (étape 5) ET dans le digest email (étape 7), pour chaque offre sans exception. Si le lien "propre" (sans tracking) n'a pas pu être reconstruit ou que la page n'a pas pu être visitée, utiliser le lien brut récupéré dans l'email plutôt que d'omettre le lien.

**Si une règle ne peut pas être respectée, ne pas improviser en silence.** Si un outil nécessaire est indisponible, une permission manque, ou une instruction de ce skill entre en conflit avec ce qui est observé (ex : impossible de récupérer les threads, impossible de visiter une page d'offre, impossible de créer le brouillon, impossible de nettoyer un email), arrêter l'étape concernée et expliquer clairement le blocage à l'utilisateur à la fin du rapport, en proposant 2-3 alternatives concrètes plutôt qu'un simple constat d'échec.

**Proposer l'évolution du skill si pertinent.** Si un pattern récurrent apparaît pendant l'exécution (ex : un nouveau critère qui revient souvent, une entreprise à toujours exclure/inclure, une étape manuelle qui pourrait être fiabilisée), le signaler à l'utilisateur en fin de rapport comme suggestion d'amélioration du skill — sans modifier le skill soi-même.

**Signaler les optimisations possibles.** Si l'exécution a été longue ou coûteuse en tokens (ex : beaucoup de pages visitées aux étapes 3/6, volume important d'offres), et qu'une piste d'optimisation est identifiable (ex : paralléliser les visites de pages, réduire le nombre de pages similaires explorées à l'étape 6), le proposer à l'utilisateur comme évolution possible du skill plutôt que de le refaire silencieusement à chaque exécution. Attention toutefois : la visite systématique de chaque offre (étape 3) est une exigence fonctionnelle et ne doit jamais être sacrifiée pour gagner du temps ou des tokens — l'optimisation porte sur le "comment", jamais sur le fait de sauter cette étape.

**Toujours refermer les onglets de navigateur ouverts pour ce skill.** Les étapes 3 et 6 utilisent l'outil Chrome pour visiter des pages LinkedIn. Dans la mesure du possible, réutiliser le même onglet d'une offre à l'autre plutôt que d'en ouvrir un nouveau à chaque fois. Dans tous les cas, avant de terminer l'exécution (juste avant ou juste après l'étape 8), fermer (`tabs_close_mcp`) tous les onglets/fenêtres ouverts spécifiquement pour cette exécution — L'utilisateur ne doit pas se retrouver avec des dizaines d'onglets LinkedIn ouverts après un run.

## Profil du candidat

**Full-Stack Software Engineer**, ~20 ans d'expérience  
Base géographique : `${CANDIDATE_LOCATION}` (défaut : Île-de-France)

**Stack principale (maîtrise forte) :**
TypeScript, JavaScript, Node.js, React, NestJS, Next.js, PostgreSQL, MongoDB, GraphQL, REST, Docker, GCP, AWS, Redis, Tailwind, Jest

**Stack secondaire (connaît mais préfèrerait éviter comme stack principale) :**
Python, PHP, MySQL

**Langues :** Français (natif) — Anglais B2 (fonctionnel mais pas à l'aise pour une communication quotidienne majoritairement en anglais)

---

## Critères de filtrage

### ✅ Critères positifs (inclure si présents)
- Poste : Software Engineer / Développeur fullstack, front ou back (tous niveaux : Senior, Staff, Lead, Principal, etc.)
- Stack dominante : TypeScript ou JavaScript (Node.js, React, NestJS, Next.js, Vue, Angular, etc.)
- Salaire annuel brut ≥ 80 000 €
- Télétravail total (remote full)
- Télétravail partiel avec ≤ 2 jours/semaine au bureau OU ≤ 3 jours/semaine grand max si bureaux proches de la gare Saint-Lazare (Paris 8e/17e/9e)
- Entreprise française ou communication de travail majoritairement en français
- Startup / scaleup / SaaS B2B (contexte familier et valorisable)

### ❌ Critères d'exclusion (rejeter si présents explicitement)
- Stack principale Java, .NET, C#, Ruby, Swift, Kotlin, Python-only, PHP-only
- Applications mobiles natives (iOS/Android) uniquement
- ESN / SSII / cabinet de conseil (type Capgemini, Sopra, Accenture, SQLI, etc.)
- Exige un "excellent niveau d'anglais" ou "English-speaking environment" ou "fluent English required"
- Salaire maximum affiché < 80 000 €
- Localisation uniquement hors Île-de-France (si pas full remote)
- Plus de 2 jours/semaine au bureau si bureaux non proches de Saint-Lazare, ou plus de 3 jours/semaine même si proches de Saint-Lazare

### ⚠️ Critères d'incertitude (signaler, ne pas rejeter automatiquement)
- Stack principale Go, Rust si expérience dessus non requise
- Salaire non mentionné dans l'offre → signaler comme info manquante
- Politique de télétravail non précisée → signaler comme info manquante
- Localisation imprécise → signaler comme info manquante
- Stack TypeScript/JS présente mais pas dominante → mentionner
- Entreprise internationale mais offre potentiellement en français → mentionner

---

## Processus d'exécution

### Étape 1 : Récupérer les emails d'alertes LinkedIn

Utiliser l'outil Gmail (`search_threads`) pour trouver les alertes LinkedIn récentes.

Requête suggérée : `from:jobalerts-noreply@linkedin.com` avec un filtre sur les 3 derniers jours (ou la période demandée par l'utilisateur).

Si l'utilisateur ne précise pas de période, prendre les 3 derniers jours.

Récupérer les threads correspondants avec `get_thread` pour accéder au contenu complet des emails.

Garder la liste des `threadId` récupérés ici de côté : c'est elle qui servira à l'étape 8 pour savoir quels emails ont été effectivement traités dans cette exécution.

### Étape 2 : Extraire les offres

Pour chaque email d'alerte LinkedIn, extraire chaque offre mentionnée :
- Titre du poste
- Nom de l'entreprise
- Localisation indiquée
- Salaire (si mentionné)
- Description courte / extrait
- Lien vers l'offre
- Date de réception de l'email

Les emails LinkedIn contiennent souvent plusieurs offres. Traiter chaque offre séparément.

### Étape 3 : Visiter systématiquement chaque offre pour en extraire les données de filtrage

Les emails LinkedIn ne contiennent presque jamais le salaire, la stack technique précise, ou la politique de télétravail réelle — et ce qu'ils contiennent peut être trompeur (titre accrocheur, description tronquée). Ces informations ne sont fiables que sur la page de l'offre elle-même. **Se baser uniquement sur le contenu de l'email pour filtrer n'est donc jamais suffisant : chaque offre extraite à l'étape 2 doit être visitée avant d'être classée**, sans exception et sans condition déclenchante — ce n'est pas une étape optionnelle d'enrichissement mais un passage obligé avant toute décision.

**Comment faire :**

Pour chaque offre extraite à l'étape 2, extraire l'ID LinkedIn depuis l'URL de l'offre (format : `linkedin.com/jobs/view/XXXXXXXXXX`) et construire une URL propre : `https://www.linkedin.com/jobs/view/XXXXXXXXXX`

Utiliser l'outil Chrome (`mcp__claude-in-chrome__get_page_text` ou `navigate` + `read_page`) pour charger la page de l'offre et en extraire :

- **Salaire** : chercher les mentions "€", "k€", "salaire", "rémunération", "package"
- **Stack technique** : chercher les sections "compétences", "stack", "technologies", "vous maîtrisez", "vous avez de l'expérience avec"
- **Télétravail** : chercher "télétravail", "remote", "hybride", "présentiel", "jours au bureau"
- **Localisation précise** : adresse ou arrondissement si Paris
- **Langue de travail** : chercher "anglais", "français", "bilingue", "working language"
- **Type d'entreprise** : ESN, startup, scale-up, grand groupe — souvent dans la description

Ces données remplacent/complètent celles de l'email et serviront de base à l'évaluation (étape 4) — ne pas évaluer une offre avant d'avoir visité sa page.

**Si une page n'est pas accessible** (LinkedIn demande une connexion, page supprimée, erreur de chargement) : c'est le seul cas où l'évaluation se fait avec les seules infos de l'email. Classer alors l'offre au mieux avec ce qui est disponible, et signaler explicitement dans le rapport que le filtrage repose sur des infos incomplètes faute d'accès à la page — conformément au principe transversal "ne pas improviser en silence".

### Étape 4 : Évaluer chaque offre

Pour chaque offre, une fois sa page visitée (étape 3), appliquer les critères ci-dessus et assigner une décision :

- **✅ À voir** : correspond aux critères positifs, aucun critère d'exclusion formel
- **⚠️ À vérifier** : peut correspondre mais des infos clés manquent (salaire, remote, localisation) — y compris quand la page n'a pas pu être visitée
- **❌ Hors critères** : au moins un critère d'exclusion est présent

Ne pas inclure les offres ❌ dans la présentation finale sauf si l'utilisateur le demande.

Exemples de raisonnement (après visite de la page) :

> Page visitée : "Senior TypeScript Engineer, full remote, Paris" mais salaire non précisé même sur la page → ⚠️ À vérifier (salaire manquant)

> Page visitée : "Développeur Node.js / React, SaaS B2B, Paris 8e, 3j télétravail/semaine, 85-100k€" → ✅ À voir

> Page visitée : "Java Backend Engineer, full remote, 90k€" → ❌ Stack Java

> Page visitée : "Lead Developer JS - ESN Sopra" → ❌ ESN

> Page visitée : "Senior Software Engineer, must be fluent in English, Paris" → ❌ Anglais requis

### Étape 5 : Présenter les résultats

Présenter les offres **✅ À voir** en premier, puis les **⚠️ À vérifier**, en ignorant les ❌.

Pour chaque offre retenue, afficher :

```
## [Décision] Titre du poste — Entreprise

**Localisation :** ...
**Salaire :** ... (ou "non précisé ⚠️")
**Télétravail :** ... (ou "non précisé ⚠️")
**Stack mentionnée :** ...
**Pourquoi ça matche :** [2-3 points clés]
**Points d'attention :** [si applicable]
**Lien :** [URL de l'offre]
```

Le champ **Lien** est obligatoire pour chaque offre affichée, sans exception — jamais laissé vide, jamais remplacé par une simple mention du nom de l'entreprise.

Terminer par un résumé :
- Nombre total d'offres analysées
- Nombre ✅ À voir / ⚠️ À vérifier / ❌ Hors critères
- Éventuellement : un commentaire sur les tendances observées (ex : "beaucoup d'offres Java cette semaine, peu de full remote")

### Gestion des cas limites

**Si une offre nécessite une précision avant de décider :** Poser la question à l'utilisateur directement dans le rapport, sous forme de `❓ Question :`.

Exemple :
> ❓ Question : Cette offre chez **Acme Corp** ne précise ni le salaire ni la politique de télétravail, même sur la page de l'offre. Le titre et la stack correspondent bien. Veux-tu que je la garde en ⚠️ ou tu veux l'ignorer ?

**Si aucune alerte LinkedIn n'est trouvée dans la période :** Indiquer clairement et suggérer d'élargir la période de recherche.

**Si une offre n'a pas pu être visitée (étape 3) :** Noter l'offre avec les infos disponibles dans l'email et indiquer clairement que le filtrage est incomplet faute d'accès à la page.

---

### Étape 6 : Exploration des offres complémentaires

Les pages d'offres LinkedIn pointent souvent vers d'autres offres du même type via des boutons comme « Afficher toutes les offres similaires » ou « Voir toutes les offres d'emploi ». Ces offres ne sont pas dans l'email d'alerte mais peuvent tout à fait correspondre au profil de l'utilisateur — ça vaut le coup de les explorer pour élargir la présélection au-delà du contenu brut des emails.

**Déclencher cette étape si :**
- Une page d'offre visitée à l'étape 3 contient un bouton menant vers d'autres offres
- L'utilisateur demande explicitement d'explorer plus d'offres ("regarde aussi les offres similaires", "explore plus loin", etc.)

**Comment faire :**

1. Sur une page d'offre déjà ouverte (étape 3), repérer un bouton/lien du type « Afficher toutes les offres similaires » ou « Voir toutes les offres d'emploi ».
2. Cliquer dessus (ou naviguer vers l'URL correspondante — en réutilisant l'onglet déjà ouvert, voir principe transversal sur les onglets) et lister les offres qui apparaissent, avec les mêmes champs qu'à l'étape 2 (titre, entreprise, localisation, lien).
3. Pour chaque offre complémentaire qui n'était pas déjà dans un email d'alerte, l'ajouter au pipeline normal : visite systématique de sa page (étape 3) puis évaluation (étape 4) — les mêmes règles s'appliquent, sans raccourci.
4. Pour éviter une exploration sans fin, se limiter à environ 5 offres complémentaires par page explorée, et ne pas rebondir une seconde fois sur les « offres similaires » d'une offre déjà elle-même complémentaire (profondeur 1 seulement).
5. Dans la présentation finale (étape 5), marquer ces offres comme « offre complémentaire (via offres similaires) » pour que l'utilisateur sache qu'elles ne viennent pas directement d'une alerte email.

### Étape 7 : Composer le digest email

En complément du rapport texte affiché dans la conversation (étape 5), préparer un email récapitulatif — pratique pour retrouver la présélection plus tard, l'archiver ou la transférer.

**Comment faire :**

1. Utiliser l'outil Gmail (`create_draft`) pour créer un brouillon adressé à `${DIGEST_RECIPIENT_EMAIL}`.
2. Objet : `LinkedIn — présélection du [date du jour]` (adapter si la période couverte est différente d'un jour).
3. Corps : reprendre la même structure que l'étape 5 (offres ✅ puis ⚠️, en ignorant les ❌), avec pour chaque offre le lien cliquable — voir le principe transversal "Toujours un lien vers l'offre".
4. Ne jamais envoyer l'email automatiquement : le brouillon reste à relire et envoyer par l'utilisateur lui-même. Indiquer dans le rapport final qu'un brouillon a été créé.

**Si `create_draft` échoue ou n'est pas disponible :** ne pas essayer de recomposer l'email via le navigateur en silence — signaler le blocage conformément au principe transversal "ne pas improviser en silence" et proposer à l'utilisateur de créer le brouillon manuellement ou de réessayer.

### Étape 8 : Marquer comme lus et mettre à la corbeille les emails traités

Une fois le rapport (étape 5) et le brouillon de digest (étape 7) produits, nettoyer la boîte de réception : les alertes LinkedIn traitées dans cette exécution n'ont plus besoin d'y rester.

**Comment faire, pour chaque `threadId` collecté à l'étape 1 :**

1. Marquer le thread comme lu : `unlabel_thread` avec `labelIds: ["UNREAD"]`.
2. Mettre le thread à la corbeille : `apply_sensitive_thread_label` avec `labelOption: "TRASH"`.

Faire les deux actions dans cet ordre (lu, puis corbeille) pour chaque thread traité. La corbeille Gmail n'est pas une suppression définitive — les emails y restent récupérables environ 30 jours — donc cette étape peut s'exécuter automatiquement à chaque run, sans redemander confirmation à l'utilisateur.

**Ne traiter que les threads réellement récupérés à l'étape 1** de cette exécution — ne jamais élargir le nettoyage à d'autres emails de la boîte, même s'ils ressemblent à des alertes LinkedIn.

**Si un thread contient une offre avec une `❓ Question` non résolue (cas limite de l'étape 5) :** laisser ce thread-là de côté (ni marqué lu, ni mis à la corbeille) jusqu'à ce que l'utilisateur ait répondu, pour ne pas perdre l'accès à l'email source pendant que la décision est en suspens.

**Si les outils Gmail (`unlabel_thread` / `apply_sensitive_thread_label`) sont indisponibles ou échouent :** basculer sur le navigateur Chrome pour faire la même chose manuellement dans l'interface Gmail — ouvrir le thread, le marquer comme lu, puis utiliser l'action de suppression (icône corbeille) de Gmail. Penser à refermer l'onglet Gmail ouvert pour l'occasion une fois l'action faite (voir principe transversal sur les onglets). Si même le navigateur échoue, ne pas laisser l'étape de côté silencieusement : le signaler dans le rapport final conformément au principe transversal "ne pas improviser en silence".
