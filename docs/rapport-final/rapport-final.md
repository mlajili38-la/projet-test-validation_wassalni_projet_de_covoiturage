# Rapport final — Test et validation de l'application WASSALNI

**Mini-projet Licence 3 Technologies de l'information, spécialité DSI — Atelier Test et validation des logiciels**
Étudiants : Mohamed Lajili, Amir Menzli · Dépôt : `projet-test-validation_wassalni_projet_de_covoiturage` (branche `main`) · Version du rapport 3.0 · 24 novembre 2026

---

## Résumé

WASSALNI est une application web de covoiturage développée avec Symfony 7.4 et MariaDB. Elle met en relation des passagers et des conducteurs, et propose un module de réclamations traité par un administrateur. Ce rapport présente la campagne de test menée sur dix semaines pour répondre à une question : **l'application peut-elle être livrée ?**

La campagne s'est déroulée en trois temps.

1. **Campagne finale du 11/11/2026 (révision `cc50095`).** 87 tests de régression réussis, mais 48 tests démontrant **32 anomalies, dont 8 majeures (S2)** : failles CSRF, comptes bloqués capables de se connecter, fuite d'adresse e-mail, surréservation en cas d'accès simultané, parcours conducteur bloqué. **Décision : refus de livraison.**
2. **Corrections du 21/11/2026 (révision `72cff37`).** Les 8 anomalies majeures ont été corrigées ; chaque correction est prouvée par un test qui échouait avant et qui réussit après. **Décision : validation conditionnelle.**
3. **Version finale du 21/11/2026 (révision `1ed214b`).** Accessibilité corrigée, API REST des réclamations ajoutée et testée, un défaut latent découvert et corrigé. **Décision : validation conditionnelle confirmée.**

| Indicateur | 11/11 | **Version finale** |
|---|---|---|
| Tests de régression PHPUnit | 87 | **131 (365 assertions), tous réussis** |
| Anomalies ouvertes | 32, dont 8 S2 | **23, dont 0 S2** (15 S3, 8 S4) |
| Tests navigateur (Playwright) | 20 / 23 | **23 / 23** |
| Assertions API (Newman) | 32 / 35 | **59 / 59** (35 requêtes) |
| Performance (k6) | Seuils respectés | Seuils respectés (p95 : 250,8 ms) |
| Couverture des lignes | 34,65 % | **44,69 %** |
| Exigences réussies / partielles / échouées / non exécutées | 9 / 9 / 14 / 3 | **15 / 10 / 7 / 3** |

Tous les seuils de sortie du plan de test sont atteints. La décision reste pourtant **conditionnelle** : 7 exigences ne sont pas satisfaites et 15 anomalies moyennes restent ouvertes. Les conditions de livraison sont détaillées au chapitre 11.

### Sommaire

1. Présentation du système testé et de son environnement
2. Exigences, critères d'acceptation et risques
3. Stratégie et plan de test
4. Matrice de traçabilité
5. Catalogue des cas de test et jeux de données
6. Scripts automatisés et instructions d'exécution
7. Résultats d'exécution et preuves
8. Registre des anomalies et état de leur traitement
9. Couverture et suite de régression
10. Bilan de la campagne, limites et risques résiduels
11. Décision de validation et recommandations

---

## 1. Présentation du système testé et de son environnement

### 1.1 L'application

WASSALNI permet à des conducteurs de publier des trajets et à des passagers de les réserver. Elle comporte trois rôles :

| Rôle | Fonctions principales |
|---|---|
| **Passager** | Rechercher un trajet (ville de départ, destination, date), réserver une ou plusieurs places, annuler, consulter ses réservations, déposer un avis et une réclamation |
| **Conducteur** | Enregistrer ses véhicules, publier des trajets, consulter les demandes et les avis reçus |
| **Administrateur** | Valider ou supprimer les trajets, activer, bloquer ou supprimer les utilisateurs, traiter les réclamations, consulter le tableau de bord |

Les principaux objets métier sont `User`, `Vehicule`, `Trajet`, `Reservation`, `Avis` et `Reclamation`. Une réservation décrémente le stock de places du trajet ; une annulation le restitue. Une réclamation est créée au statut `EN_ATTENTE`, puis traitée par l'administrateur (`TRAITEE`).

### 1.2 Environnement de test

| Élément | Poste local (campagne) | Intégration continue |
|---|---|---|
| Système | Windows 11 | Ubuntu (GitHub Actions) |
| PHP / Symfony | PHP 8.4.27, Symfony 7.4 | PHP 8.4 |
| Base de données | MariaDB 10.4.32, base dédiée `wassalni_test` | Service MariaDB 10.4 éphémère |
| Tests | PHPUnit 12.5, Playwright 1.59 (Chrome), Newman 6.2, k6, axe-core, PCOV | PHPUnit, PCOV |

**Isolation.** Aucun test ne s'exécute sur la base réelle `wassalni`. Le suffixe `_test` est imposé par la configuration de l'environnement de test, et chaque outil vérifie le nom de la base avant d'agir. Les données sont fictives (comptes `@example.test`), créées par chaque test puis supprimées. La comparaison de l'état de la base avant et après l'exécution a confirmé qu'aucune donnée ne reste.

### 1.3 Écarts avec le référentiel de l'énoncé

L'énoncé décrit une application de référence, RéclaGest, et autorise l'utilisation d'une autre application à condition de documenter les différences. Ces écarts sont analysés dans le document [15](../dossier-tests/15-ecarts-enonce.md).

