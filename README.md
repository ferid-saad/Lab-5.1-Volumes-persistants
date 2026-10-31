# Lab 5.1 : Volumes persistants

**Auteur :** Ferid Saad
**Formation :** Master DevOps & Cloud, ISET Tozeur
**Cluster :** k3s-vbox (master, worker1, worker2)
**Namespace :** tp-k8s

## Objectifs

- Constater la perte de données d'un conteneur et d'un emptyDir
- Créer un PVC et observer sa liaison avec un PV (provisionnement dynamique)
- Démontrer qu'une donnée survit à la suppression du Pod
- Repérer où les données sont stockées avec local-path
- Observer la suppression d'un PVC et diagnostiquer un PVC bloqué en Pending

## Tableau récapitulatif

| Emplacement | Survit au redémarrage du conteneur | Survit à la suppression du Pod |
|---|---|---|
| Système de fichiers du conteneur | Non | Non |
| emptyDir | Oui | Non |
| PVC | Oui | Oui |
## Étapes

###k exec temp -- sh -c 'echo "dans emptyDir" > /donnees/fichier.txt; echo "dans le conteneur" > /tmp/ephemere.txt'
k exec temp -- ls /donnees /tmp Étape 1 : Namespace et StorageClass
_à compléter_

### Étape 2 : Perte de données avec emptyDir
Pod `temp` (nginx:1.26-alpine) avec un volume `emptyDir` monté sur `/donnees` (fichier `temp.yaml`).

```bash
k apply -f temp.yaml
k wait --for=condition=Ready pod/temp --timeout=60s
k exec temp -- sh -c 'echo "dans emptyDir" > /donnees/fichier.txt; echo "dans le conteneur" > /tmp/ephemere.txt'
k exec temp -- ls /donnees /tmp
```

Avant le redémarrage : `/donnees` contient `fichier.txt` et `/tmp` contient `ephemere.txt`.

![avant demarrage](captures/tp5.1-emptyDir.png)

Redémarrage du conteneur :

```bash
k exec temp -- nginx -s stop
sleep 10
k get pod temp
k exec temp -- ls /donnees /tmp
```

Résultat : `RESTARTS` passe à 1. `/donnees` contient toujours `fichier.txt`, mais `/tmp` est vide (`ephemere.txt` a disparu).

![aprés demarrage](captures/tp5.1-emptyDir-apres-demmarage.png)

Suppression puis recréation du Pod :

```bash
k delete pod temp
k apply -f temp.yaml
k wait --for=condition=Ready pod/temp --timeout=120s
k exec temp -- ls /donnees
k delete pod temp
```

Résultat : `/donnees` est vide.

Conclusion :

- Un emptyDir est lié au **Pod** : il survit au redémarrage du conteneur, mais disparaît avec le Pod.
- Le système de fichiers du conteneur est perdu dès que le conteneur redémarre.

Incident : pendant cette étape, le master a été saturé (`soft lockup`, `TLS handshake timeout`), ce qui a retardé les commandes. Après stabilisation, la suppression et la recréation du Pod ont été refaites normalement.

Points de contrôle :

- [x] Après le redémarrage du conteneur, `fichier.txt` existait toujours et `ephemere.txt` avait disparu
- [x] Après la suppression du Pod, `/donnees` était vide

### Étape 3 : Créer un PVC
Création du PVC `data` (1 Gi, `ReadWriteOnce`, StorageClass `local-path`), fichier `pvc.yaml`.

```bash
k apply -f pvc.yaml
k get pvc,pv
k describe pvc data | tail -n 5
```

Résultat :

- Le PVC `data` est `Pending`.
- Aucun PV n'est associé à ce PVC. Les PV visibles dans `k get pv` appartiennent à d'autres namespaces du cluster (`elearning`).
- Les événements de `describe` indiquent que le PVC attend un premier consommateur (`WaitForFirstConsumer`).

![PVC data en Pending](captures/etape3-pvc-pending.png)

