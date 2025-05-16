## Monitoring et Gestion des Pannes des Batchs SQL avec Apache Airflow


- [Introduction](#introduction)
- [Pourquoi Utiliser Apache Airflow pour le Monitoring des Batchs SQL ?](#pourquoi-utiliser-apache-airflow-pour-le-monitoring-des-batchs-sql-)
- [Exemples d'Implémentation](#exemples-dimplémentation-)

### Introduction

Apache Airflow est une plateforme d'orchestration de workflows qui permet de gérer, observer et monitorer des batchs SQL de manière efficace. Grâce à ses opérateurs SQL dédiés, ses fonctionnalités de gestion des erreurs et ses outils de monitoring, Airflow est une solution robuste pour le suivi des traitements SQL complexes.

---

### Pourquoi Utiliser Apache Airflow pour le Monitoring des Batchs SQL ?

1. **Interface Utilisateur Complète :**

   * Monitoring visuel des tâches via des vues `Tree`, `Graph`, `Gantt`.
   * Accès centralisé aux logs de chaque tâche SQL, avec possibilité d'exportation.

2. **Gestion des Retrys et des Replays :**

   * Rejouer les tâches échouées grâce aux paramètres `retries` et `retry_delay`.
   * Réexécuter des DAGs spécifiques via l'interface.

3. **Notifications et Alerte en Cas d’Échec :**

   * Notifications par email (`email_on_failure`) ou via des callbacks (`on_failure_callback`).
   * Intégration avec Slack, Webhooks ou d'autres services de messagerie.

4. **Gestion des Transactions SQL :**

   * Airflow permet d'encapsuler des opérations SQL dans des transactions (`BEGIN`, `COMMIT`, `ROLLBACK`).

---

### Exemples d'Implémentation :

#### 1. Gestion des Retrys :

L'implémentation des retries permet de relancer automatiquement un batch SQL échoué :

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

#### 2. Gestion des Erreurs Avancées avec Callbacks :

```python
from airflow.operators.python import PythonOperator

def handle_failure(context):
    print(f"Task {context['task_instance'].task_id} failed. Error: {context['exception']}")

run_sql_query = PostgresOperator(
    task_id='run_sql_query',
    postgres_conn_id='postgres_default',
    sql="SELECT * FROM non_existing_table;",
    on_failure_callback=handle_failure
)
```

#### 3. Gestion des Transactions SQL :

Airflow permet de structurer des transactions SQL en plusieurs étapes tout en maintenant une cohérence transactionnelle :

```python
sql_transaction = """
    BEGIN;
    INSERT INTO modifications_temp (id, data) SELECT id, data FROM source_table WHERE modified_at >= NOW();
    -- Vérifications des données
    UPDATE source_table SET data = temp.data FROM modifications_temp temp WHERE source_table.id = temp.id;
    COMMIT;
"""

transaction_task = PostgresOperator(
    task_id='transaction_task',
    postgres_conn_id='postgres_default',
    sql=sql_transaction
)
```

#### 4. Centralisation des Logs :

Les logs des tâches SQL sont accessibles via l'interface utilisateur et peuvent être exportés vers S3, Elasticsearch, etc.

---

### ✅ Opérateurs SQL Disponibles dans Airflow :

Apache Airflow propose une large gamme d'opérateurs SQL pour interagir avec différentes bases de données, allant bien au-delà des classiques `PostgresOperator`, `MySqlOperator` et `MSSqlOperator`. Voici une liste complète des opérateurs SQL disponibles :

* **PostgreSQL :** `PostgresOperator`, `PostgresHook`
* **MySQL :** `MySqlOperator`, `MySqlHook`
* **Microsoft SQL Server :** `MSSqlOperator`, `MSSqlHook`
* **Oracle :** `OracleOperator`, `OracleHook`
* **Snowflake :** `SnowflakeOperator`, `SnowflakeHook`
* **BigQuery :** `BigQueryOperator`, `BigQueryHook`
* **Redshift :** `RedshiftSQLOperator`
* **Hive :** `HiveOperator`
* **Presto / Trino :** `PrestoOperator`, `TrinoOperator`
* **ODBC / JDBC :** `ODBCOperator`, `JDBCOperator`
* **SQLite :** `SqliteOperator`
* **Vertica :** `VerticaOperator`
* **Drill :** `DrillOperator`
* **Impala :** `ImpalaOperator`
* **ClickHouse :** `ClickHouseOperator`
* **Teradata :** `TeradataOperator`
* **SAP HANA :** `SAPHanaOperator`

Chaque opérateur est conçu pour interagir avec un type spécifique de base de données, en utilisant des hooks (`PostgresHook`, `MySqlHook`, etc.) pour établir des connexions et exécuter des requêtes SQL. Ces opérateurs offrent des fonctionnalités telles que l’exécution de requêtes SQL, l’extraction de données et la gestion des transactions.

**Exemple : Utilisation de plusieurs opérateurs SQL dans un même workflow :**

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.mysql.operators.mysql import MySqlOperator
from airflow.providers.microsoft.mssql.operators.mssql import MsSqlOperator
from datetime import datetime

default_args = {
    'owner': 'airflow',
    'start_date': datetime(2025, 5, 1),
    'retries': 1
}

with DAG('multi_db_workflow', default_args=default_args, schedule_interval='@daily') as dag:

    # PostgreSQL Task
    extract_postgres = PostgresOperator(
        task_id='extract_data_from_postgres',
        postgres_conn_id='postgres_default',
        sql='SELECT * FROM sales WHERE sale_date = CURRENT_DATE;'
    )

    # MySQL Task
    load_mysql = MySqlOperator(
        task_id='load_data_to_mysql',
        mysql_conn_id='mysql_default',
        sql='INSERT INTO sales_archive (id, amount) VALUES (%s, %s);'
    )

    # MSSQL Task
    log_mssql = MsSqlOperator(
        task_id='log_data_in_mssql',
        mssql_conn_id='mssql_default',
        sql='INSERT INTO log_table (message) VALUES ('Data synchronized successfully');'
    )

    extract_postgres >> load_mysql >> log_mssql
```

En résumé, Airflow propose une gamme d’opérateurs SQL étendue permettant de se connecter à pratiquement n’importe quelle base de données. Cela permet de centraliser l'orchestration des workflows SQL au sein d’un même environnement Airflow tout en tirant parti des fonctionnalités transactionnelles de chaque SGBD.

### ✅ Gestion de l'Atomicité dans Airflow :

Apache Airflow peut être configuré pour gérer l'atomicité des opérations SQL. Cela implique d'assurer que certaines étapes d'un workflow SQL ne s'exécutent que si d'autres ont réussi. Cela peut être réalisé en utilisant des transactions SQL (`BEGIN`, `COMMIT`, `ROLLBACK`) et des stratégies de compensation (`on_failure_callback`).

#### **Exemple : Workflow Transactionnel avec Airflow**

Dans cet exemple, nous allons :

* Capturer des modifications dans une table temporaire (`modifications_temp`).
* Exécuter un batch SQL principal.
* Si le batch réussit, appliquer les modifications.
* Si le batch échoue, annuler les modifications.

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

def rollback_modifications(**context):
    # Logique de rollback
    print("Rollback des modifications appliquées")

with DAG(
    'atomic_transaction_workflow',
    default_args=default_args,
    start_date=datetime(2025, 5, 15),
    schedule_interval='@daily',
    catchup=False
) as dag:

    # Étape 1 : Capturer les modifications dans une table temporaire
    capture_modifications = PostgresOperator(
        task_id='capture_modifications',
        postgres_conn_id='postgres_default',
        sql="""
        INSERT INTO modifications_temp (id, data)
        SELECT id, data FROM source_table WHERE modified_at >= NOW() - INTERVAL '1 DAY';
        """
    )

    # Étape 2 : Exécuter le batch SQL
    run_batch = PostgresOperator(
        task_id='run_batch',
        postgres_conn_id='postgres_default',
        sql="""
        BEGIN;
        INSERT INTO batch_results (data)
        SELECT data FROM modifications_temp;
        COMMIT;
        """
    )

    # Étape 3 : Appliquer les modifications uniquement si le batch réussit
    apply_modifications = PostgresOperator(
        task_id='apply_modifications',
        postgres_conn_id='postgres_default',
        sql="""
        UPDATE source_table SET data = mt.data
        FROM modifications_temp mt
        WHERE source_table.id = mt.id;
        """,
        trigger_rule='all_success'
    )

    # Étape 4 : Rollback en cas d'échec
    rollback_task = PythonOperator(
        task_id='rollback_task',
        python_callable=rollback_modifications,
        trigger_rule='one_failed'
    )

    capture_modifications >> run_batch >> [apply_modifications, rollback_task
```

#### ✅ **Pourquoi cette approche ?**

* **Atomicité :** Les modifications ne sont appliquées que si toutes les étapes réussissent.
* **Gestion des erreurs :** Le rollback est exécuté en cas d'échec pour garantir la cohérence des données.
* **Gestion transactionnelle :** Les transactions SQL (`BEGIN`, `COMMIT`, `ROLLBACK`) assurent la cohérence des opérations.

En résumé, Apache Airflow permet de structurer des workflows transactionnels en s'appuyant sur les mécanismes natifs des SGBD tout en intégrant une gestion des erreurs via les `on_failure_callback` et les `trigger_rule`.

### ✅ Connexion à Plusieurs Bases de Données dans Airflow :

Apache Airflow est conçu pour se connecter à plusieurs bases de données simultanément, ce qui permet d’orchestrer des workflows complexes impliquant des sources de données hétérogènes. Chaque connexion est définie par un `conn_id` distinct et peut cibler une base de données différente.

#### **Configuration des Connexions :**

* Les connexions peuvent être configurées via l'interface Airflow (`Admin > Connections`) ou via des variables d'environnement.
* Chaque connexion est identifiée par un `conn_id` unique (ex : `postgres_conn`, `mysql_conn`, `mssql_conn`).

#### **Exemple : Workflow Multi-Bases de Données :**

Dans cet exemple, nous allons orchestrer un workflow qui :

* Extrait des données de PostgreSQL (`sales_db`),
* Les charge dans MySQL (`inventory_db`),
* Enregistre un log dans MSSQL (`logging_db`).

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.mysql.operators.mysql import MySqlOperator
from airflow.providers.microsoft.mssql.operators.mssql import MsSqlOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'retries': 2,
    'retry_delay': timedelta(minutes=10),
    'start_date': datetime(2025, 5, 15)
}

with DAG('multi_db_workflow', default_args=default_args, schedule_interval='@daily') as dag:

    # Extraction des données depuis PostgreSQL
    extract_sales = PostgresOperator(
        task_id='extract_sales_data',
        postgres_conn_id='postgres_conn',
        sql="SELECT * FROM sales WHERE sale_date = CURRENT_DATE;"
    )

    # Chargement des données dans MySQL
    load_inventory = MySqlOperator(
        task_id='load_inventory_data',
        mysql_conn_id='mysql_conn',
        sql="INSERT INTO inventory (product_id, quantity) VALUES (%s, %s);"
    )

    # Enregistrement du log dans MSSQL
    log_process = MsSqlOperator(
        task_id='log_process',
        mssql_conn_id='mssql_conn',
        sql="INSERT INTO process_log (message) VALUES ('Data synchronization completed.');"
    )

    extract_sales >> load_inventory >> log_process
```

#### ✅ **Pourquoi cette approche ?**

* **Centralisation des Workflows :** Un seul DAG peut orchestrer des opérations sur plusieurs bases de données.
* **Gestion des Dépendances :** Airflow permet de contrôler l’ordre d’exécution des tâches, garantissant ainsi l’intégrité des opérations inter-bases.
* **Flexibilité :** Chaque tâche peut se connecter à une base de données différente, facilitant les intégrations complexes.

En résumé, Airflow permet de se connecter à plusieurs bases de données simultanément, d'orchestrer des workflows multi-sources et de centraliser les opérations SQL complexes au sein d'une architecture unique.

### ✅ Synchronisation des Données entre Bases de Données avec Airflow :

Apache Airflow est parfaitement adapté pour orchestrer des workflows de synchronisation de données entre plusieurs bases de données. Cela permet de centraliser les opérations de transfert, de transformation et de validation des données tout en maintenant la cohérence des données entre les systèmes.

#### **Cas d'Usage :**

* Synchroniser les données de vente d'une base PostgreSQL (`sales_db`) vers une base MySQL (`inventory_db`).
* Fusionner les données consolidées dans une base de reporting MSSQL (`reporting_db`).

#### **Exemple : Workflow de Synchronisation des Données :**

Dans cet exemple, nous allons :

* Extraire des données de PostgreSQL.
* Les transformer en utilisant Python.
* Les charger dans MySQL.
* Enregistrer un log dans MSSQL.

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.mysql.operators.mysql import MySqlOperator
from airflow.providers.microsoft.mssql.operators.mssql import MsSqlOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import pandas as pd

default_args = {
    'owner': 'airflow',
    'start_date': datetime(2025, 5, 15),
    'retries': 2,
    'retry_delay': timedelta(minutes=10)
}

def transform_data(**context):
    # Exemple de transformation : Calcul du total des ventes
    df = pd.read_csv('/tmp/sales_data.csv')
    df['total_amount'] = df['quantity'] * df['price']
    df.to_csv('/tmp/transformed_data.csv', index=False)

with DAG('data_sync_workflow', default_args=default_args, schedule_interval='@daily') as dag:

    # Extraction des données depuis PostgreSQL
    extract_data = PostgresOperator(
        task_id='extract_data',
        postgres_conn_id='postgres_conn',
        sql="""
        COPY (SELECT id, quantity, price FROM sales WHERE sale_date = CURRENT_DATE) TO '/tmp/sales_data.csv' WITH CSV HEADER;
        """
    )

    # Transformation des données
    transform_task = PythonOperator(
        task_id='transform_data',
        python_callable=transform_data
    )

    # Chargement des données dans MySQL
    load_data = MySqlOperator(
        task_id='load_data',
        mysql_conn_id='mysql_conn',
        sql="""
        LOAD DATA INFILE '/tmp/transformed_data.csv'
        INTO TABLE sales_summary
        FIELDS TERMINATED BY ','
        ENCLOSED BY '"'
        LINES TERMINATED BY '
'
        (id, quantity, price, total_amount);
        """
    )

    # Enregistrement du log dans MSSQL
    log_sync = MsSqlOperator(
        task_id='log_sync',
        mssql_conn_id='mssql_conn',
        sql="""
        INSERT INTO sync_log (message) VALUES ('Data synchronization completed successfully');
        """
    )

    extract_data >> transform_task >> load_data >> log_sync
```

#### ✅ **Pourquoi cette approche ?**

* **Orchestration Centralisée :** Un seul workflow gère l'extraction, la transformation et le chargement des données.
* **Scalabilité :** Chaque tâche est indépendante et peut être parallélisée pour un traitement plus rapide.
* **Gestion des Dépendances :** Les tâches s'exécutent dans un ordre défini, garantissant l'intégrité des données synchronisées.

En résumé, Airflow permet de synchroniser des données entre plusieurs bases de données en orchestrant les tâches de manière séquentielle ou parallèle. Cela permet de centraliser le contrôle des workflows, de garantir l'intégrité des données et de documenter chaque étape de la synchronisation.


### Gestion des Tables Source et Staging pour l'Affichage des Données aux Utilisateurs dans Airflow

Dans le cadre d'Airflow, il est courant de séparer les données brutes (staging) des données validées (source). Cependant, pour l'affichage des données aux utilisateurs, il est pertinent de créer une vue consolidée qui combine ces deux tables, permettant ainsi de présenter des informations à jour tout en maintenant l'intégrité des données.

---

#### ✅ Pourquoi Combiner les Tables Source et Staging ?

* **Expérience Utilisateur :** Fournir une vue unique qui combine les données validées (source) et les données en cours de traitement (staging).
* **Gestion des Données Temporairement Incomplètes :** Les données de staging peuvent être marquées comme `PENDING` ou `VALIDATED`, permettant ainsi de différencier les données prêtes à être validées.
* **Centralisation des Données :** Les utilisateurs accèdent à une seule source d'information (`sales_view`) sans avoir à connaître la structure des tables sous-jacentes.

---

#### ✅ Exemple : Architecture avec Table Source, Staging et Vue Consolidée

**Tables :**

* `sales` - Table Source (données validées)
* `sales_staging` - Table de Staging (données en cours de traitement)
* `sales_view` - Vue consolidée pour l'affichage aux utilisateurs

---

#### **Structure des Tables :**

**Table Source - `sales` :**

| sale\_id | customer\_id | product\_id | amount | status    |
| -------- | ------------ | ----------- | ------ | --------- |
| 1        | 101          | 1001        | 500    | VALIDATED |
| 2        | 102          | 1002        | 250    | VALIDATED |

**Table de Staging - `sales_staging` :**

| sale\_id | customer\_id | product\_id | amount | status  |
| -------- | ------------ | ----------- | ------ | ------- |
| 3        | 103          | 1003        | 300    | PENDING |
| 4        | 104          | 1004        | 150    | PENDING |

---

### ✅ Création de la Vue Consolidée :

```sql
CREATE OR REPLACE VIEW sales_view AS
SELECT
    sale_id,
    customer_id,
    product_id,
    amount,
    status
FROM
    sales
UNION ALL
SELECT
    sale_id,
    customer_id,
    product_id,
    amount,
    status
FROM
    sales_staging;
```

Cette vue `sales_view` permet aux utilisateurs d'accéder à l'ensemble des données disponibles, y compris les données en cours de validation (`PENDING`) et les données déjà validées (`VALIDATED`).

---

#### ✅ Workflow Airflow : Gestion des Données et Actualisation de la Vue

* **Tâche 1 :** Charger les nouvelles données dans `sales_staging`.
* **Tâche 2 :** Valider et transférer les données de `sales_staging` vers `sales`.
* **Tâche 3 :** Actualiser la vue `sales_view`.

**DAG : workflow\_data\_sync.py**

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'start_date': datetime(2025, 5, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=10)
}

with DAG('workflow_data_sync', default_args=default_args, schedule_interval='@daily', catchup=False) as dag:

    # 1. Charger les données dans la table de staging
    load_staging = PostgresOperator(
        task_id='load_staging',
        postgres_conn_id='postgres_default',
        sql="""
        COPY sales_staging FROM '/path/to/new_sales_data.csv' WITH CSV HEADER;
        """
    )

    # 2. Valider les données et transférer dans la table source
    validate_and_transfer = PostgresOperator(
        task_id='validate_and_transfer',
        postgres_conn_id='postgres_default',
        sql="""
        INSERT INTO sales (sale_id, customer_id, product_id, amount, status)
        SELECT
            sale_id, customer_id, product_id, amount, 'VALIDATED'
        FROM sales_staging
        WHERE status = 'PENDING';

        DELETE FROM sales_staging WHERE status = 'PENDING';
        """
    )

    # 3. Créer ou Actualiser la vue consolidée
    refresh_view = PostgresOperator(
        task_id='refresh_view',
        postgres_conn_id='postgres_default',
        sql="""
        CREATE OR REPLACE VIEW sales_view AS
        SELECT
            sale_id, customer_id, product_id, amount, status
        FROM sales
        UNION ALL
        SELECT
            sale_id, customer_id, product_id, amount, status
        FROM sales_staging;
        """
    )

    load_staging >> validate_and_transfer >> refresh_view
```

---

#### ✅ Recommandations :

* **Centraliser l'affichage des données via une vue unique** (`sales_view`), facilitant l'accès aux informations pour les utilisateurs.
* **Rafraîchir la vue après chaque batch ETL** pour garantir des données actualisées.
* **Gérer les statuts (`PENDING`, `VALIDATED`) dans les données de staging** pour indiquer clairement l'état des données en cours de traitement.

Cette approche permet de consolider les données sans dupliquer les informations tout en maintenant une distinction claire entre les données validées et celles en cours de traitement.


### ✅ Airflow et le Pattern SAGA : Comparaison et Limitations

Apache Airflow est un orchestrateur de tâches synchrones, tandis que le pattern SAGA est un modèle de gestion des transactions distribuées asynchrones. Comprendre cette distinction est essentiel pour choisir la bonne architecture.

#### **Qu'est-ce que le Pattern SAGA ?**

* Le pattern SAGA divise une transaction distribuée en plusieurs sous-transactions indépendantes, chacune ayant une opération compensatoire associée.
* En cas d’échec d’une sous-transaction, SAGA exécute une action de compensation (`rollback`), permettant de maintenir la cohérence des données de bout en bout.
* Chaque étape est déclenchée par un événement, rendant le workflow asynchrone.

#### **Pourquoi Airflow ne remplace pas le Pattern SAGA ?**

* Airflow est basé sur un modèle `task-based` où chaque tâche est exécutée séquentiellement ou parallèlement, mais sans notion d'événements asynchrones.
* Les tâches dans Airflow sont indépendantes, et il n'existe pas de gestion native des transactions distribuées avec compensation.
* Les `on_failure_callback` et `trigger_rule` permettent d'implémenter des actions compensatoires, mais cela reste manuel et non atomique.

#### **Exemple : Simulation d'un Workflow SAGA dans Airflow :**

Dans cet exemple, nous allons :

* Créer une commande (`create_order`).
* Réserver le stock (`reserve_stock`).
* Débiter le compte (`debit_account`).
* En cas d'échec, exécuter des compensations (`cancel_order`, `release_stock`).

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'start_date': datetime(2025, 5, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

with DAG('saga_workflow', default_args=default_args, schedule_interval='@daily') as dag:

    create_order = PostgresOperator(
        task_id='create_order',
        postgres_conn_id='postgres_default',
        sql="INSERT INTO orders (order_id, status) VALUES (1, 'CREATED');"
    )

    reserve_stock = PostgresOperator(
        task_id='reserve_stock',
        postgres_conn_id='postgres_default',
        sql="UPDATE inventory SET reserved = reserved + 1 WHERE product_id = 101;",
        trigger_rule='all_success'
    )

    debit_account = PostgresOperator(
        task_id='debit_account',
        postgres_conn_id='postgres_default',
        sql="UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;",
        trigger_rule='all_success'
    )

    # Compensation Tasks
    cancel_order = PostgresOperator(
        task_id='cancel_order',
        postgres_conn_id='postgres_default',
        sql="DELETE FROM orders WHERE order_id = 1;",
        trigger_rule='one_failed'
    )

    release_stock = PostgresOperator(
        task_id='release_stock',
        postgres_conn_id='postgres_default',
        sql="UPDATE inventory SET reserved = reserved - 1 WHERE product_id = 101;",
        trigger_rule='one_failed'
    )

    create_order >> [reserve_stock, cancel_order]
    reserve_stock >> [debit_account, release_stock]
```

#### ✅ **Limites de cette approche :**

* La gestion des transactions est manuelle, avec une logique `trigger_rule` non transactionnelle.
* Airflow ne gère pas l’état des transactions distribuées ; il faut coder les compensations explicitement.
* Airflow ne permet pas de réagir à des événements externes (`event sourcing`), contrairement à Kafka ou Temporal.

#### ✅ **Alternatives :**

* **Kafka :** Pour gérer des événements asynchrones et orchestrer des transactions via des topics (`ORDER_PLACED`, `PAYMENT_COMPLETED`).
* **Temporal :** Orchestrateur transactionnel avec gestion native des compensations (`rollback`) et reprise (`replay`).
* **Zeebe :** Moteur BPMN pour orchestrer des workflows transactionnels distribués.

En conclusion, Airflow peut partiellement imiter le pattern SAGA en structurant des tâches compensatoires via `trigger_rule`, mais il n’offre pas de gestion asynchrone des transactions distribuées. Pour des transactions complexes avec gestion d'état et événements asynchrones, Kafka, Temporal ou Zeebe sont plus adaptés.

### ✅ Gestion des Compensations dans Airflow vs Pattern SAGA

Dans un workflow basé sur le pattern SAGA, les données sources sont modifiées de manière incrémentale à chaque étape, avec des tâches de compensation (`rollback`) en cas d’échec. Cela permet de maintenir une cohérence transactionnelle distribuée. Cependant, Airflow ne gère pas nativement ce type de compensation asynchrone.

#### **Pourquoi Airflow ne permet pas de laisser les données sources s'altérer avec des compensations en cas d'échec ?**

* **Modèle Synchrone :** Airflow exécute des tâches de manière séquentielle ou parallèle, mais il ne gère pas les événements asynchrones (`event sourcing`).
* **Pas de Gestion Transactionnelle Distribuée :** Airflow est conçu pour orchestrer des tâches indépendantes, sans logique de `rollback` transactionnel intégré.
* **Compensations Manuelles :** Les tâches compensatoires (`on_failure_callback`, `trigger_rule='one_failed'`) doivent être explicitement définies et codées.
* **Absence de Gestion d'État :** Airflow ne conserve pas d'état transactionnel pour suivre les étapes compensatoires comme le fait SAGA.

#### **Exemple : Implémentation Manuelle de Compensations dans Airflow :**

Dans cet exemple, nous allons :

* Créer une commande (`create_order`).
* Réserver le stock (`reserve_stock`).
* Débiter le compte (`debit_account`).
* Si une étape échoue, exécuter des tâches de compensation (`cancel_order`, `release_stock`).

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'start_date': datetime(2025, 5, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

def rollback_order(**context):
    print("Rollback : Suppression de la commande")

with DAG('saga_simulation', default_args=default_args, schedule_interval='@daily') as dag:

    create_order = PostgresOperator(
        task_id='create_order',
        postgres_conn_id='postgres_default',
        sql="INSERT INTO orders (order_id, status) VALUES (1, 'CREATED');"
    )

    reserve_stock = PostgresOperator(
        task_id='reserve_stock',
        postgres_conn_id='postgres_default',
        sql="UPDATE inventory SET reserved = reserved + 1 WHERE product_id = 101;",
        trigger_rule='all_success'
    )

    debit_account = PostgresOperator(
        task_id='debit_account',
        postgres_conn_id='postgres_default',
        sql="UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;",
        trigger_rule='all_success'
    )

    # Compensation Tasks
    cancel_order = PythonOperator(
        task_id='cancel_order',
        python_callable=rollback_order,
        trigger_rule='one_failed'
    )

    release_stock = PostgresOperator(
        task_id='release_stock',
        postgres_conn_id='postgres_default',
        sql="UPDATE inventory SET reserved = reserved - 1 WHERE product_id = 101;",
        trigger_rule='one_failed'
    )

    create_order >> [reserve_stock, cancel_order]
    reserve_stock >> [debit_account, release_stock]
```

#### ✅ **Limites de cette Approche :**

* La gestion des compensations est **manuelle** et dépend de la logique `trigger_rule`.
* Si une tâche échoue au milieu du workflow, il n’y a **pas de mécanisme automatique de reprise** (`retry with compensation`).
* Les modifications sont appliquées immédiatement, sans attendre la réussite complète du workflow.

#### ✅ **Recommandation :**

* Pour des transactions distribuées complexes avec gestion d’état et compensations automatiques, envisager des orchestrateurs asynchrones comme **Temporal** ou **Zeebe**.
* Utiliser Kafka pour implémenter un système d’événements asynchrones permettant de réagir aux échecs et d’appliquer des compensations.

En résumé, Airflow peut simuler le pattern SAGA en structurant manuellement des tâches compensatoires, mais il n’offre pas de gestion native des transactions distribuées avec rollback automatique. Pour cela, des solutions spécialisées comme **Temporal** ou **Zeebe** sont plus adaptées.

### ✅ Gestion des Événements Externes avec Kafka et Temporal :

Lorsque les workflows dépendent d'événements externes (`event sourcing`), Apache Airflow atteint ses limites car il est basé sur des tâches séquentielles et synchrones. En revanche, des outils comme **Kafka** et **Temporal** sont spécifiquement conçus pour orchestrer des workflows basés sur des événements asynchrones.

#### **Qu'est-ce que l'Event Sourcing ?**

* **Event Sourcing** est une approche où les modifications d'état d'un système sont représentées sous forme d'événements immuables (`ORDER_PLACED`, `PAYMENT_COMPLETED`).
* Chaque événement est stocké dans un journal (`event log`) et peut être rejoué pour reconstruire l'état actuel du système ou pour compenser des actions échouées.

#### **Pourquoi Airflow n'est pas Idéal pour l'Event Sourcing ?**

* Airflow est conçu pour exécuter des tâches de manière séquentielle (`task-based`), sans gestion native des événements asynchrones.
* Les `Sensors` d'Airflow (`HttpSensor`, `FileSensor`) effectuent du `polling` actif, ce qui est inefficace pour un traitement événementiel.
* Les workflows dans Airflow sont stateless, tandis que le `event sourcing` nécessite une gestion d'état transactionnelle.

#### **Kafka : Gestion des Événements en Temps Réel**

* **Kafka** est un broker de messages orienté événements qui permet de publier et de consommer des événements en temps réel.
* Chaque service publie des événements (`ORDER_PLACED`, `STOCK_RESERVED`) et d'autres services peuvent les consommer.

**Exemple : Gestion d'événements avec Kafka :**

```python
from kafka import KafkaProducer, KafkaConsumer
import json

producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Publier un événement
order_event = {"order_id": 123, "status": "ORDER_PLACED"}
producer.send('order_events', value=order_event)

# Consommer les événements
consumer = KafkaConsumer(
    'order_events',
    bootstrap_servers='localhost:9092',
    value_deserializer=lambda v: json.loads(v.decode('utf-8'))
)

for message in consumer:
    print(f"Received event: {message.value}")
```

#### **Temporal : Gestion des Workflows Asynchrones**

* **Temporal** est un orchestrateur de workflows asynchrones qui gère les transactions distribuées, les événements et les compensations.
* Chaque étape du workflow est une activité (`activity`) et les événements déclenchent ces activités de manière asynchrone.

**Exemple : Workflow Asynchrone avec Temporal :**

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

#### ✅ **Comparaison Airflow vs Kafka vs Temporal :**

| **Aspect**               | **Airflow**               | **Kafka**                    | **Temporal**          |
| ------------------------ | ------------------------- | ---------------------------- | --------------------- |
| **Architecture**         | Tâches synchrones         | Événements asynchrones       | Workflows asynchrones |
| **Gestion d'État**       | Non (`stateless`)         | Oui (`event log`)            | Oui (`stateful`)      |
| **Compensation**         | Manuelle (`trigger_rule`) | Manuelle (via consommateurs) | Native (rollback)     |
| **Rejeu des Événements** | Non                       | Oui                          | Oui                   |

#### ✅ **Recommandation :**

* Si le workflow est basé sur des événements externes (`event sourcing`), Kafka ou Temporal sont plus adaptés qu’Airflow.
* Kafka permet de diffuser des événements en temps réel et de les rejouer pour compenser des actions échouées.
* Temporal est optimal pour des transactions distribuées nécessitant des compensations (`rollback`) et un suivi d'état transactionnel.

En résumé, Airflow est idéal pour les workflows synchrones, tandis que Kafka et Temporal sont conçus pour orchestrer des workflows asynchrones basés sur des événements externes et des transactions distribuées.

### ✅ Quand Passer à Kafka ou Temporal ?

La transition vers des plateformes comme **Kafka** ou **Temporal** se justifie lorsque les workflows deviennent plus complexes, distribués ou asynchrones. Voici les cas d'usage typiques où Kafka et Temporal deviennent plus adaptés qu'Apache Airflow :

#### **1. Gestion des Événements Asynchrones :**

* Si les workflows doivent réagir à des événements externes (`event sourcing`), Kafka devient essentiel pour gérer ces événements en temps réel.
* Exemple : Un service de commande (`order_service`) publie un événement `ORDER_PLACED`, et les services `inventory_service` et `payment_service` réagissent à cet événement indépendamment.

**Exemple Kafka :**

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# Producteur - Publier un événement
producer = KafkaProducer(bootstrap_servers='localhost:9092')
producer.send('order_events', json.dumps({'order_id': 123, 'status': 'ORDER_PLACED'}).encode('utf-8'))

# Consommateur - Écouter les événements
consumer = KafkaConsumer('order_events', bootstrap_servers='localhost:9092')
for message in consumer:
    print(f"Received event: {message.value.decode('utf-8')}")
```

#### **2. Transactions Distribuées avec Compensations (Pattern SAGA) :**

* Si un workflow implique plusieurs microservices avec des transactions distribuées (`reserve_stock`, `debit_account`), Temporal est plus adapté car il intègre des compensations (`rollback`) de manière native.
* Exemple : Si `reserve_stock` échoue, Temporal exécute `cancel_order` automatiquement.

**Exemple Temporal :**

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

#### **3. Gestion des États Persistants :**

* Si les workflows nécessitent un suivi d'état (`pending`, `processing`, `completed`), Temporal est plus adapté car il maintient les états de chaque étape du workflow.
* Airflow, étant `stateless`, ne peut pas reprendre une tâche en fonction de son état précédent sans logique personnalisée.

#### **4. Scalabilité Élastique :**

* Si le système doit gérer des millions d'événements par seconde (`high throughput`), Kafka est le choix naturel pour le traitement distribué.
* Temporal permet également d'exécuter plusieurs instances d'un même workflow sans gérer explicitement la concurrence.

#### **Résumé des Cas d'Usage :**

| **Scénario**                         | **Airflow**    | **Kafka** | **Temporal** |
| ------------------------------------ | -------------- | --------- | ------------ |
| Workflows Séquentiels                | ✅              | ❌         | ✅            |
| Event Sourcing / Événements Externes | ❌              | ✅         | ✅            |
| Transactions Distribuées (SAGA)      | ❌              | ✅         | ✅            |
| Gestion des États                    | ❌              | ✅         | ✅            |
| Scalabilité Élastique                | ✅ (Kubernetes) | ✅         | ✅            |

#### ✅ **Recommandation :**

* Si les workflows sont purement séquentiels et synchrones : **Airflow** est suffisant.
* Si les workflows nécessitent une orchestration événementielle (`event-driven`) : **Kafka** est indispensable.
* Si les workflows impliquent des transactions distribuées avec compensation (`SAGA pattern`) : **Temporal** est le choix optimal.

En résumé, Airflow est un excellent point de départ, mais Kafka et Temporal deviennent nécessaires lorsque les workflows évoluent vers des architectures distribuées, asynchrones ou transactionnelles.

### ✅ Airflow pour les Microservices : Est-ce le Bon Choix ?

Apache Airflow est un orchestrateur de workflows orienté tâches (`task-based`). Il est principalement conçu pour orchestrer des workflows séquentiels et synchrones, tels que des pipelines ETL, des traitements batch ou des tâches planifiées. Cependant, lorsqu'il s'agit de microservices asynchrones et distribués, son utilisation présente certaines limitations.

#### **Pourquoi Airflow n’est pas Optimal pour les Microservices ?**

* **Architecture Orientée Tâches :** Airflow est conçu pour orchestrer des tâches définies dans des DAGs (`Directed Acyclic Graph`). Chaque tâche est exécutée indépendamment et séquentiellement.
* **Pas de Gestion d'Événements Asynchrones :** Airflow ne gère pas les événements (`event sourcing`) comme Kafka ou Temporal. Il ne peut donc pas réagir en temps réel à des événements externes (`ORDER_PLACED`, `PAYMENT_COMPLETED`).
* **Pas de Gestion d'État Persistant :** Airflow est `stateless`, ce qui signifie qu'il n'a pas de gestion d'état transactionnelle native, contrairement à Temporal qui maintient l'état de chaque étape du workflow.
* **Compensations Manuelles :** Pour implémenter des compensations (`rollback`), il faut coder explicitement des tâches de compensation (`trigger_rule='one_failed'`). Cela n’est pas natif et nécessite une logique personnalisée.

#### **Exemple : Orchestration Microservices avec Airflow :**

Dans ce cas, Airflow peut être utilisé pour orchestrer des services synchrones :

* **Tâche 1 :** Créer une commande (`create_order`).
* **Tâche 2 :** Réserver le stock (`reserve_stock`).
* **Tâche 3 :** Débiter le compte (`debit_account`).

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'start_date': datetime(2025, 5, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

with DAG('microservice_workflow', default_args=default_args, schedule_interval='@daily') as dag:

    create_order = PostgresOperator(
        task_id='create_order',
        postgres_conn_id='postgres_default',
        sql="INSERT INTO orders (order_id, status) VALUES (1, 'CREATED');"
    )

    reserve_stock = PostgresOperator(
        task_id='reserve_stock',
        postgres_conn_id='postgres_default',
        sql="UPDATE inventory SET reserved = reserved + 1 WHERE product_id = 101;"
    )

    debit_account = PostgresOperator(
        task_id='debit_account',
        postgres_conn_id='postgres_default',
        sql="UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;"
    )

    create_order >> reserve_stock >> debit_account
```

#### ✅ **Quand Utiliser Kafka ou Temporal ?**

* **Kafka :** Si chaque microservice doit publier ou consommer des événements (`event-driven architecture`), Kafka devient le choix naturel. Exemple : `ORDER_PLACED`, `PAYMENT_COMPLETED` sont des événements consommés par différents services.
* **Temporal :** Si les workflows nécessitent des transactions distribuées avec des compensations (`rollback`) et une gestion d’état persistante, Temporal est plus adapté.

#### **Résumé : Airflow vs Kafka vs Temporal :**

| **Scénario**              | **Airflow** | **Kafka**     | **Temporal** |
| ------------------------- | ----------- | ------------- | ------------ |
| Orchestration Synchrone   | ✅           | ❌             | ✅            |
| Event Sourcing / Pub-Sub  | ❌           | ✅             | ✅            |
| Transactions Distribuées  | ❌           | ❌             | ✅            |
| Gestion d’État Persistant | ❌           | ✅ (Event Log) | ✅            |
| Rejeu des Événements      | ❌           | ✅             | ✅            |

En conclusion, **Airflow est le choix par défaut pour les workflows synchrones et séquentiels**, mais dès que les workflows deviennent asynchrones, basés sur des événements ou transactionnels, Kafka et Temporal deviennent plus adaptés pour orchestrer des microservices.

### Première Migration vers une Architecture Microservices avec Airflow, Kafka et Temporal

Lorsqu’on envisage une migration vers une architecture microservices, il est essentiel d’adopter une approche progressive afin de limiter les risques et de garantir la stabilité du système existant. Voici une approche structurée pour réussir cette transition :

#### **Phase 1 : Externalisation des Tâches avec Airflow**

* **Objectif :** Séparer les composants monolithiques en services indépendants orchestrés par Airflow.
* **Pourquoi Airflow ?** Airflow permet d’orchestrer des tâches séquentielles de manière centralisée, tout en assurant le monitoring et le logging des opérations.

**Exemple : Orchestration des tâches synchrones avec Airflow :**

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'start_date': datetime(2025, 5, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

with DAG('first_migration', default_args=default_args, schedule_interval='@daily') as dag:

    create_order = PostgresOperator(
        task_id='create_order',
        postgres_conn_id='postgres_conn',
        sql="INSERT INTO orders (id, status) VALUES (1, 'CREATED');"
    )

    reserve_stock = PostgresOperator(
        task_id='reserve_stock',
        postgres_conn_id='postgres_conn',
        sql="UPDATE inventory SET reserved = reserved + 1 WHERE product_id = 101;"
    )

    debit_account = PostgresOperator(
        task_id='debit_account',
        postgres_conn_id='postgres_conn',
        sql="UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;"
    )

    create_order >> reserve_stock >> debit_account
```

#### **Phase 2 : Introduction de la Communication Asynchrone avec Kafka**

* **Objectif :** Découpler les services en les faisant communiquer par des événements (`event sourcing`).
* **Pourquoi Kafka ?** Kafka permet de gérer la communication asynchrone entre services tout en assurant la persistance des événements pour les rejouer si nécessaire.

**Exemple : Publication d’événements avec Kafka :**

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

order_event = {"order_id": 123, "status": "ORDER_PLACED"}
producer.send('order_events', value=order_event)
```

#### **Phase 3 : Gestion des Transactions Distribuées avec Temporal**

* **Objectif :** Gérer les transactions distribuées avec des compensations (`rollback`) pour garantir l’intégrité des données.
* **Pourquoi Temporal ?** Temporal permet de structurer des workflows transactionnels tout en conservant un état persistant et en implémentant des tâches compensatoires.

**Exemple : Workflow transactionnel avec Temporal :**

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

#### ✅ **Résumé des Phases :**

| **Phase**                  | **Objectif**                       | **Outil** |
| -------------------------- | ---------------------------------- | --------- |
| Externalisation des tâches | Orchestration séquentielle         | Airflow   |
| Communication asynchrone   | Gestion des événements asynchrones | Kafka     |
| Transactions distribuées   | Gestion des compensations (`SAGA`) | Temporal  |

#### ✅ **Recommandation :**

* Commencer par externaliser les composants monolithiques en tâches indépendantes orchestrées par **Airflow**.
* Introduire **Kafka** pour découpler les services et gérer les événements en temps réel.
* Utiliser **Temporal** pour implémenter des transactions distribuées avec gestion des compensations et des états.

Cette approche progressive permet de migrer vers une architecture microservices tout en minimisant les risques et en assurant la stabilité du système existant.


### ✅ Recommandations :

* Configurer `retries` et `retry_delay` pour les tâches critiques.
* Utiliser des `on_failure_callback` pour des actions compensatoires (rollback, notification).
* Mettre en place une gestion transactionnelle (`BEGIN`, `COMMIT`, `ROLLBACK`) pour garantir l'intégrité des données.
* Exploiter les opérateurs SQL (`PostgresOperator`, `MySqlOperator`, etc.) pour interagir avec plusieurs bases de données.

En résumé, Apache Airflow est une solution puissante pour le monitoring des batchs SQL grâce à ses fonctionnalités de gestion des erreurs, de replays, de logs centralisés et de gestion des transactions SQL.
