# Un peu de terrorisme git

## Petit retour rapide sur `git pull` en mode `--merge` (comportement par défaut)

Prenons une divergence d'historique (ex: travail de deux personnes concurrentes sur une même branche).

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
- permet de bien visualiser la parallélisation des développement et donc de bien identifier les risques de conflits d'intégration (incompatibilité entre deux features).
- simplicité du workflow: on n'utilise que 3 commandes : `commit`, `pull`, `push`.
- permet de faciliter l'éviction d'un feature problématique pour la production d'une démo ou d'une build de production si tous les commits liés à cette feature sont bien isolés dans une branche dédiée avant d'être mergée.

**Difficultés:**
- rend l'utilisation de la consultation d'historique ou de l'outil `git blame` moins clair.
- sur une branche de feature ayant duré longtemps, le commit de merge, en cas de conflit, peut contenir un patch volumineux et les changements se trouvent alors mal compartimenté (on identifie plus difficilement la raison pour laquelle une modification a été faite).
- l'outil de recherche d'origine de régression `git bissect` devient en pratique quasi inutilisable.
- avec la croissance d'une équipe, la proportion de commits de merge par rapport aux commits de développement augmente polluant l'historique.

## vers un workflow typé `merge` plus poussé

Petit détail gênant dans l'historique final précédent, notre commit **`I`** est nommé par défaut **merged origin/master into master** ce qui faisait sens dans la structure d'arborescence d'historique d'avant push, mais beaucoup moins après. Cette incongruité de nommage de commit peut facilement être résolu en ne travaillant jamais en local sur la branche d'intégration mais sur une branche de feature (ex: **`feat/my-nice-feature`**).

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

Si notre historique a été complexifié avec un commit *inutile* de plus, on obtient un historique avec une sémantique plus claire puisque l'on aura nos commits **`I`** et **`J`** respectivement nommés:
**`I`**: *merged origin/master into feat/my-nice-feature* on comprend bien qu'il s'agit d'une resynchronisation d'une branche de feature avec l'état de l'intégration.

**`J`**: *merged feat/my-nice-feature into master* on comprend bien qu'une feature vient d'être ajoutée à l'intégration.

Ce type de workflow merge se prête, lorsque le versement d'une feature à l'intégration est effectuée au moyen d'un pull-request à:
- l'audit de code par le responsable qualité du code en prévalidation de la pull-request (la validation pouvant être automatique et sans relecture pour les équipiers de confiance ayant fait leurs preuves mais approfondie pour les nouveaux équipier ayant plus de chance de causer des problèmes).
- la qualification de la feature sur sa propre branche par le Product Owner qui choisira alors d'accepter ou non la pull-request.

**Ce type de workflow permet de mieux protéger la branche d'intégration pour utilisation dans le cadre de production de build de démos ou d'exploitation (seules les features sélectionnées pour la build étant versées dans sa branche, on évitera que du code appartenant à une feature rejetée ou en cours ne risque de causer des perturbations).**

## `git pull --rebase`

On reprend notre divergence précédente
```
            G---H (feat/my-nice-feature)
           /
  A---B---C---D---E---F (origin/master)
```

Cette fois ci, plutôt que d'effectuer un `git pull [--merge] origin master` on effectuera `git pull --rebase origin master` pour obtenir:

```
               G---H (orphan commits, will be gc'ed later)
             /
            /           G'---H' (feat/my-nice-feature)
           /           /
  A---B---C---D---E---F (origin/master)
```
Le fonctionnement de `git pull --rebase <remote> <branch>` est l'équivalant de stasher tous ses changements locaux avant de faire un pull classique puis de réappliquer le stash.
Dans le détail, ce qu'à fait git est exactement équivalent à:
```
git reset --hard origin/master
git cherry-pick G
git cherry-pick H
```

On obtient alors une branche `feat/my-nice-feature` qui est un `fast-forward` de la branche d'intégration.

En cas de conflits lors d'un `git pull --merge`, ceux-ci sont résolu interactivement de la même manière que lors d'un merge, sauf qu'au lieu de stocker le patch de résolution des conflits dans le commit de merge ce qui a tendance à rendre obscure le résultat d'un `git blame`, les conflits sont résolus au niveau de chaque cherry-pick.