| Écart | Situation finale |
|---|---|
| Pas de champ priorité (E-01) | Recherche par priorité non testable |
| Titre remplacé par une liste fermée de sujets (E-02) | Bornes de 5 à 100 caractères vérifiées au niveau de l'entité |
| Pas de longueur maximale pour la description (E-03) | Borne de 1 000 caractères non applicable |
| Cycle de vie plus court (E-04) | 3 états (`EN_ATTENTE`, `EN_COURS`, `TRAITEE`) ; `EN_COURS` atteignable par l'API |
| Pas d'affectation à un agent (E-05) | Module hors périmètre |
| Pas d'historique des statuts (E-06) | Seuls le statut courant et les dates sont vérifiés |
| Filtrage des réclamations (E-07) | Par statut et par date dans l'API ; pas de priorité |
| Pas d'API REST (E-08) | **Résolu** : API REST des réclamations ajoutée et testée |

Ces écarts sont des différences de référentiel et non des anomalies. Aucune fonctionnalité absente n'a été inventée pour la tester.

---

## 2. Exigences, critères d'acceptation et risques

### 2.1 Distinctions de base

La campagne s'appuie sur les distinctions du cours :

- une **erreur** est une action humaine incorrecte ; elle introduit un **défaut** dans le code ; à l'exécution, le défaut provoque une **défaillance** observable. Exemple : écrire `message:` au lieu de `minMessage:` (erreur) donne une contrainte mal paramétrée (défaut), d'où une erreur 500 à l'ajout d'un véhicule (défaillance, ANO-35) ;
- la **vérification** contrôle la conformité aux spécifications (« construit-on le produit correctement ? ») ; la **validation** contrôle l'adéquation au besoin (« construit-on le bon produit ? »).

### 2.2 Exigences et critères d'acceptation

L'analyse de l'application a produit **35 exigences** (EX-01 à EX-35) et **22 critères d'acceptation** vérifiables (CA-01 à CA-22). Elles couvrent sept domaines :

| Domaine | Exigences | Exemples |
|---|---|---|
| Comptes et authentification | EX-01 à EX-05 | Seuls les comptes activés se connectent ; un compte bloqué ne se connecte pas |
| Véhicules et publication | EX-06 à EX-12 | Départ futur et arrivée après le départ ; prix strictement positif ; trajet publié « en attente » |
| Recherche | EX-13 à EX-15 | Filtres ville, date, prix ; seuls les trajets validés avec des places |
| Réservation | EX-16 à EX-20 | Réserver de 1 à n places ; prix total = places × prix ; places restituées à l'annulation |
| Avis et réclamations | EX-21 à EX-25 | Note de 1 à 5 après le trajet ; réclamation d'au moins 10 caractères |
| Administration | EX-26 à EX-30 | Valider ou supprimer les trajets ; statistiques exactes |
| Sécurité et cohérence | EX-31 à EX-35 | Accès limité à son rôle ; pas d'accès aux ressources d'autrui ; modifications par POST avec jeton CSRF ; cohérence des données |

Trois ambiguïtés ont été arbitrées par le responsable produit le 11/11/2026 : le fuseau horaire attendu (Africa/Tunis), le traitement direct d'une réclamation (`EN_ATTENTE → TRAITEE` accepté) et l'absence de règle sur le statut `EN_COURS` dans l'interface.

### 2.3 Cartographie des risques

Les **21 risques** ont été cotés selon leur probabilité (1 à 3) et leur impact (1 à 3). Leur produit donne la priorité de test.

| Risque | Domaine | P | I | Niveau |
|---|---|---|---|---|
| R4 Modération contournée | Trajets, administration | 3 | 3 | **9** |
| R15 Perte d'intégrité des données | Persistance | 3 | 3 | **9** |
| R21 Fuite d'adresse e-mail par une route publique | Données personnelles | 3 | 3 | **9** |
| R1 Surréservation en cas d'accès simultané | Réservation | 2 | 3 | **6** |
| R5 Compte bloqué, non activé ou lien expiré | Authentification | 2 | 3 | **6** |
| R6 Dates incohérentes | Trajets | 3 | 2 | **6** |
| R12 Statistiques fausses ou fixes | Tableaux de bord | 3 | 2 | **6** |
| R17 Trajets passés proposés et réservables | Recherche | 3 | 2 | **6** |
| R20 Actions qui modifient des données par simple lien | Administration | 2 | 3 | **6** |
| R9, R11, R14, R16, R18 | Avis, e-mails, performance, doublons, casse | 2 | 2 | 4 |
| R2, R3, R10, R19 | Droits par rôle, ressources d'autrui, téléphone exposé, immatriculations | — | — | 3 |
| R7, R8, R13 | Prix, restitution des places, réclamation trop courte | — | — | 2 |

Les risques de niveau 9 et 6 ont reçu les tests en priorité. Chacun a révélé au moins une anomalie, sauf R15 : l'intégrité de la base a été confirmée par les tests d'intégration.

---

## 3. Stratégie et plan de test

### 3.1 Niveaux, techniques et outils

| Niveau | Objet | Outil |
|---|---|---|
| **Unitaire** | Règles des entités, validations, service de réservation | PHPUnit, Arrange – Act – Assert, stubs et mocks |
| **Intégration** | Doctrine et MariaDB : CRUD, contraintes, transactions, cascades | PHPUnit, base `wassalni_test` |
| **Système** | Application complète par HTTP, avec relecture en base | PHPUnit (`WebTestCase`) |
| **Acceptation** | Scénarios métier par rôle | Playwright (Chrome) |
| **API** | Routes JSON et API REST des réclamations | Postman + Newman ; PHPUnit |
| **Non fonctionnel** | Performance, accessibilité, sécurité élémentaire, concurrence | k6, axe-core, PHPUnit |
| **Couverture** | Lignes et méthodes exécutées | PCOV |

