### Monitoring et Gestion des Pannes des Batchs SQL avec Airflow

### ✅ Introduction

Dans cette documentation, nous allons explorer comment Apache Airflow peut être utilisé pour monitorer, observer et gérer les pannes des batchs SQL. Nous aborderons également les différents opérateurs SQL proposés par Airflow et leurs cas d’utilisation, avec des exemples concrets d'implémentation. Nous inclurons également une ouverture vers la gestion de l'asynchronisme et le pattern SAGA.

---

### ✅ Pourquoi Airflow pour Monitorer et Gérer les Pannes des Batchs SQL ?

* **Monitoring Avancé :** Interface utilisateur pour suivre l’état des tâches, logs centralisés, notifications intégrées.
* **Gestion des Replays :** Gestion des retries (`retries`, `retry_delay`) et des stratégies de fallback (`on_failure_callback`).
* **Gestion des Pannes :** Airflow peut exécuter des tâches compensatoires (`trigger_rule='one_failed'`).
* **Observation des Batchs SQL :** Intégration native avec plusieurs bases de données SQL via des opérateurs dédiés.

---

### ✅ Gestion des Pannes dans Airflow :

1. **Gestion des Retrys :**

   * Paramétrage des `retries` et `retry_delay` pour relancer automatiquement les batchs SQL échoués.
   * **Exemple :**

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'retries': 3,
    'retry_delay': timedelta(minutes=10)
}

with DAG('sql_retry_dag', default_args=default_args, start_date=datetime(2025, 5, 1), schedule_interval='@daily') as dag:
    run_query = PostgresOperator(
        task_id='run_sql_query',
        postgres_conn_id='postgres_default',
        sql="""
            SELECT * FROM orders WHERE order_date >= NOW() - INTERVAL '1 DAY';
        """
    )
```

2. **Gestion des Erreurs Avancées :**

   * Utilisation des `on_failure_callback` pour exécuter des actions compensatoires (rollback, notifications).
   * **Exemple :**

```python
from airflow.operators.python import PythonOperator

def handle_failure(context):
    print(f"Task {context['task_instance'].task_id} failed. Error: {context['exception']}")

run_sql_query = PostgresOperator(
    task_id='run_sql_query',
    postgres_conn_id='postgres_default',
    sql="""SELECT * FROM non_existing_table;""",
    on_failure_callback=handle_failure
)
```

3. **Transactionnalité :**

   * Airflow permet d’encapsuler des requêtes SQL dans des transactions (`BEGIN`, `COMMIT`, `ROLLBACK`).
   * **Exemple :**

```python
sql_query = """
    BEGIN;
    INSERT INTO sales (order_id, amount) VALUES (1, 100);
    COMMIT;
"""

transaction_task = PostgresOperator(
    task_id='transaction_task',
    postgres_conn_id='postgres_default',
    sql=sql_query
)
```

4. **Logs Centralisés :**

   * Airflow enregistre les logs de chaque tâche, permettant de suivre chaque étape du workflow SQL.
   * Ces logs peuvent être visualisés dans l’interface utilisateur ou exportés vers des systèmes externes (S3, Elasticsearch).

---

### ✅ Gestion de l'Atomicité dans Airflow :

* Airflow permet de structurer des transactions SQL complexes en plusieurs étapes, tout en s'assurant que les modifications ne sont appliquées que si toutes les étapes réussissent.
* Exemple :

```python
sql_transaction = """
BEGIN;
INSERT INTO modifications_temp (id, data) SELECT id, data FROM source_table WHERE modified_at >= NOW();
-- Effectuer des vérifications
UPDATE source_table SET data = temp.data FROM modifications_temp temp WHERE source_table.id = temp.id;
COMMIT;
"""

transaction_task = PostgresOperator(
    task_id='transaction_task',
    postgres_conn_id='postgres_default',
    sql=sql_transaction
)
```

---

### ✅ Ouverture vers la Gestion de l'Asynchronisme et du Pattern SAGA :

* **Airflow vs SAGA :**

  * Airflow est basé sur des tâches synchrones (`task-based`) et n'intègre pas nativement le pattern SAGA.
  * Le pattern SAGA divise une transaction globale en sous-transactions indépendantes et compensables, ce qui n’est pas le cas d’Airflow.
  * Airflow peut partiellement simuler un SAGA en utilisant des `trigger_rule='one_failed'` pour déclencher des tâches compensatoires, mais ce n’est pas automatique.

* **Gestion des Événements Asynchrones :**

  * Airflow est principalement orienté tâches séquentielles, tandis que Kafka et Temporal sont orientés événements (`event sourcing`).
  * **Kafka :** Gestion des événements asynchrones en temps réel (`publish/subscribe`).
  * **Temporal :** Orchestration des workflows transactionnels avec gestion des compensations (`rollback`).

* **Exemple de Workflow SAGA avec Temporal :**

```python
from temporalio import workflow

@workflow.defn
class OrderWorkflow:
    @workflow.run
    async def run(self, order_id: int):
        try:
            await workflow.execute_activity("create_order", order_id)
            await workflow.execute_activity("reserve_stock", order_id)
            await workflow.execute_activity("debit_account", order_id)
        except Exception:
            await workflow.execute_activity("cancel_order", order_id)
            raise
```

---

### ✅ Recommandation :

* Utiliser `on_failure_callback` pour exécuter des compensations SQL en cas d’échec.
* Exploiter les opérateurs SQL dédiés pour chaque base de données afin de bénéficier des fonctionnalités natives (transactions, logs).
* Pour les workflows strictement séquentiels, utiliser Airflow.
* Pour les workflows asynchrones ou transactionnels, envisager Kafka ou Temporal pour une gestion native des événements et des compensations.