On pourra conserver la facilité d'éviction de feature d'un worflow pur merge en forçant la création d'un commit de merge (`git merge --no-ff`) lors du reversement de la branche de feature dans celle d'intégration.

## `git rebase -i|--interactive` au service d'un historique propre et de titres de commits formalisés tout en permettant d'utiliser des commits fréquents en tant que points de restauration sans se prendre la tête.

Prenons un historique divergeant correspondant à une feature plus lourde avec prise de risque (et utilisation de commits régulier pour créer un point de restauration):
```
            G---H---I---J---K---L---M (feat/my-nice-feature)
           /
  A---B---C (master)---D---E---F (origin/master)
```

avec pour historique `git log G..HEAD`:
```
G    WIP du soir avant de partir du bureau
H    WIP avant de tout péter dans le modèle...
I    WIP modèle mis à jour, erreurs Eclipse dans tous les sens
J    compilation OK
K    tests IHM
L    //fix erreurs tests tests manuels
M    //fix erreurs test automatiques
```

Bon ok, j'ai fait mon boulot, la feature me semble bien fonctionner, par contre mon historique est dégueulasse...

`git rebase -i|--interactive C`
ou
`git rebase -i|--interactive G^`
ou encore plus simple si l'on a laissé `master` dans l'état où il se trouvait lors de la création de la branche de feature:
`git rebase -i|--interactive master`

git va lancer un éditeur de texte avec un contenu comme suit:
```
pick G    WIP du soir avant de partir du bureau
pick H    WIP avant de tout péter dans le modèle...
pick I    WIP modèle mis à jour, erreurs Eclipse dans tous les sens
pick J    compilation OK
pick K    tests IHM
pick L    //fix erreurs tests tests manuels
pick M    //fix erreurs test automatiques
```

et on va modifier la séquence par exemple comme ceci:
```
reword G    WIP du soir avant de partir du bureau
fixup H    WIP avant de tout péter dans le modèle...
reword I    WIP modèle mis à jour, erreurs Eclipse dans tous les sens
fixup J    compilation OK
fixup L    //fix erreurs tests tests manuels
reword K    tests IHM
fixup M    //fix erreurs test automatiques
```

On notera que l'ordre a été changé: le commit **`L`** n'étant que la correction d'une erreur effectuée dans le commit **`J`**, on a remonté le premier juste après le dernier pour les regrouper.
Lors de la fermeture de l'éditeur, git va revenir à l'état C, puis suivre la séquence demandée selon le verbe au début de chaque ligne:
- `pick`: le commit est cherry-pické.
- `reword`: comme `pick` mais git demande de saisir un nouveau message de commit.
- `edit`: comme `pick` sauf que le rebase est mis en pause après ce cherry-pick et la main vous est rendue permettant d'effectuer des actions après ce commit (par exemple amender le commit ou créer un nouveau commit entre celui-ci et le prochain.
- `fixup`: le commit est est fusionné avec le précédant en conservant le message du précédent
- `squash`: le commit est fusionné avec le précédent en ajoutant agglomérant les deux messages.

Si pour les `reword` de **`G`**, **`I`** et **`K`** on saisit:
- **`G`**: \[US-XXX\] Fichiers de configuration de la feature X
- **`I`**: \[US-XXX\] Mise à jour du modèle machin
- **`K`**: \[US-XXX\] Tests IHM de la feature X ajoutés

on obtient alors:
```
            G---H---I---J---K---L---M (orphan commits, will by gc'ed)
          /
          | X---Y---Z (feat/my-nice-feature)
          |/
  A---B---C (master)---D---E---F (origin/master)
```

avec pour historique de `master..feat/my-nice-feature`
```
X    [US-XXX] Fichiers de configuration de la feature X
Y    [US-XXX] Mise à jour du modèle machin
Z    [US-XXX] Tests IHM de la feature X ajoutés
```

dont le commit **`X`** est une fusion des commit **`G`** et **`H`**, **`Y`** est une fusion des commits **`I`**, **`J`** et **`L`** et finalement **`Z`** est la fusion des commits **`K`** et **`M`**.

Nickel, j'ai bossé comme un porc en local, mais ni vu ni connu je pousse un historique clean !
