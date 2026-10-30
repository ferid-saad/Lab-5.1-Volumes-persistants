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
| PVC | _à compléter (étapes 4 et 5)_ | _à compléter (étape 5)_ |

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
_à compléter_

### Étape 4 : Utiliser le PVC dans un Pod
_à compléter_

### Étape 5 : Prouver la persistance
_à compléter_

### Étape 6 : Supprimer le PVC
_à compléter_

### Étape 7 : Diagnostiquer un PVC en Pending
_à compléter_

## Réponses aux questions de réflexion
_à compléter_

## Nettoyage
_à compléter_
