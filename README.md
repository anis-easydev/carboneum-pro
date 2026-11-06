# Carboneum v4

Refonte complète de Carboneum : une API Node.js modulaire (Hono), une base SQLite aux contraintes renforcées, un moteur de calcul pur et testé, et un front React. Hébergement sur un VPS en Tunisie, sans service externe.

> Statut : socle livré et hébergeable sur VPS (phase 1). Reste la parité fonctionnelle avec la v3 (voir « Ce qui est fait / phase 2 »).

## Arborescence

```
v4/
├─ packages/
│  ├─ core/       moteur de calcul GES pur (GHG Protocol, ISO 14064-1, BEGES) + tests de parité v3
│  ├─ schema/     schémas Zod partagés API ↔ front
│  └─ db/         migrations SQL (0001 → 0009)
├─ apps/
│  ├─ api/        API Hono sur Node : routes/ services/ (métier) + platform/ (SQLite, fichiers, tâches, SMTP, PDF)
│  └─ web/        Vite + React + TanStack Query
├─ tools/
│  └─ migrate-v3/ ETL v3 → v4 avec rapport de parité sur les données réelles
└─ deploy/        nginx, systemd, deploy.sh / rollback.sh / cli.sh, guide d'hébergement
```

## Démarrer en local

Node.js ≥ 22.15 (24 LTS recommandé). Aucune dépendance Cloudflare : la base est un fichier SQLite.

```bash
corepack enable
pnpm install
cp apps/api/.env.example apps/api/.env    # puis renseigner PEPPER et ENC_KEY
pnpm db:migrate       # crée apps/api/data/carboneum.sqlite
pnpm dev:api          # http://localhost:8787 (rechargement automatique)
pnpm dev:web          # http://localhost:5173
pnpm test             # moteur (dont 200 inventaires de parité v3) + API sur la vraie plateforme Node/SQLite
```

Les services techniques (base, fichiers, file de tâches, e-mail SMTP, PDF Chromium, anti-robot) sont dans
`apps/api/src/platform/`, derrière les contrats de `platform/types.ts` : le code métier n'en dépend pas directement.

## Héberger (VPS en Tunisie)

Voir **[deploy/README.md](deploy/README.md)** : installation, déploiement en une commande
(`deploy/deploy.sh`, avec sauvegarde, migrations, contrôle de santé et retour arrière automatique), exploitation.

## Migration des données v3

```powershell
cd tools/migrate-v3
./export-v3.ps1 -Database <base-v3> -Remote       # lecture seule sur la v3
pnpm migrate --enc-key <ENC_KEY de la v4>         # → out/*.sql + out/report.json
```

- **Rapport de parité** : chaque inventaire est recalculé avec `back/calc.js` (v3) et le moteur v4. Le moindre écart, ou une erreur bloquante, donne un code de sortie 2 et la migration n'est pas importée.
- Importer les fichiers `out/*.sql` dans l'ordre, sur une base v4 **vide** (`wrangler d1 execute carboneum-v4 --remote --file …`).
- Appeler `POST /api/admin/periods/seal-migrated` pour calculer l'empreinte des exercices verrouillés.
- Mots de passe : les hash v3 sont repris tels quels (`legacy$…`) et réécrits au format v4 à la première connexion. Personne n'a à réinitialiser son mot de passe.
- Clés API : même algorithme de hash, donc les clés restent valides. En revanche, la route change : `/api/ingest-activity` devient `/api/v1/activities` (format documenté dans `packages/schema`).
- Les tables non migrées en phase 1 sont listées dans `report.json → notMigrated`.

## Changements de fond par rapport à la v3