Explication : avec `local-path`, le `volumeBindingMode` est `WaitForFirstConsumer`. Le PV n'est créé que lorsqu'un Pod utilise le PVC, ce qui permet de le placer sur le nœud où le Pod sera planifié.

Points de contrôle :

- [x] `k get pvc` affiche `Pending`
- [x] `describe` indique que le PVC attend un premier consommateur (`WaitForFirstConsumer`)

![Événement WaitForFirstConsumer](captures/etape3-wait.png)

### Étape 4 : Utiliser le PVC dans un Pod
Pod `app` (nginx:1.26-alpine) qui monte le PVC `data` sur `/usr/share/nginx/html` (fichier `app.yaml`).

```bash
k apply -f app.yaml
k get pvc -w
k get pvc,pv
k get pod app -o wide
PV=$(k get pvc data -o jsonpath='{.spec.volumeName}')
k describe pv $PV | grep -E -A6 "Node Affinity|Source"
```

Résultat :

- Le PVC `data` passe de `Pending` à `Bound` dès que le Pod `app` est créé (premier consommateur).
- Un PV `pvc-d5234694-29f0-41a8-b8a3-28c11a28a778` (1Gi, RWO, `local-path`) a été créé automatiquement : c'est le provisionnement dynamique.
- Le Pod s'exécute sur le nœud `master`.
- Le PV porte une affinité vers le nœud `master` (`kubernetes.io/hostname`).

![PVC data passe de Pending à Bound](captures/etape4-pvc-bound.png)

![Affinité du PV vers le nœud](captures/etape4-affinite-pv.png)

Écriture d'une page web dans le volume et lecture par nginx :

```bash
k exec app -- sh -c 'echo "Bonjour, donnee persistante" > /usr/share/nginx/html/index.html'
k exec app -- wget -qO- http://localhost
```

Résultat : nginx répond `Bonjour, donnee persistante`.

![Page servie par nginx](captures/etape4-page-nginx.png)

Repérage du dossier sur le nœud qui héberge le Pod :

```bash
sudo ls -R /var/lib/rancher/k3s/storage/
```

Résultat : le dossier `pvc-d5234694-29f0-41a8-b8a3-28c11a28a778_tp-k8s_data` existe et contient `index.html`. Avec `local-path`, les données sont donc stockées directement sur le disque du nœud.

![Dossier du volume sur le nœud](captures/etape4-dossier-noeud.png)

Points de contrôle :

- [x] Le PVC est `Bound` à un PV créé automatiquement
- [x] Le PV porte une affinité vers le nœud du Pod
- [x] La page est servie par nginx, et `index.html` existe dans le dossier du nœud

### Étape 5 : Prouver la persistance
Suppression du Pod `app`, puis recréation avec le même `app.yaml` (le PVC `data` n'est pas touché).

```bash
k delete pod app
k get pvc
k apply -f app.yaml
k wait --for=condition=Ready pod/app --timeout=120s
k get pod app -o wide
k exec app -- wget -qO- http://localhost
```

Résultat :

- Pendant que le Pod n'existe plus, le PVC `data` reste `Bound` : le PVC et le PV vivent indépendamment du Pod.
- Le nouveau Pod affiche `Bonjour, donnee persistante` : la donnée a survécu à la suppression du Pod.
- Le nouveau Pod s'exécute sur le nœud `master`, le même que le précédent. L'affinité du PV `local-path` vers ce nœud impose ce placement.

![PVC data toujours Bound sans Pod](captures/etape5-pvc-bound-sans-pod.png)

![Page conservée après recréation du Pod](captures/etape5-persistance.png)

Conclusion : contrairement à l'emptyDir (étape 2), les données d'un PVC survivent à la suppression du Pod.

Points de contrôle :

- [x] Le PVC est resté `Bound` pendant que le Pod n'existait plus
- [x] Le nouveau Pod affiche le même message
- [x] Il s'exécute sur le même nœud que le Pod précédent

### Étape 6 : Supprimer le PVC
_à compléter_

### Étape 7 : Diagnostiquer un PVC en Pending
_à compléter_

## Réponses aux questions de réflexion
_à compléter_

## Nettoyage
_à compléter_
