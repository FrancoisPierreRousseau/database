# Sommaire

- [Pourquoi les triggers complexes peuvent poser problème ?](#pourquoi-les-triggers-complexes-peuvent-poser-problème)
- [Quelle est la solution ?](#quelle-est-la-solution)
- [Mise en œuvre : Gestion des Stocks dans un E-commerce](#mise-en-œuvre--gestion-des-stocks-dans-un-e-commerce)
- [Cohérence des données : Privilégier un système événementiel avec queue](#cohérence-des-données--privilégier-un-système-événementiel-avec-queue)
- [Pourquoi cette approche est-elle meilleure ?](#pourquoi-cette-approche-est-elle-meilleure)
- [Points d’attention](#points-dattention)
- [Conclusion](#conclusion)

### Pourquoi les triggers complexes peuvent poser problème?

Les triggers sont exécutés automatiquement à chaque **insertion, mise à jour ou suppression** de données dans une table. Cela peut devenir problématique lorsque :

- **Le trigger devient trop complexe :**

  - Si un trigger comporte plusieurs étapes complexes (calculs, requêtes, mises à jour d’autres tables, notifications), cela alourdit considérablement chaque transaction.
  - Exemple : Un `INSERT` simple peut déclencher une cascade de calculs, mises à jour et notifications, transformant une opération censée durer 10ms en une série d’actions durant 500ms.

- **Les triggers se déclenchent de manière chaotique :**

  - Les triggers s'exécutent **indépendamment les uns des autres**. Si plusieurs transactions déclenchent le même trigger en parallèle, des conflits ou des blocages peuvent survenir.

- **Impact sur les performances :**

  - Si le trigger effectue plusieurs opérations en cascade (INSERT, UPDATE, DELETE dans d’autres tables), chaque transaction devient un goulot d’étranglement.
  - Pire, si un trigger effectue des appels externes (ex : envoi d’un email), cela peut **immobiliser la transaction** en attendant la fin de chaque opération.

---

### Quelle est la solution ?

**Déplacer la logique des triggers complexes vers un système événementiel basé sur des queues** plutôt que des batchs temporels.

L’idée :

- Les triggers n’exécutent **plus directement la logique métier lourde**.
- Ils insèrent un message ou un événement dans une **file d’attente (queue)**.
- Un ou plusieurs workers consomment ces événements de façon **asynchrone**, garantissant l’ordre et la résilience.

#### Avantages / Desavantages

L’un des **grands avantages** de cette méthode est qu’elle peut être **mise en place progressivement**.

- On n’a pas besoin de réécrire tout le système d’un coup. On peut commencer par identifier les triggers les plus lourds ou les plus critiques, les externaliser dans des batchs, puis itérer.
- Cependant, il est essentiel de **bien documenter chaque étape**, car cette approche **crée un découplage entre les opérations initiales (INSERT) et le traitement effectif (batchs)**.
- Si ce découplage est mal structuré ou mal documenté, cela peut rapidement **devenir un cauchemar à maintenir**, car la logique métier est disséminée entre le trigger, les tables intermédiaires et les procédures batch.

---

### Mise en œuvre - Gestion des Stocks dans un E-commerce

#### **Scénario :**

- Un site e-commerce a des triggers qui mettent à jour immédiatement `Inventory` et `Sales` lors d’un `INSERT` dans `Orders`.
- Cela entraîne des verrous longs et des risques de deadlocks.
- Solution : **déléguer la mise à jour à un worker événementiel** via une queue.

---

#### **Étape 1 : Créer une table ou une file d’événements**

SQL Server / Oracle (staging minimal) :

```sql
CREATE TABLE stock_events (
    event_id INT PRIMARY KEY,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    created_at DATETIME DEFAULT GETDATE(),
    processed BIT DEFAULT 0
);
```

Ou directement dans une **queue native** (Oracle AQ, Service Broker, RabbitMQ, Kafka).

---

#### **Étape 2 : Modifier le trigger pour pousser un événement**

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

**Oracle AQ (file d’attente) :**

```sql
DECLARE
   message RAW(2000);
BEGIN
   message := UTL_RAW.cast_to_raw('{"product_id": :NEW.product_id, "qty": :NEW.quantity}');
   DBMS_AQ.enqueue(
      queue_name          => 'stock_update_queue',
      enqueue_options     => DBMS_AQ.ENQUEUE_OPTIONS_T(),
      message_properties  => DBMS_AQ.MESSAGE_PROPERTIES_T(),
      payload             => message,
      msgid               => message
   );
END;
```

---

#### **Étape 3 : Consommer les événements via un worker**

**Exemple SQL Server (Service Broker) :**

```sql
-- Consommation des messages
WAITFOR (
   RECEIVE TOP(1)
      @msg_body = message_body
   FROM StockUpdateQueue
), TIMEOUT 1000;

-- Mise à jour de l’inventaire
UPDATE Inventory
SET stock_quantity = stock_quantity - JSON_VALUE(@msg_body, '$.qty')
WHERE product_id = JSON_VALUE(@msg_body, '$.product_id');
```

**Exemple Oracle (AQ + job scheduler) :**

```sql
BEGIN
   DBMS_SCHEDULER.create_job(
      job_name   => 'process_stock_events',
      job_type   => 'PLSQL_BLOCK',
      job_action => 'BEGIN process_stock_procedure(); END;',
      queue_spec => 'stock_update_queue',
      auto_drop  => TRUE
   );
END;
/
```

---

### Cohérence des données - Privilégier un système événementiel avec queue

Pourquoi choisir une **architecture événementielle avec queue** plutôt qu’un batch temporel ?

1. **Réactivité immédiate**
   Pas besoin d’attendre un batch toutes les 5 minutes : l’événement est traité dès qu’il arrive.

2. **Maintien de l’ordre des transactions**
   Les queues garantissent que les événements sont consommés dans le bon ordre, contrairement aux batchs.

3. **Scalabilité horizontale**
   Plusieurs workers peuvent consommer la file en parallèle, tout en respectant l’ordre par clé métier (`product_id` par ex.).

4. **Résilience**
   Si un worker échoue, le message reste dans la queue et peut être retraité (retry / backoff).

---

### Pourquoi cette approche est-elle meilleure ?

| **Aspect**            | **Trigger Direct**                   | **Batch Temporel**                 | **Événementiel avec Queue**             |
| --------------------- | ------------------------------------ | ---------------------------------- | --------------------------------------- |
| **Performance**       | Transactions ralenties               | Latence (minutes)                  | Réactif et découplé                     |
| **Ordre garanti**     | Non                                  | Non                                | ✅ Oui, via la queue                    |
| **Gestion d'erreurs** | Bloque la transaction                | Retraitement différé               | Retry, backoff, monitoring natif        |
| **Scalabilité**       | Très limitée                         | Moyenne                            | Excellente, workers parallélisables     |
| **Maintenance**       | Trigger spaghetti difficile à suivre | Logique dispersée batch + triggers | Logique claire : trigger léger + worker |

---

### Points d’attention

- **Documentation claire :**

  - Définir ce qui est géré par les triggers (insertion dans la queue).
  - Définir ce qui est géré par les workers (traitement métier).

- **Monitoring & Alerting :**

  - Longueur de la file.
  - Messages en échec.
  - Temps de traitement moyen.

- **Idempotence des workers :**

  - Chaque worker doit pouvoir retraiter un message sans provoquer de doublons (stock négatif par exemple).

- **Séparation stricte :**

  - Ne jamais mélanger mise à jour immédiate et traitement via queue pour le même événement.

---

### Conclusion

- Les **triggers complexes** posent des problèmes de performance et de concurrence.
- Les **batchs temporels** introduisent une latence et ne garantissent pas l’ordre des transactions.
- La **solution moderne** est de déléguer les traitements lourds à des **workers événementiels**, alimentés par une **queue**.
- Cette architecture est :

  - réactive,
  - scalable,
  - robuste aux erreurs,
  - cohérente avec les besoins métier critiques.

👉 L’approche recommandée est donc : **Trigger minimal → Événement dans une queue → Worker qui consomme et applique la logique métier**.
