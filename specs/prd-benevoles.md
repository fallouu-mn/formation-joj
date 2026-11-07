# PRD — Bénévoles

**Statut** : Validé

## Problème

Chaque discipline a besoin de bénévoles pour accueillir le public. Les responsables de discipline gèrent aujourd'hui les inscriptions sur papier : ils ne voient pas en temps réel si un créneau est plein, et des créneaux débordent. Pendant les Jeux, un créneau surchargé ou mal suivi se voit tout de suite à l'accueil du public.

## Utilisateurs

- Responsable de discipline : ouvre des créneaux pour sa discipline, avec le nombre de bénévoles dont il a besoin, et veut savoir qui est inscrit.
- Bénévole : cherche un créneau et s'y inscrit, ou s'en retire s'il ne peut plus venir.

## Proposition

Le responsable ouvre un créneau avec une capacité, les bénévoles s'y inscrivent, et l'application refuse toute inscription au-delà de la capacité. La règle est tenue par l'application et non par une feuille que chacun remplit à la main, ce qui supprime le débordement à la source.

## Mesure du succès

Pendant la démo, un créneau de capacité 2 accepte deux bénévoles et refuse le troisième.

## Tranches

### TR1 — Ouvrir un créneau et y inscrire un bénévole

Le responsable ouvre un créneau pour une discipline, avec une capacité, et un bénévole s'y inscrit. L'inscription est refusée dès que la capacité est atteinte. Cette tranche suffit seule à la démo du succès.

### TR2 — Consulter un créneau et ses inscrits

Le responsable et le bénévole consultent un créneau et la liste de ses inscrits. Un même email ne peut pas s'inscrire deux fois sur le même créneau.

### TR3 — Se désinscrire, et l'écran

Un bénévole se désinscrit d'un créneau et libère sa place. Un écran permet de voir un créneau, de s'y inscrire et de s'en désinscrire.

## Hors périmètre

- La modification ou la suppression d'un créneau.
- Les emails de confirmation.
- Les comptes des bénévoles : un bénévole s'identifie seulement par son nom et son email.

## Hypothèses et risques

- Hypothèse : un créneau appartient à une discipline qui existe déjà dans l'application. Si ce n'est pas le cas, il faudra décider qui crée les disciplines.
- Hypothèse : sans compte, personne ne vérifie qu'un email est le bon, et un bénévole peut se désinscrire en donnant l'email d'un autre. Risque accepté pour l'instant.
- Hypothèse : un créneau a une heure de début et une heure de fin, que le texte du besoin ne détaille pas.
- Hypothèse : tout le monde peut ouvrir un créneau, car il n'y a pas de compte pour distinguer le responsable.
- Risque : deux inscriptions simultanées sur la dernière place ne doivent pas dépasser la capacité.
