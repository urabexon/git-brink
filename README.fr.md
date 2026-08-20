# git-brink

[English](./README.md) · [日本語](./README.ja.md) · **Français**

Montre ce qu'un `git add -A` malencontreux exposerait depuis votre arbre de travail — avant que cela n'arrive.

```
$ git brink ~

git-brink  /Users/you
origin: git@github.com:you/some-public-repo.git

CRITICAL  repository root is your home directory
          everything under $HOME is inside this repository's working tree
CRITICAL  58 sensitive path(s) would be staged by 'git add -A'
          .aws/                              AWS credentials
          .ssh/                              SSH keys
          .config/gh/hosts.yml               GitHub CLI token
          .s3cfg                             S3 credentials
          .zsh_history                       shell/tool history
          projects/deploy-key.pem            key material
          ... and 38 more
WARN      159 nested git repositories inside the working tree

2 critical, 1 warning(s).
```

## Pourquoi

Les scanners de secrets examinent ce que vous avez *déjà* commité ou indexé.
git-brink examine ce qui est sur le point de *devenir* committable.

Le scénario pour lequel il a été écrit est d'une simplicité embarrassante :
un `git init` ou un `git clone` lancé un répertoire trop haut. Soudain,
`$HOME` est un dépôt, `.ssh/` et `.aws/` se trouvent dans son arbre de
travail, `origin` pointe vers un dépôt public — et rien n'est encore cassé.
Rien n'est indexé. Rien n'est commité. Un scanner ne trouve rien, parce
qu'il n'y a rien à trouver. Vous êtes à un `git add -A` d'un très mauvais
après-midi.

git-brink répond à la seule question qui compte à cet instant :
**si je lançais `git add -A && git commit && git push` maintenant,
qu'est-ce qui partirait ?**

## Installation

```sh
curl -o /usr/local/bin/git-brink \
  https://raw.githubusercontent.com/urabexon/git-brink/main/git-brink
chmod +x /usr/local/bin/git-brink
```

Tout exécutable nommé `git-*` présent dans votre `PATH` devient une
sous-commande git : vous pouvez donc ensuite l'invoquer avec `git brink`.

Nécessite bash 3.2 ou plus récent (le bash fourni avec macOS suffit) et git.
`gh` est utilisé de manière opportuniste pour déterminer si le dépôt distant
est public, mais n'est jamais requis.

## Utilisation

```sh
git brink                 # vérifie le dépôt contenant le répertoire courant
git brink ~/projects      # vérifie le dépôt contenant le chemin indiqué
git brink --no-network    # ignore la vérification de visibilité du dépôt distant
```

### Codes de sortie

| Code | Signification |
|------|---------------|
| `0`  | rien trouvé |
| `1`  | avertissements uniquement |
| `2`  | au moins un problème critique |

Utile dans un hook de pré-commit :

```sh
#!/bin/sh
git brink --no-network || exit 1
```

## Ce qu'il vérifie

| Gravité | Vérification |
|----------|-------------|
| CRITICAL | La racine du dépôt est `$HOME`, un ancêtre de `$HOME`, ou `/` |
| CRITICAL | Des chemins non suivis et non ignorés qui ressemblent à des clés, des identifiants, des jetons, des fichiers d'environnement ou des historiques de shell |
| CRITICAL | Les cas ci-dessus, lorsque le dépôt distant est public |
| WARN | Aucun `.gitignore` alors que des fichiers non suivis existent |
| WARN | Des dépôts git imbriqués dans l'arbre de travail |
| WARN | Budget d'analyse épuisé : le résultat n'est donc pas concluant |

Les fichiers d'exemple et de modèle (`.env.example`, `*.sample`,
`*.template`) ainsi que les parties publiques des paires de clés (`*.pub`)
ne sont délibérément pas signalés.

## Pourquoi c'est rapide

L'approche naïve — énumérer chaque fichier non suivi — prend plusieurs
minutes sur un dépôt dont la racine est `$HOME`, car git parcourt
`Library/`, tous les caches et tous les `node_modules` du disque. git-brink
demande à git de replier les répertoires entièrement non suivis, puis y
descend lui-même, de façon superficielle et sous un budget fixe. Le cas du
répertoire personnel ci-dessus s'exécute en trois secondes environ.

Si le budget est épuisé, git-brink le signale plutôt que d'annoncer un
résultat propre dont il ne peut pas répondre.

## Ce qu'il n'est pas

Il n'analyse pas le contenu des fichiers et ne consulte pas l'historique. Si
vous avez besoin d'une recherche par motifs sur ce qui est déjà commité,
utilisez [gitleaks](https://github.com/gitleaks/gitleaks) ou
[git-secrets](https://github.com/awslabs/git-secrets) : ils résolvent
l'autre moitié du problème, et les deux se complètent bien.

## Licence

[MIT](./LICENSE)
