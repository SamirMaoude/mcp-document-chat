# MCP Chat

Chat en ligne de commande avec Claude, connecté à un serveur MCP ([Model Context Protocol](https://modelcontextprotocol.io)) qui expose une petite collection de documents. Claude peut lire et modifier ces documents grâce aux outils MCP, et l'utilisateur peut les citer avec `@` ou lancer des commandes prédéfinies avec `/`.

## Contexte

Projet réalisé dans le cadre du cours **Introduction to Model Context Protocol** d'Anthropic. Le squelette (interface CLI, boucle de chat) est fourni par le cours. J'ai implémenté :

- le **serveur MCP** (`mcp_server.py`) : tools, resources et prompts ;
- le **client MCP** (`mcp_client.py`) : liste et appel des tools, récupération des prompts, lecture des resources.

## Fonctionnalités

- **Mention de documents** : `@report.pdf` injecte le contenu du document dans la question (via une resource MCP).
- **Commandes** : `/summarize` et `/format` lancent des prompts définis côté serveur.
- **Utilisation d'outils** : Claude appelle lui-même `read_doc_contents` et `edit_document` quand il en a besoin.
- **Autocomplétion** des commandes et des documents avec Tab.
- **Plusieurs serveurs MCP** : d'autres serveurs peuvent être passés en argument au lancement.

## Architecture

```
main.py            Point d'entrée : démarre le serveur MCP de documents et la CLI
mcp_server.py      Serveur MCP (FastMCP) : documents en mémoire, tools, resources, prompts
mcp_client.py      Client MCP (transport stdio) autour de ClientSession
core/
├── claude.py      Appels à l'API Anthropic
├── chat.py        Boucle agent : appelle Claude et exécute les tool calls jusqu'à la réponse finale
├── cli_chat.py    Traitement des @mentions et des /commandes
├── cli.py         Interface prompt_toolkit (saisie, autocomplétion)
└── tools.py       Regroupe les tools de tous les serveurs et route chaque appel vers le bon client
```

Primitives exposées par le serveur :

| Type     | Nom                         | Rôle                                         |
|----------|-----------------------------|----------------------------------------------|
| Tool     | `read_doc_contents`         | Lire le contenu d'un document                |
| Tool     | `edit_document`             | Remplacer une chaîne dans un document        |
| Resource | `docs://documents`          | Liste des identifiants de documents (JSON)   |
| Resource | `docs://documents/{doc_id}` | Contenu d'un document                        |
| Prompt   | `format`                    | Réécrire un document en Markdown             |
| Prompt   | `summarize`                 | Résumer un document                          |

## Prérequis

- Python 3.10+
- [uv](https://github.com/astral-sh/uv) (recommandé)
- Une clé API Anthropic
- Node.js, uniquement pour MCP Inspector (voir [Tester](#tester))

## Installation

```bash
git clone https://github.com/SamirMaoude/mcp-document-chat.git
cd mcp-document-chat

cp .env.example .env    # puis renseigner ANTHROPIC_API_KEY
uv sync                 # crée .venv et installe les dépendances
```

Sans uv :

```bash
python -m venv .venv
source .venv/bin/activate    # Windows : .venv\Scripts\activate
pip install "anthropic<1" python-dotenv prompt-toolkit "mcp[cli]>=1.8.0,<2"
```

puis mettre `USE_UV=0` dans `.env`.

### Variables d'environnement

| Variable            | Description                                                     |
|---------------------|-----------------------------------------------------------------|
| `ANTHROPIC_API_KEY` | Clé API Anthropic                                               |
| `CLAUDE_MODEL`      | Modèle utilisé, par exemple `claude-haiku-4-5`                  |
| `USE_UV`            | `1` pour lancer le serveur MCP avec `uv`, `0` avec `python`     |

## Lancer l'application

```bash
uv run main.py
```

Sans uv (environnement virtuel activé) :

```bash
python main.py
```

Pour connecter des serveurs MCP supplémentaires (lancés avec `uv run`), passer leurs scripts en argument :

```bash
uv run main.py autre_serveur.py
```

## Utilisation

```
> Que contient @report.pdf ?
> /summarize plan.md
> /format deposition.md
```

- Taper un message puis Entrée pour discuter avec Claude.
- `@` suivi d'un identifiant de document pour l'ajouter au contexte.
- `/` pour lister les commandes ; Tab pour compléter.
- Ctrl+C pour quitter.

Documents disponibles : `deposition.md`, `report.pdf`, `financials.docx`, `outlook.pdf`, `plan.md`, `spec.txt`. Ils sont stockés en mémoire dans `mcp_server.py` : les modifications faites par `edit_document` sont perdues au redémarrage.

## Tester

### Test rapide du serveur (sans clé API)

```bash
uv run mcp_client.py
```

Démarre le serveur, s'y connecte et affiche les tools disponibles :

```
Available tools:
- read_doc_contents: Read the contents of a document and return it as a string.
- edit_document: Edit a document by replacing a string in the documents content with a new string.
```

### MCP Inspector

```bash
uv run mcp dev mcp_server.py
```

Ouvre l'interface web de MCP Inspector (l'URL est affichée dans le terminal). On peut y appeler chaque tool, lire les resources et générer les prompts à la main, sans passer par Claude.

## Ajouter des documents

Ajouter une entrée au dictionnaire `docs` dans `mcp_server.py` :

```python
docs = {
    ...
    "notes.txt": "Contenu du nouveau document.",
}
```

## Technologies

Python, [SDK MCP](https://github.com/modelcontextprotocol/python-sdk) (FastMCP), [SDK Anthropic](https://github.com/anthropics/anthropic-sdk-python), prompt_toolkit, uv.
