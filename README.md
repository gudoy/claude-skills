# claude-skills

Skills [Claude](https://claude.com) personnels, versionnés.

Chaque skill est un dossier sous `skills/` contenant un `SKILL.md` : un en-tête YAML
(`name`, `description` — c'est la `description` qui décide du déclenchement) suivi des
instructions en Markdown.

## Skills

| Skill | Rôle |
|---|---|
| `add-to-candidatures` | Ajoute une candidature à la base Notion « Candidatures » |
| `add-to-notion-articles` | Archive un article dans la base Notion « Articles » |
| `linkedin-job-digest` | Filtre les alertes emploi LinkedIn et produit une présélection |
| `linkedin-job-digests-merge` | Fusionne plusieurs présélections LinkedIn en un digest dédupliqué |
| `ebay-daily-digest` | Compile les alertes eBay du jour en un digest HTML envoyé par email |

## Configuration

Les skills ne contiennent **aucune valeur personnelle ni secret en dur**. Tout ce qui est
spécifique à un utilisateur ou à un espace de travail passe par des variables :

```bash
cp .env.example .env   # puis renseigner les valeurs
```

Voir `.env.example` pour la liste complète et le rôle de chaque variable.

## Règles de contribution

- **Un skill = une pull request.** Jamais deux skills dans le même commit.
- **Aucun secret, aucune donnée personnelle** dans un fichier versionné : ni token, ni
  adresse email, ni adresse postale, ni ID de base privée. Utiliser une variable et la
  documenter dans `.env.example`.
- Relire le diff avant chaque commit — les skills sont du texte libre, un copier-coller
  depuis une session de travail réintroduit vite une valeur réelle.

## Installation

Copier le dossier d'un skill dans le répertoire de skills de Claude :

```bash
cp -r skills/<nom-du-skill> ~/.claude/skills/
```

Ou, dans l'application Claude, importer le `SKILL.md` via la gestion des skills.
