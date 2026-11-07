# 001 — Bénévoles : ouvrir un créneau et s'y inscrire

**Statut** : Validée
**PRD** : specs/prd-benevoles.md, TR1

## Contexte

Les responsables de discipline gèrent les inscriptions de bénévoles sur papier et des créneaux débordent. Cette tranche donne la plus petite version utile : ouvrir un créneau avec une capacité, et y inscrire des bénévoles sans jamais dépasser cette capacité.

## User stories

- En tant que responsable de discipline, je veux ouvrir un créneau avec une capacité, afin de fixer le nombre de bénévoles dont j'ai besoin.
- En tant que bénévole, je veux m'inscrire sur un créneau, afin d'accueillir le public de la discipline.

## Scénarios d'acceptation

### AS1 — Ouvrir un créneau

- **Étant donné** une discipline existante
- **Quand** le responsable ouvre un créneau avec une heure de début, une heure de fin postérieure et une capacité de 2
- **Alors** le créneau est enregistré avec ces valeurs, pour cette discipline, sans aucun inscrit.

### AS2 — Créneau invalide

- **Étant donné** une heure de fin égale ou antérieure au début, ou une capacité inférieure à 1, ou un champ absent
- **Quand** le responsable ouvre le créneau
- **Alors** c'est refusé avec un message de validation, et aucun créneau n'est enregistré.

### AS3 — Discipline ou créneau inconnu

- **Étant donné** une discipline ou un créneau qui n'existe pas
- **Quand** le responsable y ouvre un créneau, ou un bénévole s'inscrit sur le créneau
- **Alors** c'est refusé : la discipline ou le créneau est introuvable, et rien n'est enregistré.

### AS4 — Inscrire un bénévole

- **Étant donné** un créneau de capacité 2 avec une place libre
- **Quand** un bénévole s'inscrit avec son nom et son email
- **Alors** il est enregistré sur ce créneau avec son nom et son email, et le créneau compte un inscrit de plus.

### AS5 — Inscription invalide

- **Étant donné** un nom vide ou fait d'espaces, ou un email absent ou mal formé
- **Quand** le bénévole s'inscrit
- **Alors** c'est refusé avec un message de validation, et le créneau garde ses inscrits.

### AS6 — Créneau plein

- **Étant donné** un créneau de capacité 2 déjà occupé par deux bénévoles
- **Quand** un troisième bénévole s'inscrit
- **Alors** c'est refusé avec un message de conflit disant que le créneau est plein, et le créneau garde exactement deux inscrits.

### AS7 — Dernière place

- **Étant donné** un créneau de capacité 2 avec un seul inscrit, ou un créneau de capacité 1 sans inscrit
- **Quand** un bénévole s'inscrit
- **Alors** il est accepté, et le créneau est plein.

## Hors périmètre

- La consultation d'un créneau et de ses inscrits, et le refus d'un même email deux fois (TR2).
- La désinscription et l'écran (TR3).
- La modification ou la suppression d'un créneau, les emails de confirmation, les comptes des bénévoles : sans compte, tout le monde peut tout faire.

## Questions ouvertes

- Les refus sont décrits avec des mots, la demande ne donnant ni statut ni code. Hypothèse : le plan les choisit.
- Un créneau a une heure de début et de fin (hypothèse du PRD) ; aucune règle ne refuse un créneau dans le passé.