| Sujet | v3 | v4 |
|---|---|---|
| Backend | `worker.js` de 1,1 Mo, 180 `if (path === …)` | routeur Hono, une route par fichier, services, moteur pur |
| Identifiants | `AUTOINCREMENT` exposés (IDOR) | UUIDv7 non devinables |
| Isolation entre entreprises | jointure via `inventories` | `org_id` sur chaque ligne, middleware `requireOrgAccess`, `INSERT … SELECT … WHERE org_id` |
| Verrouillage d'exercice | contrôlé par le front et le worker | **triggers SQL** : la base refuse toute écriture |
| Facteurs | modifiables, historiques séparés | jeux versionnés **immuables**, valeur copiée sur chaque ligne, FTS5 |
| Sessions | JWT + relecture `token_version` | jeton opaque hashé (`sessions`), révocation immédiate, MFA TOTP |
| Mots de passe | PBKDF2 100k | PBKDF2 100k + **pepper** (secret hors base), 12 caractères minimum |
| Secrets | webhooks et jetons fournisseurs en clair | AES-256-GCM pour les secrets à relire, SHA-256 pour les jetons |
| Audit | `audit_log` modifiable | ajout seul (triggers) + **scellement quotidien chaîné** dans R2 |
| E-mails, webhooks | appels synchrones, SMTP brut | Queues avec rejeu + DLQ, webhooks signés HMAC, anti-SSRF |
| Montants | `REAL`, devises mélangées (`DT`/`EUR`) | entiers en unités mineures, ISO 4217 (`TND` = 1/1000) |
| Dates | `DATETIME` et `TEXT` mélangés | ISO 8601 UTC partout |
| Concurrence | dernier écrit gagne | verrou optimiste (`revision`, `updated_at`) → 409 explicite |

## Conformité

| Exigence | Mécanisme |
|---|---|
| ISO 14064-1 §7.2 / GHG Protocol | facteurs immuables et versionnés, `factor_value`, `factor_source` et `calc_version` figés par ligne |
| ISO 14064-1 §7.3 | propagation quadratique de l'incertitude, identique à la v3 (test de parité) |
| ISO 14064-3 | exercice `locked` imposé par la base, `snapshot_hash`, contrôle `/integrity`, table `verifications` |
| BEGES (art. R229-47, méthode v5) | postes 1.1 → 2.2, PRG AR6 imposé dans `computeBegesTotals` |
| CSRD / ESRS / VSME | `esg_disclosures` versionné par norme, double matérialité, MDR-A/T |
| CBAM | référentiels officiels immuables (`cbam_reference_versions`), données exploitant par organisation |
| RGPD (UE) / loi tunisienne 2004-63 | D1 et R2 en juridiction UE, IP tronquées, consentements prouvés, export `/me/export`, effacement `/me/delete` + anonymisation à J+30, registre `privacy_requests` |
| Sécurité (OWASP ASVS L2) | CSRF par origine + SameSite=Lax, rate limiting natif, anti-énumération, verrouillage de compte, CSP stricte, validation Zod de chaque entrée, erreurs sans fuite technique |

## Ce qui est fait / phase 2

**Fait** : schéma D1 complet (7 migrations, tous les domaines v3), authentification complète (MFA, reset, invitations, RGPD), organisations, membres, sites, exercices (cycle de vie et verrouillage), lignes d'inventaire (unitaire, en masse, verrou optimiste), suppressions de carbone, exclusions, facteurs (recherche, facteurs propres, import ADEME), justificatifs R2, API v1 idempotente, webhooks, portail fournisseur, abonnements, cron (scellement, purge, anonymisation), ETL v3 avec parité, front de saisie d'inventaire.

**Phase 2 — faite** (logique dans `packages/core`, puis route, puis écran) :

