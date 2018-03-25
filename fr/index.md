# Git Rebase, pourquoi, quand, comment ?

## Différences avec un merge

Prennons une divergeance d'historique (ex: travail de deux personnes concurrentes sur une même branche).

```
            G---H (master)
           /
  A---B---C---D---E---F (origin/master)
```

si l'on effectue un merge (via `git merge origin/master` ou `git pull`) on obtient

```
            G---H-------I (master)
           /           /
  A---B---C---D---E---F (origin/master)
```
**`I`** étant le commit de merge.
 
 après un push on obtiendra
 
```
             ---G---H---
           /             \
  A---B---C---D---E---F---I (origin/master, master)
```

Avantages:
- permet de bien visualiser la parralélisation des développement et donc de bien identifier les risques de conflits d'intégration (incompatibilité entre deux features).
- simplicité du workflow: on n'utilise que 3 commandes : `commit`, `pull`, `push`.
- permet de faciliter l'éviction d'un feature problématique pour la production d'une démo ou d'une build de production si tous les commits liés à cette feature sont bien isolés dans une branche dédiée avant d'être mergée.

Difficultés:
- rend l'utilisation de la consultation d'historique ou de l'outil `git blame` moins clair.
- sur une branche de feature ayant duré longtemps, le commit de merge, en cas de conflit, peut contenir un patch volumineux et les changements se trouvent alors mal compartimenté (on identifie plus difficilement la raison pour laquelle une modification a été faite).
- l'outil de recherche d'origine de régression `git bissect` devient en pratique quasi inutilisable.
- avec la croissance d'une équipe, la proportion de commits de merge par rapport aux commits de développement augmente polluant l'historique.

