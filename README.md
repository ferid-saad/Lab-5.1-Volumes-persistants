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
| emptyDir | _à compléter (étape 2)_ | _à compléter (étape 2)_ |
| PVC | _à compléter (étapes 4 et 5)_ | _à compléter (étape 5)_ |

## Étapes

### Étape 1 : Namespace et StorageClass
_à compléter_

### Étape 2 : Perte de données avec emptyDir
_à compléter_

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
