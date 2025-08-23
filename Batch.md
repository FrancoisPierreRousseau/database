## Sommaire

- [Introduction aux Batchs](#introduction-aux-batchs)

- [Batchs Temporels](#batchs-temporels)

  - [Avantages des Batchs Temporels](#avantages-des-batchs-temporels)
  - [Inconvénients des Batchs Temporels](#inconvénients-des-batchs-temporels)
  - [Gestion des Batchs Temporels](#gestion-des-batchs-temporels)
  - [Exemple de Batch Temporel](#exemple-de-batch-temporel)

- [Batchs Événementiels](#batchs-événementiels)

  - [Avantages des Batchs Événementiels](#avantages-des-batchs-événementiels)
  - [Inconvénients des Batchs Événementiels](#inconvénients-des-batchs-événementiels)
  - [Gestion des Batchs Événementiels](#gestion-des-batchs-événementiels)
  - [Bonnes pratiques pour l’asynchronicité et les ressources partagées](#bonnes-pratiques-pour-lasynchronicité-et-les-ressources-partagées)
  - [Exemple de Batch Événementiel](#exemple-de-batch-événementiel)

- [Comparaison des Batchs Temporels et Événementiels](#comparaison-des-batchs-temporels-et-événementiels)
- [Gestion des Pannes](#gestion-des-pannes)
- [Stratégie Recommandée : Stockage Intermédiaire des Modifications](#strategie-recommandee-stockage-intermediaire-des-modifications)
- [Comparaison entre Batchs Événementiels et Triggers](#comparaison-entre-batchs-événementiels-et-triggers)
- [Gestion des Transactions Atomiques pour Certains Champs](#gestion-des-transactions-atomiques-pour-certains-champs)
- [Conclusion](#conclusion)

---

## Introduction aux Batchs

Les batchs sont des traitements automatisés exécutés soit à des intervalles de temps réguliers (**batchs temporels**), soit en réponse à des événements spécifiques (**batchs événementiels**). Ils permettent de traiter des volumes importants de données tout en minimisant l'impact sur le système.

**Remarque importante :** l’ordre des transactions est critique pour certains traitements métier. Les batchs temporels **ne garantissent pas l’ordre de traitement** si plusieurs batchs sont lancés pour des traitements différents. Pour assurer la cohérence, les queues ou files d’attente restent la meilleure approche.

## Batchs Temporels

Les batchs temporels s'exécutent à des intervalles réguliers, par exemple toutes les 5 minutes. Ils sont utiles pour des traitements récurrents et prévisibles.

### Avantages des Batchs Temporels

- Réduction des verrous transactionnels.
- Scalabilité.
- Centralisation des traitements.
- Gestion simplifiée des erreurs.

### Inconvénients des Batchs Temporels

- Latence accrue.
- Charge serveur accrue.
- Risque de chevauchement des batchs.
- Gestion complexe des conflits de données.
- **Ne garantit pas l’ordre des transactions** si plusieurs batchs métier sont exécutés simultanément.

### Gestion des Batchs Temporels

- Vérifier l'achèvement des batchs précédents avant de lancer un nouveau batch.
- Limiter le traitement aux données modifiées (basé sur un champ `updated_at`).
- Implémenter des mécanismes de timeout et de retry.
- Pour plusieurs traitements métier critiques, **préférer une approche événementielle avec queue** plutôt que des batchs temporels simultanés.

### Exemple de Batch Temporel

**Table de log :**

```sql
CREATE TABLE batch_log (
    batch_id INT PRIMARY KEY,
    start_time DATETIME,
    end_time DATETIME,
    status VARCHAR(20),
    batch_type VARCHAR(50)
);
```

**Script batch :**

```sql
-- Vérifie si un batch est déjà en cours
IF EXISTS (SELECT 1 FROM batch_log WHERE status = 'en cours' AND batch_type = 'recalcul_stock')
BEGIN
    PRINT 'Un batch est déjà en cours, le batch actuel est annulé.';
    RETURN;
END

-- Démarre le batch
DECLARE @batch_id INT = (SELECT ISNULL(MAX(batch_id), 0) + 1 FROM batch_log);
INSERT INTO batch_log (batch_id, start_time, status, batch_type)
VALUES (@batch_id, GETDATE(), 'en cours', 'recalcul_stock');

-- Traitement des données
BEGIN TRY
    -- Recalcule uniquement les données modifiées
    UPDATE inventory
    SET stock_quantity = stock_quantity - o.quantity
    FROM orders o
    WHERE o.updated_at >= (SELECT MAX(end_time) FROM batch_log WHERE batch_type = 'recalcul_stock' AND status = 'terminé');

    -- Fin du batch
    UPDATE batch_log SET end_time = GETDATE(), status = 'terminé' WHERE batch_id = @batch_id;
END TRY
BEGIN CATCH
    -- En cas d'erreur
    UPDATE batch_log SET end_time = GETDATE(), status = 'échoué' WHERE batch_id = @batch_id;
    THROW;
END CATCH;
```

---

## Batchs Événementiels

Les batchs événementiels sont déclenchés par des événements tels qu'une mise à jour de données ou une insertion dans une table.

**Avantage clé :** ils permettent de garantir **l’ordre de traitement des transactions** via des queues, ce qui est essentiel pour la cohérence des traitements métier critiques.

### Avantages des Batchs Événementiels

- Réactivité accrue.
- Optimisation des ressources.
- Réduction de la latence.
- **Maintien de l’ordre des transactions** grâce aux files d’attente.

### Inconvénients des Batchs Événementiels

- Complexité accrue.
- Risque de surcharge en cas d'événements fréquents.

### Gestion des Batchs Événementiels

- Créer des files d’attente pour gérer les événements.
- Définir des règles de gestion des erreurs et des retries.
- Les queues garantissent un traitement ordonné et fiable des événements, même en cas de surcharge.

> ⚠️ L’utilisation directe de **triggers** pour lancer des jobs est déconseillée, car elle peut entraîner une surcharge du système et ne garantit pas un ordre cohérent d’exécution des transactions.

### Bonnes pratiques pour l’asynchronicité et les ressources partagées

1. **Champs calculés ou dérivés**

   - Les champs mis à jour par batch événementiel doivent être **calculés et non modifiables directement**.
   - Exemple : `stock_available_for_orders` calculé à partir de `stock_real`.

2. **Table de staging / stockage intermédiaire**

   - Écrire les modifications dans une table intermédiaire avant d’appliquer aux tables sources.

3. **Verrous ou indicateurs (`in_progress`)**

   - Marquer les ressources en cours de traitement pour éviter les conflits.
   - Préférer **verrous optimistes** ou marqueurs plutôt que verrous pessimistes lourds.

4. **Priorisation des événements**

   - Sur Oracle AQ : utiliser le champ `priority`.
   - Sur SQL Server : utiliser plusieurs queues ou trier par champ `priority` côté worker.

5. **Fusion et dé-duplication des événements**

   - Grouper les événements sur la même ressource pour limiter les conflits et optimiser le traitement.

6. **Retry et backoff**

   - Si la ressource est occupée, remettre le message en queue après un délai.

7. **Monitoring et alerting**

   - Surveiller la longueur de la queue, les verrous bloqués, le temps de traitement et les échecs.

### Exemple de Batch Événementiel

Dans Oracle et SQL Server, il est possible de déclencher des batchs en réponse à des événements spécifiques, offrant une flexibilité accrue dans la gestion des traitements.

**Oracle :**
Dans Oracle, les batchs peuvent être déclenchés à l'aide de jobs planifiés via Oracle Scheduler ou encore via Oracle AQ (Advanced Queuing).

1. **Déclenchement par Oracle Scheduler :**
   Un job peut être configuré pour s'exécuter lorsqu'une table est modifiée ou lorsqu'un événement spécifique est enregistré.

Exemple :

```sql
BEGIN
   DBMS_SCHEDULER.create_job(
      job_name        => 'batch_triggered_job',
      job_type        => 'PLSQL_BLOCK',
      job_action      => 'BEGIN batch_procedure(); END;',
      event_condition => 'table_update_event',
      auto_drop       => TRUE
   );
END;
/
```

2. **Déclenchement par Oracle AQ (Advanced Queuing) :**
   Oracle AQ permet de mettre en place une file d'attente de messages et de déclencher des jobs en fonction des messages reçus. Cela permet d'implémenter une architecture orientée événements pour le traitement des batchs.

Exemple :

```sql
-- Créer une file d'attente
BEGIN
   DBMS_AQADM.create_queue_table(
      queue_table => 'batch_queue_table',
      queue_payload_type => 'RAW'
   );

   DBMS_AQADM.create_queue(
      queue_name => 'batch_queue',
      queue_table => 'batch_queue_table'
   );

   DBMS_AQADM.start_queue(
      queue_name => 'batch_queue'
   );
END;
/

-- Créer un job pour traiter les messages de la file d'attente
BEGIN
   DBMS_SCHEDULER.create_job(
      job_name   => 'batch_queue_job',
      job_type   => 'PLSQL_BLOCK',
      job_action => 'BEGIN batch_procedure(); END;',
      queue_spec => 'batch_queue',
      auto_drop  => TRUE
   );
END;
/

-- Insérer un message dans la file d'attente
DECLARE
   message RAW(2000);
BEGIN
   message := UTL_RAW.cast_to_raw('Trigger Batch');
   DBMS_AQ.enqueue(
      queue_name => 'batch_queue',
      enqueue_options => DBMS_AQ.ENQUEUE_OPTIONS_T(),
      message_properties => DBMS_AQ.MESSAGE_PROPERTIES_T(),
      payload => message,
      msgid => message
   );
END;
/
```

Cette approche permet de découpler le déclenchement des batchs des événements eux-mêmes, offrant ainsi une architecture plus modulaire et plus facile à maintenir.

**SQL Server :**
Dans SQL Server, il est recommandé de s’appuyer sur une **table de staging ou Service Broker** plutôt que de déclencher directement un job via un trigger.

Exemple avec Service Broker :

```sql
-- Création d'une file de messages
CREATE QUEUE OrderUpdateQueue;

-- Création d'un service lié à la file
CREATE SERVICE OrderUpdateService
ON QUEUE OrderUpdateQueue ([DEFAULT]);

-- Lorsqu’un événement se produit, insérer un message dans la file
SEND ON CONVERSATION @dialog_handle
MESSAGE TYPE [DEFAULT]
(N'ORDER_UPDATE: {"order_id": 123, "item_id": 1, "quantity": 5}');
```

Un batch peut ensuite lire les messages de la file, les traiter en lot et garantir un ordre et une cohérence.

Cette approche est plus robuste que les triggers directs, car elle offre une meilleure scalabilité, un contrôle sur l’ordre de traitement et une résilience accrue en cas de surcharge.

## Comparaison des Batchs Temporels et Événementiels

| Critère                          | Batch Temporel              | Batch Événementiel avec Queue |
| -------------------------------- | --------------------------- | ----------------------------- |
| Exécution                        | Régulière, prévisible       | Sur événement                 |
| Latence                          | Peut être élevée            | Faible                        |
| Complexité                       | Simple                      | Plus complexe                 |
| Scalabilité                      | Moyenne si batchs multiples | Excellente                    |
| Ordre des transactions           | **Non garanti**             | **Garanti grâce à la queue**  |
| Adapté aux traitements critiques | ❌                          | ✅                            |

## Gestion des Pannes

- Timeout et retry pour les deux types de batchs.
- Journalisation des opérations pour assurer le suivi des erreurs.
- Redémarrage des batchs échoués sans compromettre la cohérence des données.

## Stratégie Recommandée : Stockage Intermédiaire des Modifications

L'application des nouvelles modifications doit suivre une stratégie qui vise à minimiser les risques d'incohérence des données et à garantir l'intégrité des traitements batchs.

### Objectif :

- Éviter toute modification directe des tables sources.
- Conserver les nouvelles valeurs dans une structure temporaire ou une table de staging.
- Appliquer ces modifications de manière transactionnelle une fois que le batch a été entièrement validé.

### Pourquoi cette stratégie ?

- **Réversibilité :** Si le batch échoue, les modifications ne sont pas appliquées aux tables sources. Cela permet un retour arrière rapide sans affecter les données en production.
- **Atomicité :** Les modifications ne sont appliquées qu'une fois l'ensemble du batch validé. En cas d'échec, aucune donnée partielle ou incomplète n'est insérée.
- **Gestion des conflits :** Les conflits de données peuvent être identifiés et résolus dans la table de staging avant d'être appliqués aux tables sources.
- **Performance :** Réduire le nombre d'opérations de lecture/écriture sur les tables sources permet de limiter les verrous transactionnels.

### Implémentation Technique :

- Créer une **table de staging** pour stocker les modifications.
- Mettre en place un **batch de traitement** qui :

  - Récupère les données modifiées.
  - Les insère ou met à jour dans la table de staging.
  - Effectue les contrôles nécessaires (validation des données, gestion des conflits).
  - Applique les modifications aux tables sources **uniquement si le batch est terminé avec succès**.
  - Archive ou supprime les données de la table de staging une fois le traitement terminé.

### Exemple d'Implémentation :

#### Table de Staging :

```sql
CREATE TABLE staging_inventory (
    item_id INT PRIMARY KEY,
    new_stock_quantity INT,
    modified_at DATETIME,
    batch_id INT
);
```

#### Batch de Traitement :

```sql
DECLARE @batch_id INT = (SELECT ISNULL(MAX(batch_id), 0) + 1 FROM batch_log);

-- Insérer les modifications dans la table de staging
INSERT INTO staging_inventory (item_id, new_stock_quantity, modified_at, batch_id)
SELECT item_id, stock_quantity - o.quantity, GETDATE(), @batch_id
FROM orders o
WHERE o.updated_at >= (SELECT MAX(end_time) FROM batch_log WHERE batch_type = 'recalcul_stock' AND status = 'terminé');

-- Validation des données
IF EXISTS (
    SELECT 1
    FROM staging_inventory
    WHERE new_stock_quantity < 0
)
BEGIN
    PRINT 'Erreur : Quantité de stock négative détectée. Batch annulé.';
    RETURN;
END

-- Appliquer les modifications aux tables sources
BEGIN TRY
    UPDATE inventory
    SET stock_quantity = s.new_stock_quantity
    FROM staging_inventory s
    WHERE inventory.item_id = s.item_id AND s.batch_id = @batch_id;

    -- Marquer le batch comme terminé
    INSERT INTO batch_log (batch_id, start_time, end_time, status, batch_type)
    VALUES (@batch_id, GETDATE(), GETDATE(), 'terminé', 'recalcul_stock');

    -- Nettoyage de la table de staging
    DELETE FROM staging_inventory WHERE batch_id = @batch_id;
END TRY
BEGIN CATCH
    PRINT 'Erreur lors de l'application des modifications. Batch annulé.';
END CATCH;
```

### Recommandations :

- **Journalisation :** Conserver un log des opérations dans une table dédiée pour le suivi des traitements.
- **Notifications :** Mettre en place des alertes en cas d’échec pour notifier les équipes responsables.
- **Optimisation :** Prévoir des index sur la table de staging pour accélérer les jointures et les validations.
- **Tests :** Effectuer des tests complets sur des environnements non productifs avant déploiement.

## Comparaison entre Batchs Événementiels et Triggers

Les triggers et les batchs événementiels sont deux mécanismes permettant d'exécuter des traitements suite à des événements, mais ils diffèrent par leur fonctionnement et leurs implications.

### Pourquoi les Batchs Événementiels sont Préférables aux Triggers ?

1. **Scalabilité :**

- Les triggers s'exécutent immédiatement et individuellement, ce qui peut provoquer des blocages et des verrous en cas de forte volumétrie.
- Les batchs événementiels regroupent plusieurs événements et les traitent en lot, minimisant ainsi les conflits de verrouillage et optimisant les performances.

2. **Gestion des Erreurs :**

- Les triggers font partie de la transaction principale. Si une erreur survient, toute la transaction peut échouer.
- Les batchs événementiels stockent les modifications de manière temporaire et ne les appliquent qu'une fois validées, permettant des rollbacks plus fins et plus contrôlés.

3. **Réversibilité :**

- Les triggers appliquent immédiatement les modifications, ce qui complique les retours en arrière.
- Les batchs événementiels permettent de vérifier les modifications dans une table de staging avant de les appliquer, rendant le rollback plus facile.

4. **Observabilité et Monitoring :**

- Les triggers sont intégrés dans la logique transactionnelle de la base de données, rendant leur suivi plus complexe.
- Les batchs événementiels peuvent être journalisés et monitorés indépendamment, offrant une meilleure traçabilité des opérations.

### Exemples :

- Avec Triggers :

```sql
CREATE TRIGGER UpdateStock
AFTER INSERT ON Orders
FOR EACH ROW
BEGIN
   UPDATE Inventory SET stock_quantity = stock_quantity - NEW.quantity WHERE item_id = NEW.item_id;
END;
```

- Avec Batch Événementiel :

1. Insertion dans la table de staging :

```sql
INSERT INTO staging_events (event_type, payload) VALUES ('ORDER_UPDATE', '{"item_id": 1, "quantity": 5}');
```

2. Batch de traitement :

```sql
DECLARE @batch_id INT = (SELECT MAX(batch_id) FROM staging_events WHERE processed = 0);

BEGIN TRANSACTION;

-- Appliquer les modifications
UPDATE Inventory
SET stock_quantity = stock_quantity - e.payload.value('quantity', 'INT')
FROM staging_events e
WHERE e.processed = 0;

-- Marquer les événements comme traités
UPDATE staging_events SET processed = 1 WHERE batch_id = @batch_id;

COMMIT;
```

## Gestion des Transactions Atomiques pour Certains Champs

Dans certains cas, il est pertinent de permettre des modifications indépendantes pour certains champs tout en appliquant une transaction atomique sur d'autres champs critiques.

### Pourquoi cette Approche ?

- **Optimisation des Performances :** Les transactions atomiques peuvent être limitées aux champs critiques pour minimiser les verrous.
- **Flexibilité :** Les champs non critiques peuvent être mis à jour indépendamment sans affecter la logique transactionnelle des champs critiques.

### Exemple :

Table :

```sql
CREATE TABLE Orders (
    order_id INT PRIMARY KEY,
    status VARCHAR(20),
    quantity INT,
    total_amount DECIMAL(10, 2),
    last_updated DATETIME
);
```

Mise à jour indépendante :

```sql
UPDATE Orders SET status = 'Shipped', last_updated = GETDATE() WHERE order_id = 1;
```

Transaction atomique :

```sql
BEGIN TRANSACTION;

BEGIN TRY
    DECLARE @quantity INT = 10;
    DECLARE @unit_price DECIMAL(10, 2) = 15.00;

    UPDATE Orders SET quantity = @quantity, total_amount = @quantity * @unit_price WHERE order_id = 1;

    COMMIT;
END TRY
BEGIN CATCH
    ROLLBACK;
    PRINT 'Erreur lors de la mise à jour atomique.';
END CATCH;
```

Cette approche permet de combiner la flexibilité des mises à jour indépendantes avec la robustesse des transactions atomiques pour les champs critiques.

## Conclusion

Les batchs, qu'ils soient temporels ou événementiels, sont des outils puissants pour automatiser les traitements en backend. Les batchs temporels sont simples à mettre en œuvre mais peuvent générer de la latence, tandis que les batchs événementiels sont plus réactifs mais plus complexes à gérer. La clé est de choisir le type de batch en fonction des besoins spécifiques du traitement et des contraintes du système.
