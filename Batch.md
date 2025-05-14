


## Sommaire

* [Introduction aux Batchs](#introduction-aux-batchs)

* [Batchs Temporels](#batchs-temporels)

  * [Avantages des Batchs Temporels](#avantages-des-batchs-temporels)
  * [Inconvénients des Batchs Temporels](#inconvénients-des-batchs-temporels)
  * [Gestion des Batchs Temporels](#gestion-des-batchs-temporels)
  * [Exemple de Batch Temporel](#exemple-de-batch-temporel)

* [Batchs Événementiels](#batchs-événementiels)

  * [Avantages des Batchs Événementiels](#avantages-des-batchs-événementiels)
  * [Inconvénients des Batchs Événementiels](#inconvénients-des-batchs-événementiels)
  * [Gestion des Batchs Événementiels](#gestion-des-batchs-événementiels)
  * [Exemple de Batch Événementiel](#exemple-de-batch-événementiel)

* [Comparaison des Batchs Temporels et Événementiels](#comparaison-des-batchs-temporels-et-événementiels)

* [Gestion des Pannes](#gestion-des-pannes)

* [Conclusion](#conclusion)

---

## Introduction aux Batchs

Les batchs sont des traitements automatisés exécutés soit à des intervalles de temps réguliers (batchs temporels), soit en réponse à des événements spécifiques (batchs événementiels). Ils permettent de traiter des volumes importants de données tout en minimisant l'impact sur le système.

## Batchs Temporels

Les batchs temporels s'exécutent à des intervalles réguliers, par exemple toutes les 5 minutes. Ils sont utiles pour des traitements récurrents et prévisibles.

### Avantages des Batchs Temporels

* Réduction des verrous transactionnels.
* Scalabilité.
* Centralisation des traitements.
* Gestion simplifiée des erreurs.

### Inconvénients des Batchs Temporels

* Latence accrue.
* Charge serveur accrue.
* Risque de chevauchement des batchs.
* Gestion complexe des conflits de données.

### Gestion des Batchs Temporels

* Vérifier l'achèvement des batchs précédents avant de lancer un nouveau batch.
* Limiter le traitement aux données modifiées (basé sur un champ `updated_at`).
* Implémenter des mécanismes de timeout et de retry.

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

### Avantages des Batchs Événementiels

* Réactivité accrue.
* Optimisation des ressources.
* Réduction de la latence.

### Inconvénients des Batchs Événementiels

* Complexité accrue.
* Risque de surcharge en cas d'événements fréquents.

### Gestion des Batchs Événementiels

* Implémenter des triggers ou des jobs planifiés.
* Créer des files d'attente pour gérer les événements.
* Définir des règles de gestion des erreurs et des retries.

### Exemple de Batch Événementiel

Dans Oracle et SQL Server, il est possible de déclencher des batchs en réponse à des événements spécifiques, offrant une flexibilité accrue dans la gestion des traitements.

**Oracle :**
Dans Oracle, les batchs peuvent être déclenchés à l'aide de triggers, de jobs planifiés via Oracle Scheduler ou encore via Oracle AQ (Advanced Queuing).

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
Dans SQL Server, des batchs peuvent être déclenchés via des SQL Server Agent Jobs ou par l'utilisation de Service Broker pour réagir à des événements spécifiques, tels qu'une insertion, une mise à jour ou une suppression dans une table.

Exemple :

```sql
CREATE TRIGGER TriggerBatchExecution
ON dbo.Orders
AFTER INSERT, UPDATE
AS
BEGIN
   EXEC msdb.dbo.sp_start_job N'BatchJobName';
END;
```

Ces approches permettent de combiner le traitement par batch avec des événements en temps réel, garantissant ainsi une exécution plus réactive tout en préservant la logique de traitement différé.

Dans Oracle et SQL Server, il est possible de déclencher des batchs en réponse à des événements spécifiques, offrant une flexibilité accrue dans la gestion des traitements.

**Oracle :**
Dans Oracle, les batchs peuvent être déclenchés à l'aide de triggers ou de jobs planifiés via Oracle Scheduler. Par exemple, un job peut être configuré pour s'exécuter lorsqu'une table est modifiée ou lorsqu'un événement spécifique est enregistré.

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

**SQL Server :**
Dans SQL Server, des batchs peuvent être déclenchés via des SQL Server Agent Jobs ou par l'utilisation de Service Broker pour réagir à des événements spécifiques, tels qu'une insertion, une mise à jour ou une suppression dans une table.

Exemple :

```sql
CREATE TRIGGER TriggerBatchExecution
ON dbo.Orders
AFTER INSERT, UPDATE
AS
BEGIN
   EXEC msdb.dbo.sp_start_job N'BatchJobName';
END;
```

Ces approches permettent de combiner le traitement par batch avec des événements en temps réel, garantissant ainsi une exécution plus réactive tout en préservant la logique de traitement différé.


## Comparaison des Batchs Temporels et Événementiels

* Temporels : Exécution régulière, prévisible mais peut générer de la latence.
* Événementiels : Réactifs, optimisés mais plus complexes à gérer.

## Gestion des Pannes

* Timeout et retry pour les deux types de batchs.
* Journalisation des opérations pour assurer le suivi des erreurs.
* Redémarrage des batchs échoués sans compromettre la cohérence des données.

## Conclusion

Les batchs, qu'ils soient temporels ou événementiels, sont des outils puissants pour automatiser les traitements en backend. Les batchs temporels sont simples à mettre en œuvre mais peuvent générer de la latence, tandis que les batchs événementiels sont plus réactifs mais plus complexes à gérer. La clé est de choisir le type de batch en fonction des besoins spécifiques du traitement et des contraintes du système.









