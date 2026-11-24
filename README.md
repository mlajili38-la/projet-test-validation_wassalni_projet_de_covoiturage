# WASSALNI — Campagne de test et validation

Mini-projet **« Atelier Test et validation des logiciels »**, Licence 3 Technologies de l'information, spécialité DSI.
Application testée : **WASSALNI**, application web de covoiturage (Symfony 7.4, PHP 8.4, MariaDB), avec trois rôles (passager, conducteur, administrateur) et un module de réclamations.

Étudiant : Mohamed Lajili Amir Menzli 

## Résultat en bref

| | Campagne finale du 11/11/2026 | Corrections S2 (21/11) | **Version finale (21/11)** |
|---|---|---|---|
| Révision | `cc50095` | `72cff37` | **`1ed214b`** |
| Régression PHPUnit (unitaire, intégration, HTTP, API) | 87 tests | 110 tests | **131 tests, 365 assertions, OK** |
| Anomalies ouvertes | 32 (dont 8 S2) | 24 (0 S2) | **23 (0 S2, 15 S3, 8 S4)** |
| API (Postman / Newman) | 19 requêtes, 32/35 | 19 requêtes, 35/35 | **35 requêtes, 59/59** (dont l'API REST des réclamations) |
| Interface (Playwright) | 20 / 23 | 20 / 23 | **23 / 23** (accessibilité corrigée) |
| Performance (k6) | Seuils respectés | Seuils respectés | Seuils respectés |
| Couverture des lignes | 34,65 % | 41,33 % | **44,69 %** |
| Décision | Refus de livraison | Validation conditionnelle | **Validation conditionnelle (confirmée)** |

## Où trouver quoi

| Besoin | Emplacement |
|---|---|
| **Rapport final** (illustré) | [`docs/rapport-final/rapport-final.md`](docs/rapport-final/rapport-final.md) · [PDF](docs/rapport-final/rapport-final.pdf) |
| Dossier de test complet (plan, exigences, matrice, catalogue, registre, bilan) | [`docs/dossier-tests/`](docs/dossier-tests/README.md) |
| Plan de test et critères de sortie | [`02-strategie-calendrier.md`](docs/dossier-tests/02-strategie-calendrier.md) |
| Matrice de traçabilité | [`09-matrice-tracabilite.md`](docs/dossier-tests/09-matrice-tracabilite.md) |
| Registre des anomalies | [`07-registre-anomalies.md`](docs/dossier-tests/07-registre-anomalies.md) |
| Bilan et décision de livraison | [`14-semaine10-bilan-decision.md`](docs/dossier-tests/14-semaine10-bilan-decision.md) |
| Écarts avec le référentiel de l'énoncé (RéclaGest) | [`15-ecarts-enonce.md`](docs/dossier-tests/15-ecarts-enonce.md) |
| Livrables PDF des semaines 1 à 4 | [`docs/`](docs/README.md) |
| Preuves d'exécution (sorties, rapports JUnit, captures) | [`preuves/`](preuves/) — rejeu final : [`preuves/semaine10/apres-ameliorations/`](preuves/semaine10/apres-ameliorations/) |

## Tests automatisés

| Niveau | Outil | Emplacement |
|---|---|---|
| Unitaire | PHPUnit (suite `unitaire`) | `tests/Entity`, `tests/Service`, `tests/Controller` |
| Intégration et base de données | PHPUnit (suite `integration`) | `tests/Integration` |
| Système HTTP et relecture en base | PHPUnit (suite `fonctionnel`) | `tests/Functional` |
| Anomalies ouvertes (échecs attendus) | PHPUnit, groupe `anomalie-ouverte` | réparti dans les suites |
| API (routes JSON et API REST `/api/reclamations`) | Postman + Newman ; PHPUnit | `postman/`, `scripts/run-newman.js`, `tests/Functional/Api` |
| Interface | Playwright (Chrome) | `tests/Playwright` |
| Performance | k6 | `tests/k6` |
| Intégration continue | GitHub Actions | `.github/workflows/ci.yml` |

## Types de tests réalisés et outils

| Type de test | Ce qui est testé | Outil | Volume |
|---|---|---|---|
| **Unitaire** | Règles métier isolées : entités (`Reservation`, `Reclamation`, `Vehicule`, `User`), service de réservation, contraintes de validation | **PHPUnit** (stubs et mocks) | 33 tests en semaine 6, puis UT-12 et UT-13 |
| **Intégration** | Application et base de données ensemble : CRUD, clés étrangères, transactions, rollback, cascades | **PHPUnit** + **Doctrine** + **MariaDB** (base `wassalni_test`) | 9 tests (IT-01 à IT-07…) |
| **Système / fonctionnel** | Application complète par requêtes HTTP, avec relecture en base : les 28 cas TC du catalogue | **PHPUnit** (`WebTestCase` de Symfony) | La majorité des 131 tests de régression |
| **Acceptation** | Scénarios métier vus par l'utilisateur (réclamations, droits, deux sessions simultanées) | **Playwright** (Chrome) | 10 scénarios (ST-01 à ST-10) |
| **API** | Routes JSON et API REST des réclamations : méthodes, codes HTTP, JSON, droits, erreurs | **Postman** + **Newman**, et **PHPUnit** | 35 requêtes, 59 assertions ; 14 tests PHPUnit (API-01 à API-14) |
| **Interface (de bout en bout)** | Parcours critique : connexion → recherche → réservation → vérification en base | **Playwright** | 23 tests navigateur |
| **Exploratoire** | Double clic, session expirée, accès direct par URL pour chaque rôle | **Playwright** (sessions outillées) | 4 sessions (EXPL-01 à EXPL-04) |
| **Performance (charge)** | Temps de réponse sous 5 utilisateurs pendant 30 s | **k6** | Seuils : p95 < 1 500 ms, moins de 1 % d'échecs |
| **Accessibilité** | Respect des règles WCAG A/AA sur 4 pages, navigation au clavier | **axe-core** (dans Playwright) | 4 pages auditées |
| **Sécurité élémentaire** | CSRF, accès aux ressources d'autrui, fuite de données, cookie de session, dépendances vulnérables | **PHPUnit**, **Playwright**, **composer audit** | Tests CA-14, CA-15, TC-45… |
| **Concurrence** | Deux réservations simultanées sur la dernière place | **PHPUnit** (deux connexions à la base) | IT-07 |
| **Statique** (sans exécuter l'application) | Syntaxe PHP, conteneur Symfony, schéma de base de données, test d'architecture | `php -l`, `lint:container`, `doctrine:schema:validate`, **PHPUnit** | À chaque exécution de la CI |
| **Régression** | Rejouer tous les tests après chaque modification | **PHPUnit** + **GitHub Actions** (CI) et script `run-regression.ps1` | 131 tests |
| **Couverture de code** (boîte blanche) | Part du code exécutée par les tests | **PCOV** + PHPUnit | 44,69 % des lignes |

**Des tests pour vérifier les tests :**

- **Tests d'anomalies ouvertes** (groupe `anomalie-ouverte` dans PHPUnit) : chacun prouve un défaut non corrigé. Quand le défaut est corrigé, son test passe dans la régression.
- **Vérification par mutation** : des défauts ont été introduits volontairement dans l'API pour vérifier que les tests échouent bien.

**Techniques de conception :** partitions d'équivalence, valeurs limites, tables de décision et transitions d'état.

## Reproduire la campagne

Prérequis : PHP 8.4, Composer, Node.js, MariaDB avec une base `wassalni_test`, Chrome ; k6 pour la performance.

```powershell
composer install ; npm ci
# Créer .env.test.local (jamais versionné) avec : WASSALNI_TEST_PASSWORD=<12 caractères minimum>
php bin/console doctrine:database:create --env=test --if-not-exists
php bin/console doctrine:migrations:migrate --env=test --no-interaction

# Toujours vider puis reconstruire le cache de test avant d'exécuter (voir ANO-T06 et ANO-T08)
Remove-Item -Recurse -Force var\cache\test
php bin/console cache:warmup --env=test --no-debug

# Régression seule
vendor\bin\phpunit
# Anomalies encore ouvertes (échecs attendus)
vendor\bin\phpunit --group anomalie-ouverte
# Campagne complète : PHPUnit, anomalies, Playwright, Newman, k6
powershell -NoProfile -ExecutionPolicy Bypass -File scripts/run-regression.ps1 -Sortie preuves/semaine10/<nouveau-dossier>
```

Toutes les commandes travaillent exclusivement sur la base `wassalni_test` ; chaque outil vérifie le nom de la base avant d'agir.

## Branches et versions

| Branche ou tag | Contenu |
|---|---|
| `main` | Application de référence |
| `baseline-v0` (tag) | État initial avant la campagne |
| `v1.0.0` (tag) | Campagne finale du 11/11/2026 (décision NO-GO) |
| `test/campagne-finale` | Dossier de test et outillage de la campagne |
| `fix/anomalies-s2` | Corrections des 8 anomalies S2 et rejeu du 21/11/2026 |
| `amelioration/a11y-api` | Accessibilité (ANO-36), API REST des réclamations, ANO-39 |
| `v2.0.0` (tag) | Après corrections S2 (validation conditionnelle) |