### 3.2 Types de tests réalisés et outils

Le tableau ci-dessous récapitule tous les types de tests menés pendant la campagne, l'objet de chacun, l'outil utilisé et le volume réalisé.

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

- **tests d'anomalies ouvertes** (groupe `anomalie-ouverte` dans PHPUnit) : chacun prouve un défaut non corrigé ; quand le défaut est corrigé, son test passe dans la régression (test de confirmation) ;
- **vérification par mutation** : des défauts ont été introduits volontairement dans l'API pour vérifier que les tests échouent bien.

**Approches :** la campagne est principalement **boîte noire** (tests construits à partir des exigences) ; l'approche **boîte blanche** sert à mesurer la couverture et à vérifier, par un test d'architecture, que le contrôleur utilise bien le décrément atomique du stock.

**Techniques de conception :** partitions d'équivalence, valeurs limites, tables de décision et transitions d'état (détaillées au chapitre 5).

### 3.3 Critères d'entrée

Avant chaque campagne : révision Git consignée, règles ambiguës arbitrées ou marquées bloquantes, base `wassalni_test` migrée avec un schéma valide, comptes fictifs disponibles pour chaque rôle, intégration continue verte.

### 3.4 Critères de sortie

**Critères qualitatifs :** cas prioritaires exécutés et réussis ; aucun défaut S1 ou S2 ouvert ; aucune incohérence de stock ou de prix ; tous les niveaux exécutés sur la même révision ; aucune exigence critique sans test ; régression réussie après la dernière correction.

**Seuils quantitatifs** (ajoutés en version 2.0 du plan, avant le rejeu après corrections) :

| Indicateur | Seuil |
|---|---|
| Régression PHPUnit | 100 % de réussite |
| Couverture globale des lignes | ≥ 30 % |
| Couverture des classes critiques | ≥ 80 % chacune |
| Exigences exécutées | ≥ 90 % |
| Défauts S1 et S2 ouverts | 0 |
| Assertions API | 100 % |
| Performance (k6) | moins de 1 % d'échecs, p95 < 1 500 ms |
| Accessibilité | 0 violation critique sur le parcours |

Le seuil de couverture globale est volontairement modeste, car l'inscription et l'envoi d'e-mails sont hors périmètre. L'exigence porte surtout sur les classes critiques, où un défaut a le plus d'impact.

### 3.5 Règle de décision

| Décision | Condition |
|---|---|
| **Validation** | Tous les critères et tous les seuils atteints |
| **Validation conditionnelle** | 0 S1 et 0 S2 ; seuils principaux atteints ; écarts restants acceptés, datés et assortis de conditions |
| **Refus de livraison** | Au moins un défaut S1 ou S2, une incohérence de stock ou de prix, ou une régression en échec |

### 3.6 Organisation du travail

Le dépôt Git suit un flux simple : `main` est la référence ; la campagne est conduite sur `test/campagne-finale` ; les corrections sont faites sur des branches dédiées (`fix/anomalies-s2`, `amelioration/a11y-api`), fusionnées par pull request après rejeu. Chaque version est marquée par un tag : `baseline-v0` (état initial), `v1.0.0` (11/11), `v2.0.0` (corrections S2) et `v2.1.0` (version finale).

---

## 4. Matrice de traçabilité

La matrice ([09](../dossier-tests/09-matrice-tracabilite.md)) relie chaque exigence à ses risques, à ses critères, à ses tests, à son résultat et à ses preuves. Aucune exigence critique n'est restée sans test. État final des 35 exigences :

| Statut | Nombre | Exigences |
|---|---|---|
| **Réussi** | 15 | EX-03, 04, 06, 07, 15, 17, 18, 20, 22, 24, 25, 31, 33, 34, 35 |
| **Partiel** | 10 | EX-08, 13, 14, 16, 19, 21, 23, 27, 28, 32 |
| **Échoué** | 7 | EX-05, 09, 10, 11, 12, 26, 29 |
| **Non exécuté** | 3 | EX-01, 02, 30 (inscription et e-mails) |

Les corrections du 21/11 ont fait passer sept exigences d'« échoué » à « réussi » (EX-03, 04, 06, 33, 34, 35) ou à « partiel » (EX-16). Les sept exigences encore échouées concernent la modification du profil (fonction absente), les règles de publication des trajets (EX-09 à EX-12) et les statistiques des tableaux de bord (EX-26, EX-29).

---

## 5. Catalogue des cas de test et jeux de données

### 5.1 Techniques de conception

| Technique | Application dans le projet |
|---|---|
| **Partitions d'équivalence** | Nombre de places : classe valide (1 au stock) et classes invalides (0, négatif, décimal, texte, supérieur au stock) |
| **Valeurs limites** | Message de réclamation de 9 (refusé) et 10 caractères (accepté) ; véhicule de 0, 1, 9 et 10 places ; note d'avis de 0, 1, 5 et 6 ; mot de passe de 5 et 6 caractères |
| **Tables de décision** | Réservation selon le statut du trajet, sa date et le stock ; connexion selon un compte vérifié ou non, bloqué ou actif ; visibilité d'un trajet |
| **Transitions d'état** | Cycle de vie d'une réclamation, d'un trajet et d'une réservation, avec les transitions autorisées et interdites |
| **Tests positifs et négatifs** | Chaque fonctionnalité testée sur le cas nominal et sur les cas qui doivent être refusés |
| **Tests exploratoires** | Quatre sessions avec une charte : rafraîchissement, accès direct par URL, quantités, double clic et session expirée |

