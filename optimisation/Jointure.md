# 📘 **Documentation : Optimisation SQL avec INNER JOIN et LEFT JOIN**

## 🔍 **1. Introduction aux jointures SQL**

Les jointures permettent de combiner des lignes de plusieurs tables selon une relation logique définie par des clés. Les deux plus courantes :

* **INNER JOIN** : retourne les lignes où une correspondance existe dans les deux tables.
* **LEFT JOIN** : retourne toutes les lignes de la table de gauche, même sans correspondance dans la table de droite (remplie par des `NULL`).

---

## ⚙️ **2. Fonctionnement et différences entre INNER JOIN et LEFT JOIN**

| Type de jointure | Comportement                                    | Performance attendue      | Risques                            |
| ---------------- | ----------------------------------------------- | ------------------------- | ---------------------------------- |
| INNER JOIN       | Ne conserve que les lignes avec correspondance. | Très rapide avec index    | Lent sans index, ou trop de tables |
| LEFT JOIN        | Conserve toutes les lignes de gauche.           | Correct avec un bon index | Risque de volumétrie excessive     |

### 📌 Exemples

#### En SQL Server

```sql
-- INNER JOIN : Clients avec commandes
SELECT c.Name, o.OrderID
FROM Customers c
INNER JOIN Orders o ON c.CustomerID = o.CustomerID;
```

```sql
-- LEFT JOIN : Tous les clients, même sans commande
SELECT c.Name, o.OrderID
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID;
```

#### En Oracle

```sql
-- INNER JOIN Oracle
SELECT c.Name, o.OrderID
FROM Customers c
JOIN Orders o ON c.CustomerID = o.CustomerID;
```

```sql
-- LEFT JOIN Oracle
SELECT c.Name, o.OrderID
FROM Customers c
LEFT OUTER JOIN Orders o ON c.CustomerID = o.CustomerID;
```

---

## ⚡ **3. Facteurs clés de performance des jointures**

### 📁 a. Indexation

* Les colonnes de jointure **doivent être indexées**.
* Les champs `ENUM` (stables, valeurs fixes) sont **excellents candidats** à l’indexation.

```sql
-- SQL Server : Création d'index
CREATE INDEX idx_customer_id ON Orders (CustomerID);
```

```sql
-- Oracle : Création d'index
CREATE INDEX orders_customer_idx ON Orders(CustomerID);
```

### 📏 b. Volumétrie des tables

* Privilégier les jointures sur **petits ensembles** ou **tables filtrées** en premier.
* Une **jointure LEFT** sur une grosse table non filtrée peut ralentir drastiquement la requête.

---

## 🧠 **4. Ordre d’exécution et plan d’optimisation**

* SQL exécute d’abord `FROM`, puis les `JOIN`, avant le `WHERE`.
* L’ordre des jointures **impacte** le plan d'exécution.

### 🔄 Bonne pratique :

Toujours commencer par des `INNER JOIN` sur des tables filtrées, pour **réduire la volumétrie dès le début**.

```sql
-- Exemple SQL Server optimisé
SELECT *
FROM Orders o
INNER JOIN Customers c ON o.CustomerID = c.CustomerID
WHERE c.Country = 'France';
```

---

## 📈 **5. Effet de cascade et jointures multiples**

### 🌪️ Problème : LEFT JOIN dès le début

Peut générer un énorme jeu de résultats intermédiaire.

### ✅ Solution : INNER JOIN initial

Permet de **réduire les données** avant d’ajouter d'autres jointures.

#### Exemple :

```sql
-- Mauvais ordre
SELECT *
FROM Orders o
LEFT JOIN Orders o ON o.CustomerID = c.CustomerID
INNER JOIN Customers c ON o.CustomerID = c.CustomerID 
WHERE c.Country = 'France';

-- Bon ordre
SELECT *
FROM Orders o
INNER JOIN Customers c ON o.CustomerID = c.CustomerID 
LEFT JOIN Orders o ON o.CustomerID = c.CustomerID
WHERE c.Country = 'France';
```

---

## 🛠️ **6. Techniques avancées d’optimisation**

### 🎯 a. Filtrage contextuel précoce

