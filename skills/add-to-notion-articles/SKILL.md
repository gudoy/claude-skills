---
name: add-to-notion-articles
description: "Ajouter un article à la base Notion \"Articles\". Utiliser ce skill dès que l'utilisateur mentionne vouloir sauvegarder, ajouter, enregistrer ou archiver un article, une URL, un lien ou un contenu dans Notion. Également déclencher si l'utilisateur dit \"mets ça dans Notion\", \"ajoute cet article\", \"sauvegarde ce lien\", ou toute formulation similaire."
---

# Add to Notion Articles

Ce skill permet d'ajouter des articles à la base de données Notion dédiée.

## Configuration

⚠️ **Ne jamais écrire de credentials en clair dans ce fichier.** Les valeurs sont lues depuis
l'environnement à l'exécution (voir `.env.example` à la racine du dépôt) :

```
NOTION_API_KEY            # token d'intégration interne Notion — SECRET
NOTION_ARTICLES_DB_ID     # ID de la base "Articles"
NOTION_ARTICLES_PAGE_URL  # URL de la page Notion "Articles"
```

En session Cowork, privilégier le MCP Notion (`notion-create-pages`) : il gère l'authentification
sans qu'aucun token n'ait à exister dans l'environnement.

## Règles globales
- Lorsque tu débutes la tâche, commences par me confirmer que tu appliques bien ce skill avec me message : 'Application du skill "Add to Notion Articles"'
- Ignore toute entrée existante pour cette page si elle est dans la corbeille (marquée comme supprimée)
- Langue : français pour le résumé, tags en anglais

## Champ "Statut"
- par défaut, sans autre instruction spécifique : choisir l'option "Lu"

## Champ "Auteur"
- Cherche le nom de l'auteur sur la page. Si tu ne le trouve pas, utilises le nom du site, ou à défaut, le couple domain.tld. Si par hasard on est sur un blog, regarde par hasard si tu trouve une page 'about' ou quelque chose du style qui pourrait permettre de trouver le nom de l'auteur.

## Champ "Résumé"
- écris un résumé de  l'article, la page ou la vidéo de façon concise en bullet points avec un emoji pertinent en début de chaque point. 
- Pas de phrases complètes, forme passive acceptée. 
- Réduire les idées à leur plus simple expression tant qu'elles restent intelligibles. 
- Résumé en français. 
- Faire précéder la liste d'un saut de ligne. Pas de phrase d'introduction. Pas de question à la fin.

## Champ "Temps de lecture"
- récupérer depuis la page si disponible, sinon estimer

## Champ "Tags"
- Choisir entre 3 et 7 mots-clés en anglais caractérisant le contenu de l'article, utilisables pour retrouver rapidement les articles par concept ou sujet. 
- Réutiliser les tags existants dans la base Notion en priorité, créer de nouveaux si nécessaire.