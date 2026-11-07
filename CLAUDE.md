# CLAUDE.md

Application d'organisation des JOJ Dakar 2026 : une API NestJS (`apps/api`), un front React (`apps/web`) et des schémas partagés (`packages/shared`), dans un monorepo pnpm. Commandes et structure détaillée : `README.md`.

## Méthodologie

- Toute fonctionnalité suit le cycle `/spec` → `/spec-plan` → `/tdd-task` → `/converge`, avec une validation humaine après la spec et après le plan. Principes : `specs/constitution.md`.
- Hors cycle, suivre l'ordre « Ajouter une fonctionnalité » du `README.md` : schéma partagé → domaine → cas d'usage → table et migration → repository Drizzle → controller et e2e → fausse API → écran.
- Un test d'abord, vu en échec pour la bonne raison (`pnpm tdd:red <fichier>`), puis le code minimal qui le fait passer.
- `pnpm verify` avant chaque commit (lancé par lefthook, jamais de `--no-verify`), `pnpm check` avant de déclarer une fonctionnalité terminée.
- Avant de conclure, faire relire par le sous-agent `relecteur` en ne lui donnant que la base git et le dossier de la spec.

## Architecture

- `packages/shared/src/<nom>.ts` : schémas Zod d'entrée et de réponse, exportés par `index.ts`. L'API et le front importent `@joj/shared`, jamais l'un l'autre.
- API : un module par domaine dans `apps/api/src/modules/<domaine>/`, sur le modèle de `disciplines` : `domain/`, `application/` (port, `use-cases/`, `testing/`), `infra/` (table Drizzle, mapper, repository), `http/` (controller) et `<domaine>.module.ts`, ajouté à `src/app.module.ts`.
- `domain/` est du TypeScript pur : aucun import de NestJS, Drizzle, `pg` ou PGlite (bloqué par oxlint), ni horloge ni hasard : l'identifiant et la date sont passés à l'entité.
- Le controller est fin : valider (DTO `createZodDto`), appeler un cas d'usage, sérialiser (`toResponse()`). Aucun `if` métier, aucun accès au repository.
- Le transverse est dans `apps/api/src/core/` (config, base, `AppError`, `ProblemDetailsFilter`). Les migrations sont générées par `pnpm db:generate`, jamais éditées.
- Front : `apps/web/src/features/<nom>/` avec `api/<nom>.api.ts` (appels HTTP validés par le schéma partagé), `api/<nom>.queries.ts` (hooks TanStack Query) et `components/` ; une route par fichier dans `src/routes/` ; shadcn dans `src/components/ui/`.
- `apps/web/src/mocks/` est la fausse API des tests (MSW) : chaque route de l'API y est reproduite, avec les mêmes schémas et codes d'erreur.

## Concepts appréciés

- TypeScript strict (`noUncheckedIndexedAccess`) : pas de `any`, types déduits des schémas (`z.infer`).
- Entité à constructeur privé : `create()` applique les règles, `restore()` relit la base sans les rejouer ; lecture par getters, changement par méthode métier (`rename()`).
- Une classe `@Injectable()` par cas d'usage, avec une seule méthode publique `execute()`.
- Le port de persistance est une classe abstraite dans `application/`, implémentée par Drizzle (`infra/`) et en mémoire (`application/testing/`).
- Fonctions courtes et nommées ; un court commentaire JSDoc en français qui dit le pourquoi.

## Bonnes pratiques

- Toute erreur métier est une sous-classe d'`AppError` (`<nom>.errors.ts`) : code stable en union littérale et catégorie VALIDATION, NOT_FOUND, CONFLICT ou BUSINESS_RULE. Seul `ProblemDetailsFilter` choisit le statut HTTP (RFC 9457) ; ni `throw new Error()` ni `HttpException` dans le code applicatif.
- Une entrée est validée par un schéma Zod de `packages/shared`, une réponse sérialisée par `@ZodSerializerDto` ; le domaine revérifie quand même ses invariants.
- Une unicité est vérifiée par le cas d'usage (message clair) et garantie par une contrainte en base.
- Tests : domaine et cas d'usage avec le repository en mémoire (`*.spec.ts`), repository sur PGlite en mémoire (`*.int.spec.ts`), API par supertest dans `apps/api/test/` (`*.e2e.spec.ts`), front sur la fausse API (`*.test.tsx` à côté du fichier, un audit `*.a11y.test.tsx` par route). Un test vérifie le statut et le corps ou le code d'erreur, pas seulement `toBeDefined`.
- Le front traduit les codes d'erreur pour l'utilisateur (`components/error-message.ts`) ; il n'affiche jamais le message brut de l'API.
- Aucun secret dans le code : les valeurs sensibles vont dans `apps/api/.env` (modèle : `.env.example`).

## Nommage

- Fichiers en kebab-case suffixés par leur rôle : `create-discipline.ts`, `disciplines.errors.ts`, `discipline.mapper.ts`, `drizzle-discipline.repository.ts`, `disciplines.schema.ts`, `disciplines.queries.ts`.
- Classes en PascalCase ; un cas d'usage est un verbe suivi de l'entité : `CreateDiscipline`, `ListDisciplines`. Port `DisciplineRepository`, implémentation `DrizzleDisciplineRepository`, erreur `DisciplineError`.
- Schémas en camelCase suffixés `Schema` (`createDisciplineSchema`), types `CreateDisciplineInput` et `DisciplineResponse`, constantes en majuscules (`DISCIPLINE_CATEGORIES`).
- Codes d'erreur en majuscules préfixés par l'entité (`DISCIPLINE_NAME_TAKEN`), créés par une fabrique en camelCase (`disciplineNotFound(id)`).
- Code, messages d'erreur et titres de tests en anglais ; commentaires et textes affichés en français. Un titre de test commence par son scénario : `it('001-AS2: rejects a full slot')` (`AS2:` seul dans `disciplines`).
- Commits Conventional Commits, le scope est le module ou la zone : `docs(task): …`, `feat(api): …` ; pendant `/tdd-task` : `feat(benevoles): T03 refuse a full slot (001-AS2)`.