Utiliser des **sous-requêtes** ou des **CTE (Common Table Expressions)** pour filtrer avant les jointures.

```sql
-- SQL Server
WITH ClientsFrance AS (
    SELECT * FROM Customers WHERE Country = 'France'
)
SELECT cf.Name, o.OrderID
FROM ClientsFrance cf
JOIN Orders o ON cf.CustomerID = o.CustomerID;
```

### 📂 b. Pré-agrégation avant la jointure

```sql
-- Oracle
SELECT d.Region, SUM(s.Sales)
FROM (
  SELECT CustomerID, SUM(Sales) AS Sales
  FROM Orders
  GROUP BY CustomerID
) s
JOIN Customers d ON s.CustomerID = d.CustomerID
GROUP BY d.Region;
```

---

### ⚠️ c. Éviter les sous-requêtes corrélées

Les sous-requêtes corrélées (exécutées pour chaque ligne de la requête principale) sont **fortement déconseillées** dans les contextes d'optimisation, car elles peuvent :

* Générer des boucles d'exécution (`N x M`) très coûteuses,
* Empêcher le moteur SQL d'utiliser efficacement les index,
* Être remplacées par des jointures bien plus performantes.

#### ❌ Exemple à éviter :

```sql
SELECT c.Name,
       (SELECT COUNT(*) FROM Orders o WHERE o.CustomerID = c.CustomerID) AS OrderCount
FROM Customers c;
```

#### ✅ Recommandé avec jointure :

```sql
SELECT c.Name, COUNT(o.OrderID) AS OrderCount
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
GROUP BY c.Name;
```

✔️ Cette version permet une **exécution en une seule passe**, **exploitant les index** et **réduisant le coût total** de la requête, surtout sur de gros volumes.

---

### 🧩 d. Sous-requêtes dans les `JOIN` (`INNER`, `LEFT`, etc.)

Les sous-requêtes utilisées dans une clause de `JOIN` sont appelées **tables dérivées**. Elles sont généralement **exécutées une seule fois**, et peuvent être **très performantes** si elles sont bien construites.
Mais leur efficacité dépend de plusieurs facteurs : **structure interne, corrélation, indexation et volumétrie**.

---

#### ✅ Cas recommandés : sous-requête dérivée **autonome et optimisable**

```sql
SELECT c.Name, o.TotalSales
FROM Customers c
LEFT JOIN (
    SELECT CustomerID, SUM(Sales) AS TotalSales
    FROM Orders
    GROUP BY CustomerID
) o ON o.CustomerID = c.CustomerID;
```

✔️ Cette sous-requête est :

* **Indépendante** (aucune dépendance avec la table `Customers`)
* Facilement **optimisable** par le moteur SQL
* Compatible avec des **index sur `Orders.CustomerID`**

Elle est exécutée une seule fois, peut être **matérialisée**, et offre un plan efficace.

---

#### ⚠️ Sous-requêtes contenant des `EXISTS` ou des `JOIN` internes

Tu peux utiliser des `JOIN` ou `EXISTS` à l’intérieur d’une sous-requête joinée, **à condition qu’ils ne soient pas corrélés à l’extérieur**.

```sql
SELECT c.Name
FROM Customers c
LEFT JOIN (
    SELECT DISTINCT o.CustomerID
    FROM Orders o
    WHERE EXISTS (
        SELECT 1 FROM Audit a WHERE a.OrderID = o.ID
    )
) filtered_orders ON filtered_orders.CustomerID = c.CustomerID;
```

✅ Ici :

* La sous-requête avec `EXISTS` est **indépendante de `Customers`**
* Elle peut bénéficier d’index sur `Orders.ID` et `Audit.OrderID`
* Le moteur peut optimiser avec des **semi-joins**

---

#### ❌ Cas à éviter : sous-requête joinée **corrélée**

```sql
SELECT *
FROM Customers c
JOIN (
    SELECT * FROM Orders o
    WHERE EXISTS (
        SELECT 1
        FROM Audit a
        WHERE a.OrderID = o.ID AND a.Region = c.Region  -- ❌ dépendance à la table principale
    )
) x ON x.CustomerID = c.CustomerID;
```

