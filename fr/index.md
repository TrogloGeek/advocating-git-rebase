# Un peu de terrorisme git

## Petit retour rapide sur `git pull` en mode `--merge` (comportement par défaut)

Prennons une divergeance d'historique (ex: travail de deux personnes concurrentes sur une même branche).

```
            G---H (master)
           /
  A---B---C---D---E---F (origin/master)
```

si l'on effectue un merge (via `git merge origin/master` ou `git pull [origin master]`) on obtient

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

**Avantages:**
- permet de bien visualiser la parralélisation des développement et donc de bien identifier les risques de conflits d'intégration (incompatibilité entre deux features).
- simplicité du workflow: on n'utilise que 3 commandes : `commit`, `pull`, `push`.
- permet de faciliter l'éviction d'un feature problématique pour la production d'une démo ou d'une build de production si tous les commits liés à cette feature sont bien isolés dans une branche dédiée avant d'être mergée.

**Difficultés:**
- rend l'utilisation de la consultation d'historique ou de l'outil `git blame` moins clair.
- sur une branche de feature ayant duré longtemps, le commit de merge, en cas de conflit, peut contenir un patch volumineux et les changements se trouvent alors mal compartimenté (on identifie plus difficilement la raison pour laquelle une modification a été faite).
- l'outil de recherche d'origine de régression `git bissect` devient en pratique quasi inutilisable.
- avec la croissance d'une équipe, la proportion de commits de merge par rapport aux commits de développement augmente polluant l'historique.

## vers un workflow plus poussé

Petit détail génant dans l'historique final précédent, notre commit **`I`** est nommé par défaut **merged origin/master into master** ce qui faisait sens dans la structure d'arborescence d'historique d'avant push, mais beaucoup moins après. Cette incongruité de nommage de commit peut facilement être résolu en ne travaillant jamais en local sur la branche d'intégration mais sur une branche de feature (ex: **`feat/my-nice-feature`**).

On obtiendrait alors avant le pull:
```
            G---H (feat/my-nice-feature)
           /
  A---B---C---D---E---F (origin/master)
```

avant de pousser on commencera alors par resynchroniser la branche de feature avec l'état de la branche d'intégration avec `git pull origin master` avant de faire nos tests finaux:
```
            G---H-------I (feat/my-nice-feature)
           /           /
  A---B---C---D---E---F (origin/master)
```

puis si nos tests sont réussi on pourra reverser la branche de feature dans la branche d'intégration via une pull-request ou `git checkout master; git pull; git merge feat/my-nice-feature; git push`.
```
            G---H-------I (feat/my-nice-feature)
           /           / \
  A---B---C---D---E---F---J (origin/master)
```

Si notre historique a été complexifié avec un commit *inutile* de plus, on obtient un historique avec une sémentique plus claire puisque l'on aura nos commits **`I`** et **`J`** respectivement nommés:
**`I`**: *merged origin/master into feat/my-nice-feature* on comprend bien qu'il s'agit d'une resynchronisation d'une branche de feature avec l'état de l'intégration.

**`J`**: *merged feat/my-nice-feature into master* on comprend bien qu'une feature vient d'être ajoutée à l'intégration.

Ce type de workflow merge se prête, lorsque le versement d'une feature à l'intégration est effectuée au moyen d'un pull-request à:
- l'audit de code par le responsable qualité du code en prévalidation de la pull-request.
- la qualification de la feature sur sa propre branche par le Product Owner qui choisira alors d'accepter ou non la pull-request.

Ce type de workflow permet de mieux protéger la branche d'intégration pour utilisation dans le cadre de production de build de démos ou d'exploitation (seules les features sléectionnées pour la build étant versées dans sa branche, on évitera que du code appartenant à une feature rejetée ou en cours ne risque de causer des perturbations).

## `git pull --rebase`

On reprend notre divergeance précédente
```
            G---H (master)
           /
  A---B---C---D---E---F (origin/master)
```

Cette fois ci, plutôt que de merger la branche distante
