---

# 7. Concepts Avancés

## 7.1 GROUPING SETS

Les **GROUPING SETS** permettent de réaliser plusieurs agrégations dans une seule requête SQL, optimisant ainsi les performances en évitant des appels multiples avec `UNION ALL`.

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

**Avantages** :

- Réduit le nombre de lectures nécessaires.
- Améliore l'optimisation du plan d'exécution.

## 7.2 Réplication Logique (PostgreSQL)

La réplication logique permet de répliquer des objets spécifiques (tables, vues, etc.) plutôt que des fichiers entiers, et offre une compatibilité entre différentes versions majeures.

## 7.3 Parallélisation des Requêtes DISTINCT

Les SGBD modernes comme SQL Server et Oracle peuvent paralléliser certaines requêtes, telles que celles utilisant `DISTINCT`, pour améliorer les performances sur de grandes quantités de données.

---

# 8. Notes sur la Sécurité

- **PostgreSQL** utilise `pg_hba.conf` pour gérer les accès, avec des possibilités de filtrage par adresse IP et expression régulière.
- **SQL Server et Oracle** n'ont pas d'équivalent direct à `pg_hba.conf`, mais gèrent la sécurité au niveau du pare-feu et des configurations spécifiques à chaque SGBD (par exemple, `sqlnet.ora` pour Oracle).

---