### 5.2 Les cas TC du catalogue

Le catalogue de la semaine 4 détaille **28 cas TC**, tous automatisés, plus des variantes par la voie JSON (« bis ») et 21 critères CA testés directement. Extrait :

| Cas | Objet | Résultat attendu |
|---|---|---|
| TC-03, TC-04 | Connexion d'un compte non vérifié, puis bloqué et réactivé | Aucune session pour un compte non vérifié ou bloqué |
| TC-09 à TC-14 | Publication d'un trajet : cas nominal, dates passées ou incohérentes, prix non numérique | Trajet enregistré ou refusé selon les règles |
| TC-15, TC-16 | Statut et visibilité d'un trajet publié | `EN_ATTENTE`, absent de la recherche avant validation |
| TC-18 à TC-20 | Recherche : trajets passés, complets ou en attente exclus ; filtre de prix ; casse | Seuls les trajets réservables affichés |
| TC-22 à TC-25 | Réservation : toutes les places, plus que le stock, quantités invalides, calcul du total | Stock et prix cohérents, refus des quantités invalides |
| TC-28 à TC-30 | Doublon, annulation puis nouvelle réservation, annulation d'un trajet passé | Une seule réservation ; places restituées ; refus après le départ |
| TC-33, TC-36 | Valeurs limites de la note d'avis et du message de réclamation | Bornes respectées |
| TC-38, TC-41, TC-45 | Validation par GET, pages d'administration pour un passager, route publique | Refus, 403, aucune donnée personnelle |
| TC-49 à TC-51 | Réservation d'un trajet passé ou non validé ; mauvais identifiants | Refus, message identique, aucune session |

### 5.3 Jeux de données

Les données sont créées par des fixtures déterministes (`tests/Support/CatalogueFixture.php` et la commande `app:tests:prepare-week7-9`) :

- **comptes :** passagers U-PASS1 et U-PASS2, conducteurs U-COND1 et U-COND2, administrateur U-ADMIN, compte non vérifié U-PASS-NV ;
- **trajets :** T-NOMINAL (validé, futur), T-P4 (4 places à 22 DT), T-FULL (complet), T-PEND (en attente), T-PAST (passé), T-FUT-EARLY ;
- **réservations :** RES-F1 (future), RES-PAST (trajet passé).

Le mot de passe des comptes fictifs n'est jamais écrit dans un fichier versionné : il est lu depuis `.env.test.local` ou depuis une variable d'environnement en CI.

---

## 6. Scripts automatisés et instructions d'exécution

### 6.1 Organisation des tests

| Dossier | Contenu |
|---|---|
| `tests/Entity`, `tests/Service`, `tests/Controller` | Suite `unitaire` |
| `tests/Integration` | Suite `integration` (Doctrine et MariaDB) |
| `tests/Functional` | Suite `fonctionnel` (HTTP), dont `Catalogue/` (cas TC) et `Api/` (API REST) |
| `tests/Playwright` | Parcours navigateur, scénarios d'états, explorations, accessibilité |
| `tests/k6` | Test de charge |
| `postman/` | Collection et environnement Newman |
| `scripts/` | `run-regression.ps1`, `run-newman.js`, `run-k6.js`, `generer-pdf.js` |

Les tests qui démontrent une anomalie encore ouverte appartiennent au groupe `anomalie-ouverte`. Ils sont exclus de la régression pour que celle-ci reste un indicateur fiable, et rejoués séparément.

### 6.2 Commandes

```powershell
composer install ; npm ci
# .env.test.local (non versionné) : WASSALNI_TEST_PASSWORD=<12 caractères minimum>
php bin/console doctrine:migrations:migrate --env=test --no-interaction

Remove-Item -Recurse -Force var\cache\test
php bin/console cache:warmup --env=test --no-debug

vendor\bin\phpunit                               # régression
vendor\bin\phpunit --group anomalie-ouverte      # anomalies ouvertes (échecs attendus)
powershell -NoProfile -ExecutionPolicy Bypass -File scripts/run-regression.ps1 -Sortie preuves/semaine10/<dossier>
```

Le script `run-regression.ps1` enchaîne, sur une même révision, PHPUnit (régression puis anomalies), Playwright, Newman et k6, et écrit une synthèse datée avec la révision testée.

### 6.3 Intégration continue

Le workflow GitHub Actions (`.github/workflows/ci.yml`) démarre une base MariaDB neuve, installe l'application, vérifie la syntaxe et le schéma, rejoue la régression avec la couverture, rejoue les anomalies ouvertes à titre informatif et publie les rapports.

Au début de la campagne, ses 12 premières exécutions échouaient toutes : le fichier `.env` n'était pas versionné et Symfony s'arrêtait dès l'installation des dépendances. Le workflow a été réparé le 21/11. Il est maintenant vert à chaque push.

![Figure 1](../../preuves/semaine10/S10_CI_verte.png)

**Figure 1** — Intégration continue verte après réparation du workflow.

---

## 7. Résultats d'exécution et preuves

### 7.1 Semaine 1 — Exploration initiale

Les premiers essais manuels ont révélé des comportements suspects, confirmés ensuite par des tests automatisés.

![Figure 2](../../preuves/semaine1/anonymise/S01_E10_get_last_user_email_sans_connexion.png)

**Figure 2** — `/get-last-user-email` renvoie une adresse e-mail sans connexion (copie anonymisée). Écart A-02, confirmé par TC-45 (ANO-22), corrigé le 21/11.

### 7.2 Semaine 5 — Transitions d'état, droits et exploration

