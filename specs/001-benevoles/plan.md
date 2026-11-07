# 001 — Bénévoles : ouvrir un créneau et s'y inscrire : plan

**Statut** : Validé

## 1. Résumé

On construit le module `volunteers` de l'API : un responsable de discipline peut y ouvrir un créneau avec une capacité, et un bénévole peut s'y inscrire avec son nom et son email, jamais au-delà de cette capacité. À la fin des tâches, on peut le montrer avec l'API : un créneau de capacité 2 accepte deux bénévoles et refuse le troisième. Le travail est découpé en cinq tâches, dans l'ordre des couches : domaine, inscription, cas d'usage, base de données, puis routes HTTP. Sont laissés de côté : la consultation des inscrits, le refus d'un même email deux fois, la désinscription et l'écran (TR2 et TR3).

Les mots de la spec et leur nom dans le code :

| Dans la spec | Dans le code      | Ce que c'est                                                             |
| ------------ | ----------------- | ------------------------------------------------------------------------ |
| créneau      | `Slot`            | une plage horaire d'une discipline, avec une capacité maximale           |
| inscription  | `Registration`    | la place prise par un bénévole sur un créneau, avec son nom et son email |
| capacité     | `capacity`        | le nombre maximal d'inscriptions d'un créneau (au moins 1)               |
| inscrits     | `registeredCount` | le nombre d'inscriptions actuelles du créneau                            |
| responsable  | (aucun)           | pas de compte : personne n'est identifié, tout le monde peut ouvrir      |

## 2. Fichiers par couche

On crée le module `volunteers` dans `apps/api/src/modules/volunteers/`, organisé comme le module Disciplines : un dossier par couche (`domain`, `application`, `infra`, `http`).

- **shared** : `packages/shared/src/volunteers.ts` contient les schémas Zod d'entrée (`createSlotSchema`, `registerVolunteerSchema`, `slotIdParamsSchema`) et de réponse (`slotSchema`, `registrationSchema`) ; on l'exporte depuis `packages/shared/src/index.ts`.
- **domain** :
  - `domain/slot.ts` : l'entité `Slot`. `Slot.create()` vérifie la fin après le début et la capacité d'au moins 1 ; `Slot.restore()` relit la base ; `slot.register()` refuse l'inscription si le créneau est plein.
  - `domain/registration.ts` : l'entité `Registration`. `Registration.create()` vérifie le nom (non vide une fois les espaces retirés, 100 caractères au plus) et l'email (forme `x@y`).
  - `domain/volunteers.errors.ts` : `VolunteerError extends AppError`, avec les codes de la section 4.
- **application** :
  - `application/slot.repository.ts` : le port `SlotRepository` (classe abstraite) : `insert(slot)`, `findById(id)` (le créneau et son `registeredCount`) et `addRegistration(registration)`.
  - `application/testing/in-memory-slot.repository.ts` : le repository en mémoire, qui refuse aussi une inscription au-delà de la capacité.
  - `application/use-cases/open-slot.ts` : `OpenSlot` vérifie que la discipline existe (`DisciplineRepository`), crée le créneau et l'enregistre.
  - `application/use-cases/register-volunteer.ts` : `RegisterVolunteer` charge le créneau, appelle `slot.register()`, enregistre l'inscription puis relit le créneau pour renvoyer le compte exact.
  - `application/volunteers.application.spec.ts` : les tests des cas d'usage.
- **infra** :
  - `infra/volunteers.schema.ts` : les tables `slots` et `registrations` (Drizzle), puis une migration générée.
  - `infra/slot.mapper.ts` : `toDomain()` et `toRow()` pour les créneaux et les inscriptions.
  - `infra/drizzle-slot.repository.ts` : l'implémentation Drizzle. `addRegistration()` s'exécute dans une transaction qui verrouille la ligne du créneau, recompte, puis insère : deux inscriptions simultanées sur la dernière place ne dépassent pas la capacité.
  - `infra/drizzle-slot.repository.int.spec.ts` : le test d'intégration sur PGlite.