| Module | Moteur (`packages/core`) | API | Parité v3 |
|---|---|---|---|
| Consolidation groupe | `consolidation.ts` : déduction intragroupe, sous-total BEGES France / indicatif, incertitude, N-1 à périmètre constant, année de référence, alertes de méthode | `/api/accounts/:id/consolidation`, simulation, exclusions, paramètres, recalcul de l'année de référence (§6.4.2), détection des changements de périmètre, validation | tests unitaires |
| Multi-sites | `siteBreakdown`, `simulateSiteClosure` | `/sites-breakdown`, `/sites-simulate-closure` | ✔ (`site-consolidation.js`) |
| Plan de transition | `transition.ts` : trajectoire top-down / bottom-up, réintégration des crédits cédés, suivi | `/transition/actions`, `/targets`, `/trajectory`, `/monitoring` | port ligne à ligne |
| CSRD / ESRS | `esg.ts` : catalogue des 11 thèmes (champs v3), contrôles de cohérence, complétude, double matérialité | `/esg/:year/:topic` (verrou optimiste, historique champ par champ, approbation), matérialité, MDR-A / MDR-T | mêmes bornes et contrôles |
| Empreinte produit (ISO 14067) | `pcf.ts` : phases du cycle de vie, nomenclature, rapprochement cat.1, incertitude, clé de comparabilité ISO 14026 | `/products` (calcul serveur, conservation des allocations ≤ 100 %), PCR | voir écart ci-dessous |
| Projets (ISO 14064-2) | `projects.ts` : 6 principes, réduction nette (fuites, tampon) | `/projects`, rapports de suivi (brouillon → soumis → vérifié) | mêmes contrôles |
| Vérification (ISO 14064-3) | `assuranceStatement` | `/verification/:periodId` : mandat, indépendance, constats, avis ; l'exercice passe en `verified` si l'empreinte est intacte | — |
| CBAM | `cbam/streams.ts` (15 types de flux, éq. 12 à 55, PFC, CKD, cogénération, CEMS), `cbam/see.ts`, `cbam/defaults.ts` | `/cbam` : installations, processus, flux, biens, répartitions (molaire éq. 43), SEE, valeurs par défaut officielles, calculateurs, communication annexe IV ; référentiels versionnés avec vérification à quatre yeux | ✔ (`cbam-calc.js`, 4 scénarios sectoriels) |
| Rapports | `services/reports.ts` : GES et BEGES en HTML imprimable | aperçu, archivage R2 avec SHA-256, validation formelle liée à l'empreinte | — |
| ETL | ESG éclaté par thème ; CBAM, produits, nomenclatures, projets, suivis | | |

**Écart volontaire avec la v3 (PCF)** : la v3 recalculait un poste lié à un produit à partir d'un objet partiel (sans libellé ni source), ce qui appliquait le RFI aux facteurs aériens « avec traînées » (double compte des effets non-CO2) et ignorait la détection des fluides frigorigènes. La v4 utilise exactement le même calcul que l'inventaire. Les PCF concernés peuvent donc baisser après migration : c'est une correction, à signaler aux clients.

**Phase 3 — faite** :
- évaluation réglementaire (`packages/core/src/standards.ts`, port de `sustainability-standards.js`) : applicabilité CSRD post-Omnibus, ESRS 2023 / révisés selon l'exercice, ESRS 2, points clés par thème, norme volontaire (UE) 2026/1560 B1-B11 (rubrique B3 calculée directement depuis l'inventaire) → `GET /esg/:year/assessment` ;
- durabilité au niveau groupe → `GET /api/accounts/:id/esg` : mode de rapport sur base consolidée, couverture des points clés par entité, agrégats (sommes, fourchettes de pourcentages, textes par entité), norme volontaire agrégée ;
- PDF natif : tâche de fond `render_pdf` (Cloudflare Browser Rendering, binding `BROWSER` optionnel), PDF archivé à côté du HTML validé avec sa propre empreinte ;
- écrans : saisie CBAM (installations, processus, flux sources par type), produits (nomenclature, aperçu PCF calculé par le serveur), panneau de vérification (constats, réponses, avis), import Excel des lignes d'inventaire (modèle téléchargeable, contrôle local avec le schéma partagé, import tout ou rien).

**Reste à faire** :
- export au format officiel de la Commission (CBAM Communication Template, Excel / XML) — à faire à partir du modèle officiel en vigueur ;
- écrans d'administration (référentiels CBAM, import ADEME), paramétrage des objectifs et de la consolidation dans l'interface ;
- i18n EN / AR (RTL) ;
- tests e2e Playwright, et premier passage complet de `pnpm typecheck && pnpm test` (le code n'a encore jamais été exécuté).