Les scénarios ST-01 à ST-10 ont été joués dans Chrome. Neuf ont réussi ; ST-09 a été bloqué faute de règle sur le statut `EN_COURS`, puis déclaré conforme à la règle validée le 11/11.

![Figure 3](../../preuves/semaine5/S05_ST04_acces_interdit.png)

**Figure 3** — ST-04 : le passager B ouvre la réclamation du passager A par URL directe. Résultat : HTTP 403, aucune donnée révélée (EX-32).

![Figure 4](../../preuves/semaine5/S05_ST06_traitement_admin.png)

**Figure 4** — ST-06 : réponse de l'administrateur. Résultat : statut `TRAITEE`, réponse et date de traitement enregistrées.

![Figure 5](../../preuves/semaine5/S05_ST10_2_session_B_voit_stock_mis_a_jour.png)

**Figure 5** — ST-10 : deux sessions simultanées. Après la réservation du passager A, la session B voit 3 places ; l'annulation rend la place, et une annulation répétée ne la rend pas deux fois.

L'exploration a aussi révélé une erreur 500 sur le tableau de bord passager (ANO-34).

### 7.3 Semaine 6 — Tests unitaires

33 tests unitaires en Arrange – Act – Assert : valeurs limites des prix, des places, des notes et des dates ; un stub pour isoler le service de réservation ; un mock pour vérifier une interaction. Les tests unitaires ont révélé ANO-35 : une contrainte mal paramétrée faisait planter tout le formulaire de véhicule.

![Figure 6](../../preuves/semaine6/campagne-finale/S06_TESTS_UNITAIRES.png)

**Figure 6** — Sortie de la suite unitaire : 33 tests, 52 assertions, OK.

### 7.4 Semaine 7 — Intégration et base de données

9 tests d'intégration : CRUD, clés étrangères, unicité, rollback explicite, atomicité d'un enregistrement en échec, transactions, relations et cascades. Résultat : la base garantit l'intégrité ; les erreurs 500 observées à la suppression (ANO-23) viennent de l'absence de gestion de ces refus dans l'application.

![Figure 7](../../preuves/semaine7/campagne-finale/S07_TESTS_INTEGRATION.png)

**Figure 7** — Sortie des tests d'intégration : 9 tests, 29 assertions, OK.

### 7.5 Semaine 8 — API

La collection Postman a d'abord couvert les 4 routes JSON existantes (réservation, état de santé, suivi d'inscription). Le 21/11, une **API REST des réclamations** a été ajoutée :

| Requête | Rôle | Codes testés |
|---|---|---|
| `GET /api/reclamations` (filtres `statut`, `dateMin`, `dateMax`) | Passager : les siennes ; administrateur : toutes | 200, 400, 401 |
| `GET /api/reclamations/{id}` | Auteur ou administrateur | 200, 401, 403, 404 |
| `POST /api/reclamations` | Utilisateur connecté | 201 + `Location`, 400, 401, 415, 422 |
| `PATCH /api/reclamations/{id}` | Administrateur | 200, 401, 403, 404, 409, 415, 422 |
| `DELETE`, `PUT` | — | 405 |

L'API est couverte par 14 tests PHPUnit (API-01 à API-14) et 16 requêtes Newman. Une réclamation traitée est close : toute nouvelle modification renvoie 409. Les requêtes qui modifient des données exigent un contenu JSON (sinon 415), ce qui protège contre le CSRF.

**Efficacité des tests vérifiée par mutation.** Deux défauts ont été introduits volontairement : la suppression du contrôle du propriétaire, puis l'autorisation de transitions interdites. Chaque fois, un test a échoué (API-06, puis API-12).

![Figure 8](../../preuves/semaine8/S08_TESTS_API.png)

**Figure 8** — Synthèse Newman du 11/11 : 32 assertions sur 35 (les 3 échecs démontraient ANO-22). Version finale : 35 requêtes, 59 assertions sur 59.

### 7.6 Semaine 9 — Interface et non fonctionnel

**Parcours critique.** Connexion, recherche, réservation de 2 places et confirmation, vérifiées jusqu'en base (50 DT, stock de 4 à 2).

![Figure 9](../../preuves/semaine9/S09_PARCOURS_1_recherche.png)

**Figure 9** — Parcours critique, étape 1 : recherche d'un trajet par le passager.

![Figure 10](../../preuves/semaine9/S09_PARCOURS_3_confirmation.png)

**Figure 10** — Parcours critique, étape 3 : confirmation de la réservation.

**Performance (k6).** 5 utilisateurs virtuels pendant 30 secondes, sur la recherche, le détail d'un trajet, la connexion et l'état de santé. Version finale : 0 % d'échec, 100 % des vérifications réussies, p95 de 250,8 ms (seuil : 1 500 ms).

**Accessibilité (axe-core).** Le 11/11, 3 pages présentaient des violations critiques : des étiquettes de formulaire non reliées à leur champ (ANO-36). Après correction, aucune violation critique ne subsiste sur les 4 pages auditées. Un défaut de contraste reste sur la page de connexion (ANO-37, mineur).

![Figure 11](../../preuves/semaine9/S09_A11Y_recherche.png)

**Figure 11** — Page de recherche auditée par axe-core le 11/11, avant la correction d'ANO-36.

**Sécurité élémentaire.** `composer audit` ne signale aucune vulnérabilité ; le cookie de session est `HttpOnly` et `SameSite` ; la page d'erreur ne divulgue ni trace ni chemin de fichier. Les en-têtes de protection HTTP sont absents (recommandation REC-01).

**Concurrence.** Deux transactions entrelacées sur la dernière place produisaient 2 réservations (ANO-38). Après correction, une seule réservation est confirmée.