- **http** :
  - `http/volunteers.controller.ts` : controller fin (DTO `createZodDto`, appel d'un cas d'usage, sérialisation).
  - `POST /api/disciplines/:disciplineId/slots` ouvre un créneau et renvoie `201` avec le créneau (`id`, `disciplineId`, `startsAt`, `endsAt`, `capacity`, `registeredCount`).
  - `POST /api/slots/:slotId/registrations` inscrit un bénévole et renvoie `201` avec l'inscription (`id`, `slotId`, `name`, `email`, `createdAt`) et le créneau à jour (`slot`).
  - `volunteers.module.ts` relie le port à Drizzle ; il importe `DisciplinesModule`. On l'ajoute aux `imports` de `apps/api/src/app.module.ts`.
  - `apps/api/src/modules/disciplines/disciplines.module.ts` : on y ajoute `exports: [DisciplineRepository]`, pour que `OpenSlot` vérifie l'existence de la discipline.
  - `apps/api/test/volunteers.e2e.spec.ts` : le test e2e de l'API.
- **web** : rien. L'écran est hors périmètre de cette tranche (TR3) ; la fausse API des tests (`apps/web/src/mocks/`) est complétée à ce moment-là.

## 3. Données

On stocke les créneaux et leurs inscriptions. Migration : `pnpm db:generate --name=create_slots_and_registrations`.

- Table `slots` : `id` (uuid, clé primaire), `discipline_id` (uuid, clé étrangère vers `disciplines.id`, `on delete cascade`), `starts_at` et `ends_at` (timestamptz, précision 3), `capacity` (entier), `created_at` (timestamptz).
  - La clé étrangère protège AS3 : pas de créneau pour une discipline inconnue. Le `cascade` garde la suppression d'une discipline possible (spec 000).
  - Deux contraintes `check` (`capacity >= 1`, `ends_at > starts_at`) protègent AS2 si le domaine était contourné.
- Table `registrations` : `id` (uuid, clé primaire), `slot_id` (uuid, clé étrangère vers `slots.id`, `on delete cascade`, indexée), `name` (varchar 100), `email` (varchar 254), `created_at` (timestamptz).
  - La clé étrangère protège AS3 (inscription sur un créneau inconnu). Pas d'unicité sur `(slot_id, email)` : c'est la règle de TR2.
- La capacité n'est pas une contrainte SQL : elle dépend d'un compte, elle est tenue par le verrou de la ligne du créneau (section 2, `addRegistration()`).

## 4. Erreurs

