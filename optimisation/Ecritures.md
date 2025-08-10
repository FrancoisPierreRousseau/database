# Sommaire

- [Optimisation des Opérations CRUD : INSERT, UPDATE, DELETE](#optimisation-des-opérations-crud--insert-update-delete)
  - [Batch Inserts vs. Multi-Valued Inserts / INSERT ALL](#batch-inserts-vs-multi-valued-inserts--insert-all)
  - [Quand choisir l'un ou l'autre ?](#quand-choisir-lun-ou-lautre)
  - [Multi-Valued INSERT avec Insertion par Lot](#multi-valued-insert-avec-insertion-par-lot)
  - [Gestion des Update/Delete Massifs](#gestion-des-updatedelete-massifs)
  - [MERGE / UPSERT](#merge--upsert)
  - [Triggers](#triggers)

## Optimisation des Opérations CRUD : INSERT, UPDATE, DELETE

Cette documentation vise à fournir des lignes directrices pour optimiser les opérations de création, mise à jour et suppression (INSERT, UPDATE, DELETE) dans les bases de données SQLServer et Oracle dans le cadre d'une application web à volumétrie moyenne à grande échelle.

### Batch Inserts vs. Multi-Valued Inserts / INSERT ALL

#### 1. Batch Inserts

* **Qu'est-ce que c'est ?**

  * Les batch inserts consistent à envoyer plusieurs commandes `INSERT` distinctes mais regroupées dans une même transaction ou dans une même requête.
  * Par exemple, sur SQL Server, on peut utiliser `INSERT INTO Table VALUES (...)`, `INSERT INTO Table VALUES (...)`, etc., puis faire un `COMMIT` unique.

* **Avantages :**

  * Permet de traiter chaque ligne indépendamment (chaque ligne peut réussir ou échouer indépendamment des autres).
  * Réduit le nombre de transactions ouvertes si on fait un seul `COMMIT` pour tout le batch.
  * Très efficace pour **les ETL ou les processus asynchrones**.

* **Inconvénients :**

  * Plus de code à écrire et à maintenir.
  * Peut nécessiter une gestion fine des erreurs (si une ligne échoue, faut-il annuler tout le lot ou continuer ?).
  * **Sur Oracle PL/SQL**, l'utilisation de `FORALL` est préférable à des batchs d'inserts individuels, car `FORALL` minimise le nombre d'aller-retours entre le moteur SQL et PL/SQL.

---

#### 2. Multi-Valued INSERT (SQL Server) / INSERT ALL (Oracle)

* **Qu'est-ce que c'est ?**

  * Les `INSERT ALL` (Oracle) et `INSERT INTO ... VALUES (...)` multi-valeurs (SQL Server) permettent d'insérer plusieurs lignes **en une seule commande SQL**.

  **Exemple Oracle :**

  ```sql
  INSERT ALL 
    INTO employees VALUES (1, 'John', 'Doe')
    INTO employees VALUES (2, 'Jane', 'Smith')
  SELECT 1 FROM DUAL;
  ```

  **Exemple SQL Server :**

  ```sql
  INSERT INTO employees (id, first_name, last_name) 
  VALUES 
    (1, 'John', 'Doe'),
    (2, 'Jane', 'Smith');
  ```

* **Avantages :**

  * Réduit le nombre de requêtes envoyées au serveur.
  * Traite toutes les lignes dans une transaction unique.
  * Moins de verrous pris et moins d'overhead réseau.

* **Inconvénients :**

  * Si une ligne échoue, la commande entière échoue (pas de gestion individuelle des erreurs).
  * Les `INSERT ALL` ne permettent pas de traiter chaque ligne indépendamment, ce qui peut poser problème en cas de conflit unique (ex : contrainte d'unicité sur une colonne).

---

### Quand choisir l'un ou l'autre ?

| **Scénario**                                             | **Batch Inserts**                                           | **INSERT ALL / Multi-Valued INSERT**          |
| -------------------------------------------------------- | ----------------------------------------------------------- | --------------------------------------------- |
| **Import massif de données brutes**                      | ✅                                                           | ✅ (si aucune contrainte d'unicité à vérifier) |
| **Transaction critique, gestion d'erreurs individuelle** | ✅ (chaque ligne peut être traitée indépendamment)           | ❌ (tout échoue si une ligne échoue)           |
| **Inserts rapides, peu de lignes**                       | ❌ (overhead réseau)                                         | ✅                                             |
| **Haute volumétrie, opérations asynchrones**             | ✅ (réduction des allers-retours)                            | ✅                                             |
| **PL/SQL avec beaucoup de lignes**                       | `FORALL` (beaucoup plus rapide que des inserts individuels) | ❌                                             |

En résumé :

* **Pour des inserts rapides en une seule opération**, le `INSERT ALL` (Oracle) ou `INSERT multi-valeurs` (SQL Server) est plus simple.
* **Pour des processus massifs ou des ETL**, les `Batch Inserts` sont plus adaptés car ils permettent un contrôle granulaire des erreurs et une meilleure gestion des transactions.
* **En Oracle PL/SQL**, `FORALL` est un compromis idéal pour les batchs de données, combinant rapidité et gestion d'erreurs.


### Multi-Valued INSERT (SQL Server) / INSERT ALL (Oracle) avec Insertion par Lot

---

#### 1. **Qu'est-ce que c'est ?**

L’insertion multi-valeurs (`INSERT ALL` en Oracle, `INSERT ... VALUES` en SQL Server) permet d’insérer plusieurs enregistrements en une seule commande. Cependant, si le volume de données est très élevé (ex : des milliers de lignes), il est préférable de **diviser les inserts en lots** (par exemple, 1000 lignes par lot) pour optimiser l’utilisation des ressources et réduire le risque de verrous prolongés ou d’escalade de verrous.

---

#### 2. **Pourquoi insérer par lot ?**

* **Gestion de la mémoire :** Les SGBD chargent la totalité des données à insérer dans le buffer avant d’exécuter la commande. Un trop grand volume peut saturer le buffer.
* **Journalisation :** Chaque lot est journalisé indépendamment, réduisant ainsi la taille des logs de transactions.
* **Gestion des verrous :** Les verrous sont libérés à chaque `COMMIT`, minimisant ainsi le risque de contention.
* **Gestion des erreurs :** Si un lot échoue, seuls les enregistrements de ce lot seront affectés, facilitant ainsi la reprise.

---

#### 3. **Exemple : SQL Server - Multi-Valued INSERT par Lot de 1000**

```sql
DECLARE @BatchSize INT = 1000;
DECLARE @Counter INT = 0;
DECLARE @TotalRecords INT = (SELECT COUNT(*) FROM TempData);

WHILE @Counter < @TotalRecords
BEGIN
    INSERT INTO TargetTable (col1, col2, col3)
    SELECT TOP (@BatchSize) col1, col2, col3 
    FROM TempData 
    WHERE Id > @Counter AND Id <= @Counter + @BatchSize;

    SET @Counter = @Counter + @BatchSize;

    -- COMMIT à chaque lot
    COMMIT TRANSACTION;
END;
```

* **Explication :**

  * Les données sont insérées par lots de 1000 lignes.
  * Le `COMMIT` est effectué après chaque lot, libérant les verrous progressivement.

---

#### 4. **Exemple : Oracle - INSERT ALL par Lot de 1000**

En PL/SQL, il est préférable d’utiliser `FORALL` pour minimiser le nombre d’appels entre PL/SQL et SQL.

```sql
DECLARE
    CURSOR c_data IS 
        SELECT col1, col2, col3 FROM TempData;
    TYPE data_array IS TABLE OF TempData%ROWTYPE INDEX BY PLS_INTEGER;
    l_data data_array;
    l_counter PLS_INTEGER := 0;
    l_batch_size PLS_INTEGER := 1000;
BEGIN
    OPEN c_data;
    LOOP
        FETCH c_data BULK COLLECT INTO l_data LIMIT l_batch_size;
        EXIT WHEN l_data.COUNT = 0;

        FORALL i IN 1 .. l_data.COUNT
            INSERT INTO TargetTable (col1, col2, col3)
            VALUES (l_data(i).col1, l_data(i).col2, l_data(i).col3);

        -- COMMIT à chaque lot
        COMMIT;

        l_counter := l_counter + l_data.COUNT;
        DBMS_OUTPUT.PUT_LINE('Inserted ' || l_counter || ' records.');
    END LOOP;
    CLOSE c_data;
END;
/
```

* **Explication :**

  * `FORALL` minimise les aller-retours SQL/PLSQL.
  * Les lots de 1000 lignes sont insérés et committés séparément.

---

#### ✅ **Quand utiliser l’insertion par lot ?**

| **Scénario**                    | **Multi-Valued INSERT**                   | **INSERT par Lot (Batch)**             |
| ------------------------------- | ----------------------------------------- | -------------------------------------- |
| **Moins de 1000 lignes**        | ✅ (simple, efficace)                      | ❌ (inutilement complexe)               |
| **Entre 1000 et 10 000 lignes** | ✅ (si peu de contraintes)                 | ✅ (pour gestion fine des erreurs)      |
| **Plus de 10 000 lignes**       | ❌ (risque de saturer le buffer ou le log) | ✅ (meilleure gestion mémoire et logs)  |
| **Chargement massif (ETL)**     | ❌ (pas adapté aux très gros volumes)      | ✅ (contrôle par lot, logs plus légers) |

---

#### ✅ **Conclusion :**

* **Pour des inserts de moins de 1000 lignes**, le multi-valued insert est simple et efficace.
* **Pour des volumes supérieurs à 10 000 lignes**, le traitement par lots devient indispensable pour éviter les verrous prolongés, réduire la taille des logs et mieux gérer les erreurs.
* Sur **Oracle**, `FORALL` est particulièrement efficace pour les lots, car il minimise le nombre d’aller-retours SQL/PLSQL.
* Sur **SQL Server**, la combinaison de `TOP()`, des transactions et des commits contrôlés permet de gérer finement les ressources.


### Gestion des Update/Delete Massifs : Approche par Lot (Batch Processing) vs. Opération Unique

---

#### 1. **Opération Unique : UPDATE/DELETE en une seule requête**

* **Qu'est-ce que c'est ?**

  * Effectuer un `UPDATE` ou `DELETE` sur un grand nombre de lignes en une seule commande SQL.
  * Par exemple, supprimer toutes les commandes d’un client inactif depuis 5 ans :

  **Exemple SQL Server / Oracle :**

  ```sql
  DELETE FROM orders 
  WHERE customer_id = 123 AND order_date < SYSDATE - 1825;
  ```

* **Avantages :**

  * Simple à écrire et à exécuter.
  * Une seule transaction à gérer.
  * Moins de code, moins de logique applicative.

* **Inconvénients :**

  * **Impacte les verrous :** Plus le nombre de lignes est grand, plus le verrou est long, risquant des blocages pour d'autres transactions.
  * **Risques d'escalade de verrous (SQL Server) :** Si trop de lignes sont modifiées, le moteur peut passer d'un verrou de ligne (`ROWLOCK`) à un verrou de page ou de table (`TABLOCK`), ce qui peut bloquer toutes les autres opérations sur la table.
  * **Journalisation excessive :** Un `DELETE` ou `UPDATE` massif peut générer une quantité importante de logs de transactions, affectant les performances globales.

---

#### 2. **Approche par Lot (Batch Processing)**

* **Qu'est-ce que c'est ?**

  * Découper un `UPDATE` ou `DELETE` massif en plusieurs lots plus petits (ex : 1000 lignes à la fois) avec des commits intermédiaires.

  **Exemple Oracle PL/SQL :**

  ```sql
  DECLARE
    CURSOR c1 IS 
      SELECT order_id FROM orders WHERE order_date < SYSDATE - 1825;
    TYPE order_array IS TABLE OF orders.order_id%TYPE INDEX BY PLS_INTEGER;
    l_orders order_array;
    l_counter PLS_INTEGER := 0;
  BEGIN
    OPEN c1;
    LOOP
      FETCH c1 BULK COLLECT INTO l_orders LIMIT 1000;
      EXIT WHEN l_orders.COUNT = 0;

      FORALL i IN 1..l_orders.COUNT
        DELETE FROM orders WHERE order_id = l_orders(i);
      
      COMMIT;
      l_counter := l_counter + l_orders.COUNT;
      DBMS_OUTPUT.PUT_LINE('Deleted ' || l_counter || ' orders');
    END LOOP;
    CLOSE c1;
  END;
  /
  ```

  **Exemple SQL Server :**

  ```sql
  WHILE (1=1)
  BEGIN
      DELETE TOP (1000) FROM orders
      WHERE order_date < DATEADD(YEAR, -5, GETDATE());

      IF @@ROWCOUNT = 0 BREAK;

      -- COMMIT à chaque lot
      COMMIT TRANSACTION;
  END;
  ```

* **Avantages :**

  * **Réduction des verrous :** Chaque lot est plus petit, ce qui diminue la durée des verrous.
  * **Gestion fine des erreurs :** Si une opération échoue, seule une petite portion est concernée.
  * **Contrôle de la journalisation :** Chaque lot peut être committé indépendamment, ce qui réduit la taille des logs de transactions.
  * **Flexibilité :** La taille des lots peut être ajustée en fonction de la charge du serveur.

* **Inconvénients :**

  * Complexité accrue : Plus de code à gérer.
  * Risque d’incohérence si un lot échoue (si la logique de reprise n’est pas bien conçue).
  * Performance : Chaque `COMMIT` implique un aller-retour avec le disque.

---

#### ✅ **Comparaison : Opération Unique vs. Batch Processing**

| **Scénario**                          | **Opération Unique**                 | **Batch Processing**                               |
| ------------------------------------- | ------------------------------------ | -------------------------------------------------- |
| **Petite quantité de données**        | ✅ (simple, rapide)                   | ❌ (inutilement complexe)                           |
| **Suppression de millions de lignes** | ❌ (risque d’escalade de verrous)     | ✅ (verrous plus courts, moins de logs)             |
| **Haute concurrence**                 | ❌ (risque de blocage long)           | ✅ (chaque lot relâche les verrous plus rapidement) |
| **Gros logs de transactions**         | ❌ (log unique volumineux)            | ✅ (lots plus petits, logs plus légers)             |
| **Gestion d’erreurs**                 | ❌ (rollback de toute la transaction) | ✅ (gestion lot par lot)                            |

---

#### ✅ **Conclusion :**

* Si la table est **peu sollicitée**, un `DELETE` ou `UPDATE` massif en une seule opération peut être acceptable.
* Si la table est **grande et très concurrentielle**, le traitement par lots est fortement recommandé pour réduire les risques de verrous prolongés et de saturation des logs.
* Dans le cas de bases Oracle, l’utilisation de PL/SQL avec `BULK COLLECT` et `FORALL` permet de traiter les lots de manière optimisée, en réduisant les appels entre PL/SQL et le moteur SQL.
* Sur SQL Server, le `DELETE TOP()` ou `UPDATE TOP()` est une approche simple mais efficace pour gérer des suppressions ou mises à jour en lots.

### MERGE / UPSERT

Le **MERGE** peut poser des problèmes en utilisation temps réel pour plusieurs raisons :

1. **Verrous prolongés :**

   * Le MERGE est une opération combinée qui effectue des INSERT, UPDATE et DELETE en une seule transaction. Cela peut entraîner des verrous plus longs car le moteur doit maintenir le verrou jusqu’à ce que toutes les opérations soient terminées. Si la table est très sollicitée, cela peut créer des **goulots d'étranglement**.

2. **Impact sur la concurrence :**

   * En environnement concurrentiel (beaucoup d’utilisateurs effectuant des modifications), le MERGE peut **bloquer d’autres transactions** plus simples (INSERT/UPDATE seuls) car il nécessite généralement des verrous plus étendus.

3. **Risques de Deadlock :**

   * Si plusieurs sessions exécutent des MERGE sur la même table ou des tables liées, il est possible de **provoquer des interblocages (deadlocks)** plus facilement qu'avec des opérations distinctes. C'est particulièrement vrai sur SQL Server, où l'optimiseur peut parfois choisir des stratégies de locking moins prévisibles.

#### ✅ **Quand utiliser MERGE ?**

* **Opérations programmées, batchs ou ETL** : Lorsque le volume de données est important et que l’on veut synchroniser des tables ou fusionner des datasets.
* **Synchronisation de données en dehors des heures de pointe**, où le risque de contention est moindre.
* **Mise à jour d’un dataset complet** (par exemple, répercuter les changements d’une table source vers une table cible).

#### 🚫 **Quand éviter MERGE ?**

* **Transactions critiques en temps réel**, surtout en cas de forte concurrence.
* **Petites opérations unitaires** où un simple INSERT ou UPDATE serait plus rapide et plus léger.
* **Applications sensibles aux délais**, car le MERGE peut entraîner des attentes sur des verrous.

En résumé, le MERGE est une excellente commande pour des opérations complexes en lot ou des synchronisations planifiées, mais pour des traitements transactionnels en temps réel, il peut effectivement poser problème. Une alternative est d'**isoler les opérations INSERT/UPDATE en transactions distinctes**, avec des index appropriés et une gestion fine des verrous.


### Triggers

**Impact :** Les triggers s'exécutent pour chaque opération de modification, ce qui peut ralentir considérablement les inserts/updates massifs.

**Recommandation :**

* Limiter les triggers à des besoins critiques.
* Déplacer les traitements lourds dans des batchs ou des procédures asynchrones.

**Cas d'utilisation :**

* Suivi des modifications (audit log).
* Gestion d'alertes en temps réel.