### 7.7 Semaine 10 et rejeux — Comparaison des trois campagnes

| Niveau | 11/11 (`cc50095`) | 21/11 (`72cff37`) | **Final (`1ed214b`)** |
|---|---|---|---|
| Régression PHPUnit | 87 / 87 | 110 / 110 | **131 / 131** |
| Tests d'anomalies ouvertes | 48 | 27 | 27 |
| Playwright | 20 / 23 | 20 / 23 | **23 / 23** |
| Newman | 32 / 35 | 35 / 35 | **59 / 59** |
| k6 (p95) | 217,0 ms | 392,8 ms | 250,8 ms |
| Couverture des lignes | 34,65 % | 41,33 % | **44,69 %** |

Les variations du p95 de k6 restent très en dessous du seuil. Elles s'expliquent par le serveur de développement mono-processus, dont les temps varient d'une exécution à l'autre ; le scénario k6 ne réserve pas.

Les preuves de chaque campagne sont conservées séparément : `preuves/semaine10/campagne-finale/`, `apres-corrections/` et `apres-ameliorations/`.

---

## 8. Registre des anomalies et état de leur traitement

### 8.1 Méthode

Une anomalie n'entre au registre qu'**après reproduction par exécution**. Chaque entrée précise les préconditions, les étapes, le résultat attendu et le résultat observé, une preuve, la sévérité, la priorité et le test qui la démontre.

| Sévérité | Signification | Priorité | Signification |
|---|---|---|---|
| S1 | Critique : perte de données, sécurité grave, application inutilisable | P1 | Corriger avant toute livraison |
| S2 | Majeure : fonction essentielle incorrecte ou faille de sécurité | P2 | Corriger avant la mise en production |
| S3 | Moyenne : fonction secondaire incorrecte, contournement possible | P3 | Corriger dans la version suivante |
| S4 | Mineure : défaut cosmétique ou de confort | P4 | Opportuniste |

### 8.2 Bilan

| Catégorie | Nombre | Statut |
|---|---|---|
| Corrigées avant la campagne (ANO-01 à 06) | 6 | Vérifiées |
| Majeures (S2) corrigées le 21/11 | 8 | Vérifiées |
| Accessibilité et défaut latent (ANO-36, ANO-39) | 2 | Vérifiées |
| **Ouvertes** | **23** | 15 S3, 8 S4, aucune P1 |
| Anomalies de l'outillage de test (ANO-T01 à T08) | 8 | Corrigées |

### 8.3 Les anomalies majeures et leur correction

| Anomalie | Défaut observé | Correction |
|---|---|---|
| **ANO-07** | Réserver −1 place par le formulaire HTML rendait une place au stock et créait un prix négatif | Nombre de places validé comme un entier ≥ 1, comme pour la voie JSON |
| **ANO-12** | Réservation et annulation sans protection CSRF ; annulation par simple lien | POST obligatoire et jeton propre à l'action et à l'objet |
| **ANO-13** | Validation, blocage et suppression par l'administrateur via de simples liens | Même protection ; liens remplacés par des formulaires POST |
| **ANO-20** | Un compte non vérifié obtenait une session | `UserChecker` : refus avant toute authentification |
| **ANO-21** | Un compte bloqué pouvait se connecter | Même `UserChecker` |
| **ANO-22** | Une route publique révélait l'e-mail du dernier inscrit | Réponse limitée à la session d'inscription en cours ; 403 sinon |
| **ANO-35** | L'ajout de véhicule provoquait une erreur 500 | Paramètre de la contrainte `Range` corrigé |
| **ANO-38** | Deux réservations simultanées pouvaient obtenir la dernière place | Décrément atomique en SQL, dans la même transaction que la réservation |

Pour chaque anomalie, le test qui la démontrait est sorti du groupe `anomalie-ouverte` et fait désormais partie de la régression : c'est le **test de confirmation**.

### 8.4 Anomalie significative : la surréservation (ANO-38)

**Le défaut.** Le contrôleur lisait le stock de places, vérifiait qu'il en restait assez, puis écrivait le nouveau stock. Si deux passagers réservent la dernière place au même moment, les deux lisent « 1 place », les deux passent le contrôle, et deux réservations sont confirmées pour une seule place. Le stock final est faux (mise à jour perdue).

**La démonstration.** Le serveur de développement ne traite qu'une requête à la fois : la concurrence HTTP réelle n'y est pas reproductible. Le test IT-07 rejoue donc l'algorithme dans **deux connexions distinctes à la base**, dans un ordre imposé : A lit, B lit, A écrit, B écrit. Le test est déterministe, donc reproductible.

**La correction.** Les trois étapes sont remplacées par une seule requête SQL conditionnelle :

```sql
UPDATE trajet SET places_disponibles = places_disponibles - :n
WHERE id = :id AND places_disponibles >= :n
```

La base verrouille la ligne pendant l'opération : la seconde réservation ne trouve plus de place et échoue proprement. L'enregistrement de la réservation se fait dans la même transaction ; en cas d'échec, le stock est restauré. L'annulation rend les places par un incrément atomique.

**Ce que l'incident a appris.** Le premier commit de correction ne contenait pas le contrôleur modifié. Pourtant, le rejeu était vert : IT-07 vérifiait l'atomicité du service, pas son utilisation par le contrôleur. Un **test garde-fou** a été ajouté ; il échoue sur la version incomplète et réussit sur la version corrigée. **Un test ne prouve que ce qu'il vise réellement.**

### 8.5 Un défaut découvert en route (ANO-39)

