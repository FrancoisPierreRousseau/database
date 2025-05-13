### ✅ **1. Pourquoi les triggers complexes peuvent poser problème ?**

Les triggers sont exécutés automatiquement à chaque **insertion, mise à jour ou suppression** de données dans une table. Cela peut devenir problématique lorsque :

* **Le trigger devient trop complexe :**

  * Si un trigger comporte plusieurs étapes complexes (calculs, requêtes, mises à jour d’autres tables, notifications), cela alourdit considérablement chaque transaction.
  * Exemple : Un `INSERT` simple peut déclencher une cascade de calculs, mises à jour et notifications, transformant une opération censée durer 10ms en une série d’actions durant 500ms.

* **Les triggers se déclenchent de manière chaotique :**

  * Les triggers s'exécutent **indépendamment les uns des autres**. Si plusieurs transactions déclenchent le même trigger en parallèle, des conflits ou des blocages peuvent survenir.

* **Impact sur les performances :**

  * Si le trigger effectue plusieurs opérations en cascade (INSERT, UPDATE, DELETE dans d’autres tables), chaque transaction devient un goulot d’étranglement.
  * Pire, si un trigger effectue des appels externes (ex : envoi d’un email), cela peut **immobiliser la transaction** en attendant la fin de chaque opération.

---

### ✅ **2. Quelle est la solution ?**

**Déplacer la logique des triggers complexes vers des batchs planifiés.**

L’idée est de **découpler la logique métier complexe des transactions principales**, afin de ne pas pénaliser le traitement principal.

#### **Comment ?**

* **Au lieu de faire toutes les opérations immédiatement dans le trigger, on va stocker des informations dans une table d’état ou de log.**
* Ensuite, un batch planifié (ex : une procédure stockée ou un job SQL Server Agent) **va traiter ces états** de manière asynchrone, selon une fréquence déterminée (toutes les 5 minutes, toutes les heures, etc.).

---

### ✅ **3. Mise en œuvre : Gestion des Stocks dans un E-commerce**

---

**Scénario :**

* Un e-commerce a implémenté des triggers complexes pour mettre à jour le stock (`Inventory`) et les ventes (`Sales`) à chaque insertion dans la table `Orders`.
* Cependant, ces triggers sont exécutés immédiatement, ce qui provoque des verrous prolongés et des risques de deadlocks lorsque plusieurs commandes sont insérées en parallèle.
* Solution : **Reporter ces mises à jour vers un batch planifié**, en utilisant une table intermédiaire pour enregistrer les événements.

---

#### **Étape 1 : Créer une table d'événements pour les mises à jour de stock**

* Plutôt que de mettre à jour immédiatement le stock et les ventes, on enregistre chaque commande dans une table `stock_events`.

**SQL Server / Oracle :**

```sql
CREATE TABLE stock_events (
    event_id INT PRIMARY KEY,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    processed BIT DEFAULT 0,
    created_at DATETIME DEFAULT GETDATE()
);
```

---

#### **Étape 2 : Modifier le trigger pour enregistrer les événements**

* Le trigger ne met plus à jour `Inventory` et `Sales` immédiatement.
* Il insère plutôt un enregistrement dans `stock_events` pour chaque ligne insérée dans `Orders`.

**SQL Server :**

```sql
CREATE TRIGGER trg_OrderInsert
ON Orders
AFTER INSERT
AS
BEGIN
    INSERT INTO stock_events (product_id, quantity)
    SELECT product_id, quantity FROM inserted;
END;
```

**Oracle :**

```sql
CREATE OR REPLACE TRIGGER trg_OrderInsert
AFTER INSERT ON Orders
FOR EACH ROW
BEGIN
    INSERT INTO stock_events (event_id, product_id, quantity)
    VALUES (stock_seq.NEXTVAL, :NEW.product_id, :NEW.quantity);
END;
```

---

#### **Étape 3 : Créer un batch planifié pour traiter les événements de stock**

* Le batch va regrouper les événements par `product_id`, effectuer une mise à jour unique par produit et marquer les événements comme traités.

**SQL Server - Procédure stockée et Job SQL Agent :**

```sql
CREATE PROCEDURE ProcessStockEvents AS
BEGIN
    DECLARE @ProductId INT;
    DECLARE @TotalQuantity INT;

    DECLARE stock_cursor CURSOR FOR 
    SELECT product_id, SUM(quantity) 
    FROM stock_events 
    WHERE processed = 0 
    GROUP BY product_id;

    OPEN stock_cursor;
    FETCH NEXT FROM stock_cursor INTO @ProductId, @TotalQuantity;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Mettre à jour le stock
        UPDATE Inventory 
        SET stock_quantity = stock_quantity - @TotalQuantity 
        WHERE product_id = @ProductId;

        -- Mettre à jour les ventes
        UPDATE Sales 
        SET total_sales = total_sales + @TotalQuantity 
        WHERE product_id = @ProductId;

        -- Marquer les événements comme traités
        UPDATE stock_events 
        SET processed = 1 
        WHERE product_id = @ProductId;

        FETCH NEXT FROM stock_cursor INTO @ProductId, @TotalQuantity;
    END;

    CLOSE stock_cursor;
    DEALLOCATE stock_cursor;
END;
```

* **Planification du Job :** Le job est planifié toutes les 5 minutes via le **SQL Server Agent**.

---

**Oracle - Processus PL/SQL :**

```sql
BEGIN
    FOR rec IN (SELECT product_id, SUM(quantity) AS total_qty FROM stock_events WHERE processed = 0 GROUP BY product_id) LOOP
        -- Mise à jour du stock
        UPDATE Inventory 
        SET stock_quantity = stock_quantity - rec.total_qty 
        WHERE product_id = rec.product_id;

        -- Mise à jour des ventes
        UPDATE Sales 
        SET total_sales = total_sales + rec.total_qty 
        WHERE product_id = rec.product_id;

        -- Marquer les événements comme traités
        UPDATE stock_events 
        SET processed = 1 
        WHERE product_id = rec.product_id;
    END LOOP;

    COMMIT;
END;
/
```

* En Oracle, ce processus peut être planifié via le package `DBMS_SCHEDULER`.

---

### ✅ **Pourquoi cette approche est-elle meilleure ?**

| **Aspect**            | **Trigger Direct**                                               | **Batch Planifié**                                 |
| --------------------- | ---------------------------------------------------------------- | -------------------------------------------------- |
| **Performance**       | Impact immédiat, verrous plus longs.                             | Exécution différée, verrous plus courts.           |
| **Gestion d'erreurs** | En cas d'erreur, la transaction échoue.                          | Les erreurs peuvent être gérées indépendamment.    |
| **Scalabilité**       | Les triggers s’exécutent individuellement pour chaque opération. | Les logs sont regroupés et traités par lots.       |
| **Concurrence**       | Le trigger bloque les transactions.                              | Le batch est asynchrone, minimisant la contention. |
| **Maintenance**       | Difficulté à suivre les étapes (trigger spaghetti).              | Le batch est centralisé, le code est plus clair.   |

---

### ✅ **Conclusion :**

* Les triggers complexes peuvent devenir des goulots d’étranglement, surtout sous forte concurrence ou avec des opérations lourdes.
* En déplaçant la logique vers des batchs planifiés, on libère les transactions principales des traitements lourds, améliorant ainsi les performances globales.
* On stocke simplement **l’événement ou l’état dans une table de logs**, puis on traite ces logs en différé.
* Cette approche est particulièrement efficace pour des calculs d'agrégats, des notifications, des recalculs ou des processus d'intégration asynchrone.