⛔ Cette sous-requête dépend de `c.Region`, ce qui la rend **corrélée**.
Résultat :

* Elle est exécutée **une fois par ligne de `Customers`**
* Le moteur **ne peut pas l’optimiser comme une table dérivée**
* Le plan d’exécution devient **très coûteux**

---

#### 📌 Conclusion :

✔️ Utilise des sous-requêtes dans les `JOIN` quand :

* Elles sont **non corrélées à la requête principale**
* Elles sont **filtrées ou agrégées efficacement**
* Elles bénéficient d’**index adaptés**

❌ Évite toute **corrélation implicite avec la requête extérieure**, surtout dans des jointures complexes.

🛠️ Toujours vérifier le **plan d’exécution (`EXPLAIN`, `SHOWPLAN`)** pour confirmer que le moteur SQL les traite comme prévu.


#### ⚠️ Attention : les sous-requêtes dans les `JOIN` ne sont pas toujours bénéfiques

Bien qu'elles soient lisibles et structurantes, leur impact sur les performances dépend de :

* La volumétrie de la sous-requête
* La présence ou non d’**index** sur les colonnes concernées
* L'absence de **sous-requêtes corrélées imbriquées**, qui peuvent fortement pénaliser le plan d'exécution

#### ❌ Exemple inefficace :

```sql
SELECT *
FROM Customers c
JOIN (
    SELECT * FROM Orders WHERE Status = 'actif' AND EXISTS (
        SELECT 1 FROM Audit WHERE Audit.OrderID = Orders.ID
    )
) o ON o.CustomerID = c.CustomerID;
```

➡️ Cette sous-requête contient une **corrélation interne**, rendant l’optimisation difficile.

📌 **Conclusion** :
Utilise les sous-requêtes dans les `JOIN` lorsque :

* Elles sont **autonomes** (non corrélées)
* Elles sont **filtrées et agrégées intelligemment**
* Elles s’appuient sur des **index efficaces**

Et comme toujours : **vérifie leur impact avec `EXPLAIN` ou `SHOWPLAN`**.

---

## 🔍 **7. Utilisation des indexes sur champs ENUM ou faibles cardinalités**

### Avantages :

* Recherche rapide
* Filtrage précoce
* Faible fragmentation

```sql
-- SQL Server
CREATE TABLE Accounts (
    ID INT PRIMARY KEY,
    Status VARCHAR(20) NOT NULL CHECK (Status IN ('actif', 'inactif', 'suspendu'))
);
CREATE INDEX idx_status ON Accounts(Status);
```

---

## 🧪 **8. Vérification et analyse du plan d'exécution**

* Utilisez `EXPLAIN` (Oracle) ou `SET SHOWPLAN_ALL ON` (SQL Server)
* Analysez les étapes de jointure, les coûts estimés, les "Nested Loop" vs "Hash Match", etc.

```sql
-- SQL Server
SET STATISTICS PROFILE ON;

SELECT ...
FROM A
JOIN B ON ...
```

```sql
-- Oracle
EXPLAIN PLAN FOR
SELECT ...
FROM A
JOIN B ON ...;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

---

## 📋 **9. Récapitulatif des bonnes pratiques**

| Action                                              | Bénéfice                             |
| --------------------------------------------------- | ------------------------------------ |
| Utiliser `INNER JOIN` en début de requête           | Réduction du volume de données       |
| Indexer toutes les colonnes utilisées dans les `ON` | Accélération des correspondances     |
| Filtrer dans des sous-requêtes ou CTE               | Réduction des lectures               |
| Analyser le plan d’exécution                        | Détection des goulets d’étranglement |
| Pré-agréger les données avant jointure              | Moins de lignes à traiter            |
| Éviter les jointures inutiles                       | Allègement des ressources            |

---

## 📚 **10. Conclusion**

Le choix entre `INNER JOIN` et `LEFT JOIN` ne doit jamais être arbitraire. Il dépend du **besoin fonctionnel**, mais aussi de la **volumétrie** et de la **structure des données**.

Une requête bien écrite peut réduire les temps d'exécution de plusieurs minutes à quelques secondes. À l’inverse, une mauvaise utilisation des jointures peut gravement compromettre les performances, notamment en production.
