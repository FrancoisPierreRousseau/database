# Sommaire

- [Gestion des Batchs : Avantages, Inconvénients et Gestion des Pannes](#gestion-des-batchs--avantages-inconvénients-et-gestion-des-pannes)
  - [Point que je souhaite mettre en avant](#point-que-je-souhaite-mettre-en-avant)
  - [1. Avantages des batchs](#1-avantages-des-batchs)
  - [2. Inconvénients des batchs](#2-inconvénients-des-batchs)
  - [3. Gestion des batchs toutes les 5 minutes](#3-gestion-des-batchs-toutes-les-5-minutes)
  - [4. Exemple de batch toutes les 5 minutes](#4-exemple-de-batch-toutes-les-5-minutes)
  - [5. Gestion des pannes dans les batchs](#5-gestion-des-pannes-dans-les-batchs)
  - [6. Déclenchement des Batchs via des Événements dans Oracle et SQL Server](#6-déclenchement-des-batchs-via-des-événements-dans-oracle-et-sql-server)
  - [7. Conclusion](#6-conclusion)

### Gestion des Batchs : Avantages, Inconvénients et Gestion des Pannes

Les batchs sont une solution puissante pour exécuter des traitements lourds de manière différée, mais leur fréquence peut impacter la performance et la cohérence des données.

### Point que je souhaite mettre en avant

Idempotence : Toutes les opérations doivent pouvoir être rejouées sans provoquer d'incohérence.

---

### 1. Avantages des batchs

* **Réduction des verrous transactionnels :** Les traitements lourds sont exécutés en dehors des transactions critiques, réduisant le risque de deadlocks.
* **Scalabilité :** Les batchs peuvent être planifiés pour être exécutés à des moments de faible charge, optimisant ainsi les ressources serveur.
* **Centralisation des traitements :** La logique métier est centralisée dans des batchs, ce qui facilite la maintenance et le débogage.
* **Contrôle des erreurs :** Les batchs permettent de gérer les erreurs de manière structurée, avec des mécanismes de retry et de compensation.
* **Simplicité de gestion en cas de panne :** Si le serveur ou la base de données est indisponible, le batch peut simplement être replanifié ou relancé sans gestion complexe d'événements.

---

### 2. Inconvénients des batchs

* **Latence :** Les batchs sont exécutés à des intervalles réguliers (ex : toutes les 5 minutes), ce qui peut entraîner des décalages de mise à jour des données.
* **Charge serveur accrue :** Si le batch est lourd, une exécution fréquente peut saturer le serveur.
* **Risque de chevauchement :** Si un batch prend plus de temps à s'exécuter que l'intervalle prévu, le batch suivant pourrait démarrer alors que le précédent n'est pas terminé, créant des conflits ou des incohérences.
* **Gestion des conflits :** Si les données sont modifiées pendant le recalcul, il peut y avoir des écarts temporaires entre l'état réel et les données calculées.

---

### 3. Gestion des batchs toutes les 5 minutes

* **Vérification de l'achèvement du batch précédent :** Avant de lancer un nouveau batch, vérifier si le précédent est terminé avec succès pour éviter les chevauchements.
* **Limitation du traitement :** Ne traiter que les données modifiées depuis le dernier batch, basées sur un champ `updated_at`.
* **Timeout et retry :** Si le batch ne termine pas en 4 minutes, il est stoppé et réessayé plus tard pour éviter l'accumulation des batchs en échec.
* **Journalisation des opérations :** Utiliser une table de log pour suivre l'état de chaque batch (`en cours`, `terminé`, `échoué`).

---

### 4. Exemple de batch toutes les 5 minutes

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

### 5. Gestion des pannes dans les batchs

L'un des principaux avantages des batchs est leur simplicité de gestion en cas de panne. Contrairement aux opérations asynchrones ou temps réel qui nécessitent des mécanismes complexes de reprise ou de compensation, les batchs sont conçus pour fonctionner uniquement lorsque la base de données est accessible.

#### **Pourquoi ?**

* Les batchs ne répondent pas en temps réel. Si la base de données ou le serveur est indisponible, le batch échoue simplement et peut être reprogrammé ou réessayé lors du prochain cycle.
* Le traitement est global : un batch ne traite pas des opérations individuelles mais une série d'opérations. Ainsi, s'il échoue, il est relancé intégralement, garantissant la cohérence des données.
* Aucune gestion de file d'attente n'est nécessaire : contrairement aux opérations asynchrones qui doivent stocker les événements en attente, le batch peut tout simplement réessayer les opérations manquées.

#### **Gestion des erreurs :**

* **Timeouts :** Si le batch ne termine pas dans le délai imparti, il est marqué comme échoué et peut être réessayé.
* **Retry automatique :** Les batchs peuvent être configurés pour être relancés automatiquement en cas d'échec.
* **Journalisation :** Tous les batchs échoués doivent être enregistrés dans une table de log pour permettre un suivi des erreurs et une reprise ciblée.


### 6. Déclenchement des Batchs via des Événements dans Oracle et SQL Server

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


### 7. Conclusion

Les batchs sont une solution robuste pour traiter des opérations lourdes ou non critiques à intervalles réguliers. Lorsqu'ils sont exécutés toutes les 5 minutes, il est essentiel de s'assurer que le traitement est optimisé pour éviter les chevauchements, minimiser la charge serveur et garantir la cohérence des données.

La gestion des pannes est simplifiée : en cas d'échec, le batch est simplement relancé ou reprogrammé. Cela en fait une solution plus simple que l'asynchrone pur, tout en maintenant une certaine proximité avec le temps réel.































