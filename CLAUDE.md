# CLAUDE.md

Application d'organisation des JOJ Dakar 2026 : une API NestJS (`apps/api`), un front React (`apps/web`) et des schémas partagés (`packages/shared`), dans un monorepo pnpm. Commandes et structure détaillée : `README.md`.

## Méthodologie

- Toute fonctionnalité suit le cycle `/prd` → `/spec` → `/spec-plan` → `/tdd-task` → `/converge`, avec validation humaine après le PRD, la spec et le plan. Règles communes : `specs/constitution.md`.
- La spec dit le quoi (sans route, table ni composant), le plan dit le comment ; aucun besoin nouveau dans le plan.
- Test d'abord, vu en échec pour la bonne raison (`pnpm tdd:red <fichier>`), puis le code minimal ; `pnpm verify` avant chaque commit, `pnpm check` pour tout le dépôt.
- Une tâche = un commit qui cite la tâche et son scénario : `feat(benevoles): T03 refuse a full slot (001-AS2)`.
- Celui qui produit ne valide pas : la fonctionnalité est relue par le sous-agent `relecteur`, avec seulement la base git et le dossier de la spec.

## Architecture

- Un module Nest par domaine dans `apps/api/src/modules/<domaine>/`, découpé en `domain/`, `application/`, `infra/`, `http/`, sur le modèle de `disciplines`. Le transverse (config, base, erreurs, santé) est dans `apps/api/src/core/`.
- `domain/` est du TypeScript pur : aucun import de NestJS, Drizzle ou pg (vérifié par oxlint).
- `application/` : un port `<entité>.repository.ts` (classe abstraite), un cas d'usage par classe dans `use-cases/`, un repository en mémoire dans `testing/`. Le module relie le port à son implémentation Drizzle (`infra/`).
- Controller fin : valider (DTO `createZodDto`), appeler un cas d'usage, sérialiser (`toResponse`). Aucune règle métier.
- Les schémas Zod d'entrée et de réponse vivent dans `packages/shared/src/<nom>.ts`, exportés par `index.ts`, et servent à l'API comme au front.
- Front : une route par fichier dans `apps/web/src/routes/` (TanStack Router) ; chaque fonctionnalité dans `src/features/<nom>/` avec `api/` (appels HTTP séparés des hooks TanStack Query) et `components/` ; composants shadcn dans `src/components/ui/`.
- `apps/web/src/mocks/` est la fausse API des tests (MSW) : mêmes schémas et mêmes codes d'erreur que `apps/api`.
- Migrations générées par `pnpm db:generate --name=<sujet>`, jamais éditées à la main.

## Concepts appréciés

- Typage strict (`strict`, `noUncheckedIndexedAccess`), pas de `any` ; types dérivés des schémas avec `z.infer`.
- Entités à constructeur privé : `create()` applique les règles, `restore()` relit la persistance sans les rejouer ; les mutations (`rename()`) réutilisent les mêmes validateurs.
- Le domaine ne fait pas confiance à la validation HTTP : il revérifie ses invariants.
- Cas d'usage `@Injectable()` avec une seule méthode `execute()`. Le temps et les identifiants sont passés au domaine (`now`, `id`), pas lus par lui.
- Fonctions courtes et nommées ; un court commentaire `/** … */` dit le pourquoi, pas le quoi.

## Bonnes pratiques

- Toute erreur métier est une sous-classe d'`AppError` (code stable + catégorie `VALIDATION`, `NOT_FOUND`, `CONFLICT` ou `BUSINESS_RULE`), déclarée dans `domain/<nom>.errors.ts`. Seul `ProblemDetailsFilter` la transforme en HTTP (RFC 9457).
- Une entrée est validée par un schéma Zod de `packages/shared`, jamais à la main dans le controller.
- Aucun secret dans le code : les valeurs sensibles vont dans `apps/api/.env` (ignoré par git). Un hook refuse l'écriture d'un secret probable.
- Chaque scénario d'acceptation a au moins un test ; `pnpm spec:check <spec> --trace` vérifie la traçabilité.
- API : tests unitaires du domaine, tests des cas d'usage avec le repository en mémoire, intégration Drizzle et e2e (supertest) sur PGlite en mémoire.
- Front : tests sur la fausse API (`fixture`, `mockControl`), rendus par `renderRoute` / `renderWithProviders`, réseau vérifié par `trackRequests()` ; un audit d'accessibilité (`*.a11y.test.tsx`) par état d'écran.
- Jamais de `--no-verify` ni de `LEFTHOOK=0` : lefthook lance `pnpm verify --staged` avant chaque commit.

## Nommage

- Fichiers en kebab-case : `create-discipline.ts`, `drizzle-discipline.repository.ts`, `discipline.mapper.ts`, `disciplines.schema.ts`.
- Classes en PascalCase, cas d'usage nommés par un verbe : `CreateDiscipline`, `ListDisciplines` ; erreurs `<Entité>Error` et fabriques en camelCase (`disciplineNotFound(id)`).
- Codes d'erreur en majuscules préfixés par l'entité : `DISCIPLINE_NAME_TAKEN`. Constantes partagées en majuscules : `DISCIPLINE_NAME_MAX_LENGTH`.
- Schémas Zod en camelCase suffixés `Schema` (`createDisciplineSchema`), types en PascalCase (`CreateDisciplineInput`, `DisciplineResponse`).
- Tests API : `*.spec.ts`, `*.int.spec.ts`, `*.e2e.spec.ts` ; tests front : `<fichier>.test.tsx` à côté du fichier testé, ouvert par un commentaire qui dit ce qu'il prouve.
- Titres de tests en anglais, préfixés par le scénario : `it('001-AS2: rejects a full slot', …)`.
- Commits Conventional Commits en anglais : `feat(api): add disciplines`, `fix(sdd): verify only staged files before a commit`.
- Code et identifiants en anglais ; commentaires et documentation en français.