En reconstruisant le cache de l'application, Symfony a chargé toutes les règles de validation, et une erreur est apparue : la contrainte du mot de passe de `User` utilisait un paramètre inexistant. Toute validation d'un utilisateur provoquait donc une erreur fatale. Le défaut était **latent** : la page d'inscription, seule à valider un utilisateur, était déjà en erreur pour une autre raison (ANO-25). Il a été corrigé, et un test vérifie désormais que les contraintes des six entités se chargent sans erreur. Ce test aurait aussi détecté ANO-35.

### 8.6 Anomalies encore ouvertes

| Sévérité | Anomalies | Thèmes |
|---|---|---|
| S3 (15) | ANO-08, 09, 10, 11, 14, 15, 16, 17, 19, 23, 25, 26, 29, 31, 34 | Doublon par la voie HTML, trajets passés ou non validés réservables, règles de dates et de prix, modération contournée, véhicule d'un autre conducteur, erreurs 500, messages non affichés, absence de modification du profil |
| S4 (8) | ANO-18, 24, 27, 28, 30, 32, 33, 37 | Filtre de prix absent, statistiques fausses ou codées en dur, message en anglais, fuseau horaire, contraste |

Toutes sont rejouées à chaque régression par 27 tests du groupe `anomalie-ouverte`.

### 8.7 Anomalies de l'outillage de test

Huit défauts ont aussi été trouvés dans les tests eux-mêmes, et corrigés. Deux sont particulièrement instructifs :

- **ANO-T06 :** après une modification de configuration, les tests gardaient l'ancien comportement, car le cache de test n'était pas reconstruit. Procédure : vider `var/cache/test` avant chaque exécution ;
- **ANO-T07 :** le scénario ST-10 échouait environ une fois sur deux, car la page se recharge seule après une réservation et interrompait la navigation du test. Le test attend maintenant ce rechargement ; il a réussi cinq fois de suite.

---

## 9. Couverture et suite de régression

### 9.1 Couverture de code

| Mesure | 11/11 | Version finale |
|---|---|---|
| Lignes | 34,65 % | **44,69 %** |
| Méthodes | 49,56 % | **57,61 %** |

| Classe critique | Couverture finale |
|---|---|
| `ApiReclamationController` | 98,1 % |
| `ReclamationController` | 100 % |
| `ReservationService` | 100 % |
| `Reservation` (entité) | 94,9 % |
| `Reclamation` (entité) | 93,5 % |
| `ReservationController` | 87,8 % |
| `TrajetController` | 85,7 % |

![Figure 12](../../preuves/semaine6/campagne-finale/S06_COUVERTURE.png)

**Figure 12** — Rapport de couverture PCOV du 11/11.

**Interprétation.** La couverture indique quelles lignes ont été **exécutées**, pas si leur résultat a été **vérifié**. Le 11/11, `ReservationController` était couvert à 84,5 % et contenait pourtant 5 anomalies. Une couverture élevée ne garantit pas l'absence de défauts ; c'est pourquoi la décision ne repose pas sur ce seul indicateur.

### 9.2 Suite de régression

La suite de régression compte **131 tests** (365 assertions). Elle est rejouée :

- automatiquement par GitHub Actions à chaque push et à chaque pull request ;
- manuellement par `scripts/run-regression.ps1`, qui y ajoute les anomalies ouvertes, Playwright, Newman et k6.

Elle s'enrichit à chaque correction : le test qui démontrait l'anomalie y est déplacé. Elle est passée de 87 tests le 11/11 à 131 tests dans la version finale.

---

## 10. Bilan de la campagne, limites et risques résiduels

### 10.1 Ce que la campagne a permis

- découvrir **39 anomalies applicatives** (ANO-01 à ANO-39, dont une latente) et **8 défauts d'outillage**, tous reproduits et documentés ;
- corriger les **8 anomalies majeures**, l'accessibilité et le défaut latent, chaque correction étant prouvée par un test ;
- passer d'un **refus de livraison** à une **validation conditionnelle**, sur des preuves rejouables ;
- disposer d'une **chaîne automatisée** : une seule commande rejoue toute la campagne, et l'intégration continue protège contre les régressions.

### 10.2 Limites

- **Inscription et e-mails non testés** (EX-01, EX-02, EX-30) : le transport d'e-mails est désactivé en environnement de test. Un outil comme Mailpit permettrait de les intercepter.
- **Performance** mesurée sur un serveur de développement mono-processus, avec une charge légère : c'est un test de fumée, pas un test de charge réaliste.
- **Concurrence** démontrée par simulation déterministe, et non par des requêtes HTTP réellement simultanées.
- **Accessibilité** vérifiée par un audit automatique, qui ne couvre qu'une partie des critères WCAG ; aucune conformité n'est revendiquée.
- **Explorations** menées avec un outil, sans session entièrement manuelle.
- **Écarts avec l'énoncé** : priorité, affectation et historique des réclamations n'existent pas dans WASSALNI.

### 10.3 Risques résiduels

| Risque | Situation |
|---|---|
| R4 Modération contournée | Toujours présent (ANO-16) |
| R17 Trajets passés réservables | Toujours présent (ANO-10) |
| R6 Dates et prix incohérents à la publication | Partiellement présent (ANO-14, ANO-15) |
| R12 Statistiques fausses | Toujours présent (ANO-24, 28, 33) |
| R11 E-mails | Non testé |
| R1, R5, R20, R21 | **Couverts et corrigés** |

---

## 11. Décision de validation et recommandations

### 11.1 Évolution de la décision

