**Partitionnement déclaratif en bases de données relationnelles**

---

# 1. Introduction au partitionnement déclaratif

Le partitionnement déclaratif est une fonctionnalité introduite à partir de PostgreSQL 10 permettant de découper logiquement une table en plusieurs partitions physiques basées sur des règles de partitionnement. Cette approche simplifie la gestion de gros volumes de données et améliore les performances.

**Objectifs principaux :**

- Répartir logiquement les données.
- Améliorer les performances d'insertion et de requêtage.
- Faciliter la maintenance des grandes tables.

---

# 2. Mécanisme de fonctionnement

## 2.1 Types de partitionnement

- **RANGE** : Par intervalles (ex : dates).
- **LIST** : Par valeurs discrètes (ex : pays).
- **HASH** : Par fonction de hachage.

## 2.2 Exemples PostgreSQL

**Table parente virtuelle :**

```sql
CREATE TABLE measurement (
    city_id int NOT NULL,
    logdate date NOT NULL,
    peaktemp int
) PARTITION BY RANGE (logdate);
```

**Partitions physiques :**

```sql
CREATE TABLE measurement_2023 PARTITION OF measurement
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
```

La table parente ne stocke pas les données mais définit le schéma et la stratégie de partitionnement.

---

# 3. Avantages du partitionnement déclaratif

- **Routage transparent** des INSERTs vers la bonne partition.
- **Héritage** des index, contraintes, triggers.
- **Optimisation** des requêtes grâce à l'élagage des partitions.
- **Performance** d'insertion équivalente à une table standard.
- **Maintenance facilitée** : archivage ou suppression de partitions entrières.

---

# 4. Limitations actuelles

- Pas de **contraintes d'unicité globale**.
- Impossibilité de **modifier la clé** de partitionnement.
- Restrictions sur les **clés étrangères** et certains **triggers**.
- Sensibilité aux modifications de structure pour le **partitionnement par hachage**.

---

# 5. Comparaison PostgreSQL - Oracle - SQL Server

| Caractéristique                    | PostgreSQL      | Oracle                | SQL Server           |
| ---------------------------------- | --------------- | --------------------- | -------------------- |
| Partitionnement RANGE, LIST, HASH  | Oui             | Oui                   | RANGE, LIST          |
| Création automatique de partitions | Non             | Oui (clause INTERVAL) | Non                  |
| Tablespaces par partition          | Non directement | Oui                   | Oui                  |
| Modification de la structure       | Difficile       | Plus flexible         | Moyennement flexible |

**Exemple Oracle :**

```sql
CREATE TABLE planning (
    event_id NUMBER GENERATED ALWAYS AS IDENTITY,
    event_date DATE NOT NULL,
    event_name VARCHAR2(100),
    event_description CLOB,
    location VARCHAR2(100),
    organizer VARCHAR2(50)
)
PARTITION BY RANGE (event_date) (
    PARTITION planning_2025 VALUES LESS THAN (TO_DATE('2026-01-01', 'YYYY-MM-DD')),
    PARTITION planning_2026 VALUES LESS THAN (TO_DATE('2027-01-01', 'YYYY-MM-DD')),
    PARTITION planning_2027 VALUES LESS THAN (TO_DATE('2028-01-01', 'YYYY-MM-DD'))
);
```

**Exemple SQL Server :**

```sql
CREATE PARTITION FUNCTION PF_PlanningDate (DATE)
AS RANGE RIGHT FOR VALUES ('2025-01-01', '2026-01-01', '2027-01-01');

CREATE PARTITION SCHEME PS_Planning
AS PARTITION PF_PlanningDate ALL TO ([PRIMARY]);

CREATE TABLE planning (
    event_id INT IDENTITY(1,1),
    event_date DATE NOT NULL,
    event_name VARCHAR(100),
    event_description TEXT,
    location VARCHAR(100),
    organizer VARCHAR(50),
    CONSTRAINT PK_Planning PRIMARY KEY (event_date, event_id)
) ON PS_Planning(event_date);
```

---

# 6. Maintenance des partitions

| Action                  | PostgreSQL                  | SQL Server                             | Oracle                      |
| ----------------------- | --------------------------- | -------------------------------------- | --------------------------- |
| Ajouter une partition   | `CREATE TABLE PARTITION OF` | `ALTER PARTITION FUNCTION SPLIT RANGE` | `ALTER TABLE ADD PARTITION` |
| Supprimer une partition | `DETACH PARTITION`          | `MERGE RANGE`                          | `DROP PARTITION`            |

**Important :** Lors de la suppression, il faut gérer l'archivage des données préexistantes si nécessaire.

---

# 7. Concepts avancés

## 7.1 GROUPING SETS

Permet de réaliser plusieurs agrégations dans une seule requête SQL, évitant plusieurs `UNION ALL`.

**Exemple :**

```sql
SELECT
    region,
    product,
    SUM(amount) AS total_sales
FROM
    sales
GROUP BY
    GROUPING SETS (
        (region, product),
        (region),
        (product),
        ()
    )
ORDER BY
    region, product;
```

**Avantages :**

- Moins de scans de la table.
- Lecture unique des données.
- Plan d'exécution optimisé.

## 7.2 Réplication logique (PostgreSQL)

- Basée sur la publication/abonnement.
- Permet de répliquer des objets spécifiques, pas l'intégralité des fichiers.
- Compatible entre différentes versions majeures.

## 7.3 Parallélisation de DISTINCT

- SQL Server et Oracle peuvent paralléliser l'opération `DISTINCT` selon la configuration (éventuellement via `MAXDOP` ou hints de parallélisme).

Exemple SQL Server :

```sql
SELECT DISTINCT col1, col2
FROM table_name
OPTION (MAXDOP 8);
```

Exemple Oracle :

```sql
ALTER SESSION FORCE PARALLEL QUERY PARALLEL 8;
SELECT /*+ PARALLEL(8) */ DISTINCT col1, col2
FROM table_name;
```

---

# 8. Notes sur la sécurité

## PostgreSQL

- Utilise `pg_hba.conf` pour gérer les accès (expressions régulières possibles).

## SQL Server & Oracle

- Pas d'équivalent direct à `pg_hba.conf`.
- Contrôles IP au niveau pare-feu ou `sqlnet.ora` pour Oracle.

---

**Conclusion :**
Le partitionnement déclaratif constitue une évolution majeure pour la gestion de très grandes tables, avec des gains de performance et de maintenance. Chaque SGBD propose ses propres mécanismes et limitations qu'il est essentiel de connaître lors de la conception d'une base partitionnée.
