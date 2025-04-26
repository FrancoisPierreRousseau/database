# Utilisation Optimisée des Tables Temporaires en Bases de Données

---

# 1. Introduction aux Tables Temporaires

### ⚠️ Attention : Toutes les tables temporaires ne sont pas égales

- Les tables temporaires sont parfaites pour stocker des données transitoires pendant l'exécution d'une requête ou d'une procédure.
- Mal utilisées, elles peuvent devenir un goulot d'étranglement (I/O disque, verrouillages inutiles...).
- Les bases modernes (PostgreSQL, SQL Server) offrent des solutions pour **accélérer** ces structures, notamment via l'**optimisation mémoire**.

---

### ✅ Mini checklist : "Dois-je optimiser ma table temporaire ?"

> Réponds rapidement : si tu coches plusieurs cases, l'optimisation devient prioritaire.

- Ma table temporaire est créée plusieurs milliers de fois par heure ?
- Elle contient moins de quelques Mo de données ?
- Son accès est critique pour la performance globale de l’application ?
- J’observe beaucoup d'I/O disque sur tempdb ou pg_temp ?
- J’utilise déjà des structures simples sans contraintes complexes (index, FK...) ?

➔ Si oui à 3 ou plus ➔ pense à utiliser l'optimisation mémoire !

---

# 2. Concepts Clés

## 2.1 Tables Temporaires vs Variables de Table

- **SQL Server** distingue `#TempTable` (créées dans tempdb) et `@TableVariable` (stockées en mémoire).
- **PostgreSQL** utilise les **tables temporaires** (`CREATE TEMP TABLE`) stockées sur disque par défaut, mais propose des astuces pour éviter ce coût (ex: `UNLOGGED`, CTEs, stockage en RAM).

**Problème commun** : même si les données sont temporaires, elles subissent les mêmes contraintes d'I/O et de journalisation que les tables permanentes !

---

## 2.2 Accélération avec l'Optimisation Mémoire

### SQL Server : Tables Temporaires Optimisées

- Depuis SQL Server 2019, on peut créer des tables temporaires **optimisées en mémoire**.
- Principe : utilisation de **In-Memory OLTP** ➔ accès ultra-rapide.
- Syntaxe type :

```sql
CREATE TYPE dbo.MyMemoryOptimizedTableType AS TABLE
(
    ID INT PRIMARY KEY NONCLUSTERED,
    Value NVARCHAR(100)
) WITH (MEMORY_OPTIMIZED = ON);
```

- Ensuite, déclaration :

```sql
DECLARE @MyTempTable dbo.MyMemoryOptimizedTableType;
```

**Avantages** :

- Pas d'écriture sur disque.
- Moins de contention de verrouillage.
- Très rapide pour les petits jeux de données.

---

### PostgreSQL : Astuces d'Optimisation

- **Tables `UNLOGGED`** : pas de WAL (Write Ahead Log) ➔ accélération.
- **Tables locales en CTE** (`WITH`) : pour stocker temporairement sans accès disque.
- **Utiliser des séquences de requêtes plutôt que des tables** : parfois, une série de `WITH` et `SELECT` bien agencés suffit.

Exemple :

```sql
WITH temp_data AS (
    SELECT generate_series(1, 1000) AS id
)
SELECT * FROM temp_data WHERE id % 2 = 0;
```

---

# 3. Bonnes Pratiques

| Situation                      | Recommandation                                                                    |
| :----------------------------- | :-------------------------------------------------------------------------------- |
| Petits volumes (< 100k lignes) | Memory-Optimized Tables (SQL Server) ou UNLOGGED/CTE (PostgreSQL)                 |
| Gros volumes                   | Table Temporaire standard avec index adapté                                       |
| Usage en boucle (cursor/loop)  | Privilégier la mémoire pour éviter la surcharge disque                            |
| Transactions longues           | Attention : les structures "memory" ne supportent pas toutes les contraintes ACID |

---

# 4. Limites et Pièges

### SQL Server

- In-Memory = besoin de mémoire RAM disponible !
- Types mémoire doivent être créés à l'avance (`CREATE TYPE`).

### PostgreSQL

- `UNLOGGED` ➔ perte de données en cas de crash.
- Tables `TEMP` sont détruites en fin de session, attention au scope.

---

# 5. Conclusion

Optimiser l'usage des tables temporaires, c’est **réduire la latence** de vos requêtes critiques.  
Grâce aux innovations sur **In-Memory OLTP** (SQL Server) et aux bonnes pratiques (PostgreSQL), vos traitements intermédiaires peuvent passer de plusieurs secondes à quelques millisecondes.

**Règle d'or** :

> ➔ Si tu utilises beaucoup de tables temporaires ➔ pense **RAM** avant **disque** !
