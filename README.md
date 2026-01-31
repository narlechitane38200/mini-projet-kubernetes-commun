# Mini-projet: Déployez PayMyBuddy à l’aide de manifests ☺

## Contexte
Il s'agit de déployer l'application SpringBoot PayMyBuddy à l'aide de manifests yaml que vous allez écrire et non pas avec helm,
ce qui vous permettra de comprendre plus en détails la mise en place d'une application dans un cluster Kubernetes.

[PayMyBuddy](https://github.com/eazytraining/PayMyBuddy/)

## Etapes à réaliser

- Déployez PayMyBuddy en suivant les étapes suivantes
     - Créez un deployment mysql avec un seul replicat
     - Créez un service de type clusterIP pour exposer vos pods mysql
     - Créez un deployment paymybuddy avec les bonnes variables d’environnement pour se connecter à la base de données mysql. Vous trouverez les variables d'environnement nécessaire dans ce fichier: [Dockerfile](https://github.com/eazytraining/PayMyBuddy/blob/main/Dockerfile)
     - Votre deployment devra stocker les données PayMyBuddy sur un volme mounté dans le /data de votre nœud
     - Créez un service de type nodeport pour exposer le frontend wordpress
- Nous vous conseillons d’utiliser les manifests pour réaliser cet exercice
- A la fin de votre travail, poussez vos manifests sur github et envoyez nous le lien de votre repo à
eazytrainingfr@gmail.com et nous vous dirons si votre solution respecte les bonnes pratiques et si votre solution
bonne. Nous vous proposerons aussi notre solution/