| Date | Révision | Décision | Raison principale |
|---|---|---|---|
| 11/11/2026 | `cc50095` | **Refus de livraison** | 8 anomalies majeures : sécurité, intégrité des réservations, parcours bloqué |
| 21/11/2026 | `72cff37` | **Validation conditionnelle** | 0 anomalie majeure ; seuil d'accessibilité non atteint |
| 21/11/2026 | `1ed214b` | **Validation conditionnelle (confirmée)** | Tous les seuils atteints ; 7 exigences non satisfaites |

### 11.2 Justification de la décision finale

Tous les seuils quantitatifs du plan sont atteints : régression à 100 %, couverture de 44,69 % (classes critiques entre 85,7 % et 100 %), aucun défaut S1 ou S2, API à 100 %, parcours critique réussi, performance et accessibilité conformes.

La **validation simple n'est pourtant pas retenue**. L'énoncé la définit par l'absence de défaut bloquant connu et la satisfaction des exigences essentielles. Or :

- **7 exigences ne sont pas satisfaites**, dont les règles de publication des trajets (EX-09 à EX-12) ;
- **15 anomalies S3 restent ouvertes**, dont 7 qui touchent l'intégrité métier ;
- une réserve subsiste sur ANO-15 : un prix non numérique est enregistré à 0 DT.

Ces défauts sont connus, documentés et rejoués par des tests. Ils ne compromettent ni la sécurité ni la cohérence du stock : ils relèvent de la **validation conditionnelle**. Des indicateurs au vert ne remplacent pas l'analyse de ce qui reste insatisfait.

### 11.3 Conditions de livraison

1. **Avant la mise en production :** corriger les anomalies qui touchent l'intégrité métier : ANO-08 (doublon), ANO-09 (annulation d'un trajet passé), ANO-10 et ANO-11 (réservation de trajets passés ou non validés), ANO-15 (prix non numérique), ANO-16 (modération contournée), ANO-17 (véhicule d'un autre conducteur) ; et supprimer les erreurs 500 connues : ANO-23, ANO-25, ANO-34.
2. **Dans le mois suivant :** corriger les autres anomalies S3 et S4, dont le contraste (ANO-37).
3. **Pour chaque correction :** sortir le test correspondant du groupe `anomalie-ouverte`, rejouer la campagne complète et garder l'intégration continue verte.

### 11.4 Recommandations

| Recommandation | Bénéfice attendu |
|---|---|
| Tester l'inscription et les e-mails avec Mailpit | Couvrir EX-01, EX-02 et EX-30, aujourd'hui non exécutées |
| Ajouter les en-têtes de sécurité HTTP (CSP, `X-Frame-Options`…) | Réduire les risques d'injection et de détournement de clic (REC-01) |
| Faire un test de charge sur un serveur de production (PHP-FPM) | Mesurer la performance réelle et la concurrence par HTTP |
| Compléter l'audit d'accessibilité manuellement (clavier, lecteur d'écran) | Viser une conformité WCAG démontrée |
| Exécuter Playwright et Newman dans l'intégration continue | Détecter aussi les régressions d'interface et d'API à chaque push |
| Corriger l'encodage des sorties du script PowerShell | Rendre lisibles les fichiers de preuves `.txt` |

### 11.5 Visa

| Décisionnaire | Décision | Révision | Date |
|---|---|---|---|
| Responsables de campagne | Validation conditionnelle (confirmée) | `1ed214b` | 21/11/2026 |
| Enseignant | À viser | — | — |

---

## Conclusion

Cette campagne montre que le test logiciel est un **processus continu de réduction des risques**, et non une liste de scénarios exécutés à la fin du développement. L'analyse des risques a orienté les tests ; les tests ont révélé des défauts réels, dont certains graves ; chaque correction a été prouvée par un test rejouable ; et la décision de livraison s'appuie sur des critères définis à l'avance, sans les confondre avec la qualité elle-même.

Trois enseignements ressortent :

1. **Un test ne prouve que ce qu'il vise** : le commit incomplet d'ANO-38 est passé inaperçu jusqu'à l'ajout d'un test garde-fou.
2. **Les tests doivent eux-mêmes être testés** : la vérification par mutation et la correction de huit défauts d'outillage l'ont montré.
3. **Les indicateurs éclairent la décision sans la remplacer** : tous les seuils sont atteints, mais la validation reste conditionnelle tant que des exigences essentielles ne sont pas satisfaites.

---

## Annexes

| Document | Contenu |
|---|---|
| [Plan de test (02)](../dossier-tests/02-strategie-calendrier.md) | Niveaux, environnements, critères d'entrée et de sortie, règle de décision |
| [Matrice de traçabilité (09)](../dossier-tests/09-matrice-tracabilite.md) | Exigences → risques → tests → résultats → preuves |
| [Registre des anomalies (07)](../dossier-tests/07-registre-anomalies.md) | Toutes les anomalies, leurs corrections et l'outillage |
| [API (11)](../dossier-tests/11-semaine8-api.md) | Collection Newman et API REST des réclamations |
| [Interface et non fonctionnel (12)](../dossier-tests/12-semaine9-interface-non-fonctionnel.md) | Playwright, k6, accessibilité, sécurité |
| [Bilan et décision (14)](../dossier-tests/14-semaine10-bilan-decision.md) | Résultats comparés, seuils, décisions successives |
| [Écarts avec l'énoncé (15)](../dossier-tests/15-ecarts-enonce.md) | Comparaison avec RéclaGest |
| [Journal de campagne (08)](../dossier-tests/08-journal-campagne.md) | Chronologie des travaux |
| [Preuves](../rapport-preuves/index-preuves.md) | Index des sorties, rapports JUnit et captures |