Chaque refus de la spec devient une erreur métier (`AppError`), avec un code stable et un statut HTTP. Une entrée qui ne passe pas le schéma Zod donne `400 VALIDATION_FAILED` (comportement commun à toute l'API).

| Code AppError                     | Catégorie  | HTTP | Quand                                               | Scénario |
| --------------------------------- | ---------- | ---- | --------------------------------------------------- | -------- |
| `SLOT_END_BEFORE_START`           | VALIDATION | 400  | la fin est égale ou antérieure au début             | AS2      |
| `SLOT_CAPACITY_INVALID`           | VALIDATION | 400  | la capacité est inférieure à 1                      | AS2      |
| `DISCIPLINE_NOT_FOUND` (existant) | NOT_FOUND  | 404  | on ouvre un créneau sur une discipline inconnue     | AS3      |
| `SLOT_NOT_FOUND`                  | NOT_FOUND  | 404  | on inscrit un bénévole sur un créneau inconnu       | AS3      |
| `REGISTRATION_NAME_REQUIRED`      | VALIDATION | 400  | le nom est vide ou fait d'espaces                   | AS5      |
| `REGISTRATION_EMAIL_INVALID`      | VALIDATION | 400  | l'email est absent ou mal formé                     | AS5      |
| `SLOT_FULL`                       | CONFLICT   | 409  | le créneau a déjà autant d'inscrits que sa capacité | AS6      |

## 5. Tests par scénario

Chaque scénario de la spec est prouvé par au moins un test, au niveau le plus bas qui suffit.

| Scénario | Fichier de test                                                                                            | Niveau                     | Ce que le test vérifie                                                                                                                           |
| -------- | ---------------------------------------------------------------------------------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| AS1      | `domain/slot.spec.ts`, `apps/api/test/volunteers.e2e.spec.ts`                                              | unitaire, e2e              | le créneau garde ses valeurs et `registeredCount` vaut 0 ; la route répond `201` avec le créneau                                                 |
| AS2      | `domain/slot.spec.ts`, `apps/api/test/volunteers.e2e.spec.ts`                                              | unitaire, e2e              | fin ≤ début et capacité 0 lèvent `SLOT_END_BEFORE_START` et `SLOT_CAPACITY_INVALID` ; la route répond `400` et n'enregistre rien                 |
| AS3      | `application/volunteers.application.spec.ts`, `apps/api/test/volunteers.e2e.spec.ts`                       | application, e2e           | discipline inconnue : `DISCIPLINE_NOT_FOUND` ; créneau inconnu : `SLOT_NOT_FOUND` ; les routes répondent `404`                                   |
| AS4      | `domain/slot.spec.ts`, `infra/drizzle-slot.repository.int.spec.ts`, `apps/api/test/volunteers.e2e.spec.ts` | unitaire, intégration, e2e | l'inscription garde nom et email ; le compte passe à 1 ; la base relit l'inscription ; la route répond `201`                                     |
| AS5      | `domain/registration.spec.ts`, `apps/api/test/volunteers.e2e.spec.ts`                                      | unitaire, e2e              | nom vide ou email mal formé : `REGISTRATION_NAME_REQUIRED` / `REGISTRATION_EMAIL_INVALID` ; la route répond `400` et le compte ne bouge pas      |
| AS6      | `domain/slot.spec.ts`, `infra/drizzle-slot.repository.int.spec.ts`, `apps/api/test/volunteers.e2e.spec.ts` | unitaire, intégration, e2e | `SLOT_FULL` ; trois inscriptions lancées ensemble sur la capacité 2 en laissent exactement deux ; la route répond `409` avec le code `SLOT_FULL` |
| AS7      | `domain/slot.spec.ts`, `apps/api/test/volunteers.e2e.spec.ts`                                              | unitaire, e2e              | la dernière place est acceptée (capacité 2 avec un inscrit, capacité 1 vide) et `registeredCount` égale `capacity`                               |

## 6. Tâches

- [ ] T01 [001-AS1, AS2] Écrire d'abord `apps/api/src/modules/volunteers/domain/slot.spec.ts` (« 001-AS1: opens a slot with no registration », « 001-AS2: rejects an end before the start », « 001-AS2: rejects a capacity below 1 »), puis `packages/shared/src/volunteers.ts`, `domain/slot.ts` et `domain/volunteers.errors.ts`
- [ ] T02 [001-AS4, AS5, AS6, AS7] Écrire d'abord `domain/registration.spec.ts` et les tests `register` de `domain/slot.spec.ts` (« 001-AS4: registers a volunteer », « 001-AS5: rejects an empty name », « 001-AS5: rejects a malformed email », « 001-AS6: refuses a registration on a full slot », « 001-AS7: accepts the last place »), puis `domain/registration.ts` et `slot.register()`
- [ ] T03 [001-AS3, AS4, AS6] Écrire d'abord `application/volunteers.application.spec.ts` (« 001-AS3: rejects an unknown discipline », « 001-AS3: rejects an unknown slot », « 001-AS4: registers and returns the updated count », « 001-AS6: refuses the third volunteer of a slot of 2 »), puis `application/slot.repository.ts`, `application/testing/in-memory-slot.repository.ts` et les cas d'usage `OpenSlot` et `RegisterVolunteer`
- [ ] T04 [001-AS4, AS6] Écrire d'abord `infra/drizzle-slot.repository.int.spec.ts` (« 001-AS4: reads back a slot and its registration », « 001-AS6: keeps the capacity under simultaneous registrations »), puis `infra/volunteers.schema.ts`, la migration (`pnpm db:generate --name=create_slots_and_registrations`), `infra/slot.mapper.ts` et `infra/drizzle-slot.repository.ts`
- [ ] T05 [001-AS1, AS2, AS3, AS4, AS5, AS6, AS7] Écrire d'abord `apps/api/test/volunteers.e2e.spec.ts` (« 001-AS1: opens a slot (201) », « 001-AS2: rejects an invalid slot (400) », « 001-AS3: answers 404 for an unknown discipline or slot », « 001-AS4: registers a volunteer (201) », « 001-AS5: rejects an invalid volunteer (400) », « 001-AS6: refuses the third volunteer with 409 SLOT_FULL », « 001-AS7: accepts the last place »), puis `http/volunteers.controller.ts`, `volunteers.module.ts`, l'export de `DisciplineRepository` dans `disciplines.module.ts` et l'import dans `app.module.ts`
