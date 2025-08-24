# 📘 Documentation – Vues matérialisées : Oracle vs SQL Server

## 1. Définition générale

Une **vue matérialisée** est une structure de base de données qui stocke physiquement le résultat d’une requête (souvent complexe ou agrégée), contrairement à une vue classique qui ne stocke que la définition SQL.

- Cela permet de **réduire les temps de réponse** pour des requêtes lourdes et répétitives.
- Mais cela implique aussi une **gestion de la fraîcheur des données** : les données de la vue matérialisée doivent être synchronisées avec les tables sources.
- Selon le moteur, les nouvelles données peuvent apparaître **immédiatement** ou avec un **décalage**.

---

## 2. Fonctionnement dans Oracle

Oracle propose un mécanisme très avancé de vues matérialisées :

- **Création** : on définit une requête (souvent avec agrégations ou jointures), et Oracle stocke son résultat.
- **Rafraîchissement** : Oracle propose deux modes principaux :

  - **Fast Refresh** : Oracle ne recharge que les modifications depuis le dernier rafraîchissement (grâce à des _Materialized View Logs_).
  - **Complete Refresh** : Oracle supprime et reconstruit entièrement la vue matérialisée.

👉 Grâce au _Fast Refresh_, Oracle peut maintenir efficacement les vues matérialisées quasi en temps réel, même sur de gros volumes de données.
⚠️ La visibilité des nouvelles données dépend du moment du rafraîchissement.

---

## 3. Fonctionnement dans SQL Server

SQL Server ne possède pas de "vues matérialisées" au sens Oracle, mais l’équivalent s’appelle une **Indexed View** (vue indexée) :

- **Création** : la vue est définie, puis matérialisée grâce à un index clusterisé unique.
- **Rafraîchissement** : contrairement à Oracle, SQL Server **rafraîchit systématiquement la vue à chaque modification des tables sources**.

  - Il n’y a **pas de Fast Refresh**.
  - Les données apparaissent normalement immédiatement, sauf si des transactions longues ou des verrous ralentissent l’actualisation.

👉 Dans SQL Server, les _Indexed Views_ accélèrent la lecture mais peuvent **dégrader fortement les performances des écritures** sur les tables sources.

---

## 4. Différences majeures Oracle vs SQL Server

| Aspect               | Oracle                                          | SQL Server                          |
| -------------------- | ----------------------------------------------- | ----------------------------------- |
| Nom                  | Materialized View                               | Indexed View                        |
| Rafraîchissement     | Fast Refresh (incrémental) ou Complete Refresh  | Toujours complet et automatique     |
| Performance écriture | Impact limité si Fast Refresh                   | Impact direct sur chaque DML        |
| Flexibilité          | Très élevé (logs, refresh programmés, partiels) | Moins flexible (toujours synchrone) |

👉 **Point clé** :
Il est illusoire de vouloir redévelopper le _Fast Refresh_ d’Oracle dans SQL Server. Cela demanderait une gestion manuelle des logs et des synchronisations complexes, et ne rattraperait pas 20 ans de R\&D Oracle.

---

## 5. Bonnes pratiques – Index et performances

### a) Index clusterisé et non clusterisé

- **Clusterisé unique** : obligatoire pour les vues indexées SQL Server, recommandé dans Oracle, pour stocker physiquement les résultats et garantir l’unicité.
- **Non clusterisé** : recommandé sur :

  - Les PK agrégées des vues matérialisées/indexées.
  - Les colonnes fréquemment utilisées pour la recherche ou les jointures.
  - Les FK utilisées dans les recherches ou jointures, pour accélérer les lectures et réduire les scans complets lors de DELETE/UPDATE.

💡 **Principe** : clusterisé = stockage et unicité, non clusterisé = rapidité pour les lectures/recherches.

---

### b) Prudence extrême sur les index de recherche

⚠️ Ajouter des index non clusterisés peut avoir **de lourdes conséquences**, surtout dans SQL Server :

1. **Latence sur les écritures** : chaque DML sur la table source met à jour tous les index. Sur des tables volumineuses, cela peut fortement ralentir INSERT/UPDATE/DELETE.
2. **Surcharge inutile** : indexer toutes les colonnes par principe alourdit les DML sans réel bénéfice.
3. **Bonnes pratiques** :

   - Indexer uniquement les colonnes réellement utilisées pour filtrage ou jointure.
   - Tester l’impact sur les performances avant le déploiement.
   - Documenter chaque index : usage, requêtes desservies, risques potentiels.

---

### c) Exemple concret (SQL Server)

- Vue indexée sur une table de ventes volumineuse.
- Ajout d’un index non clusterisé sur `CustomerID` pour accélérer les recherches par client.
- Effet secondaire : INSERT/UPDATE sur la table ventes deviennent beaucoup plus lents.
- Solution : n’ajouter l’index que si la colonne est réellement utilisée dans les rapports critiques.

---

## 6. Optimisation des jointures et recherche de données

### a) Réduire les jointures complexes

- Dans de nombreuses applications ou rapports analytiques, certaines requêtes doivent **joindre de nombreuses tables** (parfois 10 ou plus) pour produire un résultat.
- Une **vue matérialisée** permet de **pré-calculer et stocker les résultats des jointures ou des clés essentielles**, afin de :

  - réduire le **nombre de jointures nécessaires** dans chaque requête,
  - diminuer le **temps de calcul et la complexité**,
  - centraliser les données critiques pour les applications et rapports.

💡 Exemple concret :

- Une requête doit joindre 10 tables liées aux ventes, clients, produits, régions, promotions, etc.
- En créant une vue matérialisée qui **regroupe certaines clés et données essentielles**, les applications peuvent accéder directement à ces informations avec **une seule jointure** ou seulement quelques jointures.

---

### b) Optimiser les recherches de données

- Dans une application métier, les utilisateurs recherchent souvent des **dossiers, clients, commandes ou objets spécifiques** avec plusieurs filtres.
- Les vues matérialisées permettent de **pré-agréger les données et stocker les clés importantes**, ce qui :

  - réduit le **nombre de scans et jointures**,
  - accélère les recherches dans l’application,
  - améliore la **réactivité et l’expérience utilisateur**.

💡 Exemple concret :

- Retrouver tous les dossiers clients ouverts pour un type de produit donné.
- Au lieu de joindre plusieurs tables pour chaque requête, la vue matérialisée contient **les clés et colonnes nécessaires pour filtrer rapidement**.

---

### c) Pratiques recommandées

- Identifier les **jointures répétitives et coûteuses** ou les colonnes fréquemment utilisées dans les recherches.
- Créer des vues matérialisées qui **centralisent ces clés et données critiques**.
- Ajouter des **index non clusterisés** sur les PK agrégées et les FK importantes pour améliorer la vitesse de lecture.
- Limiter les colonnes stockées à celles qui sont réellement utilisées pour réduire la maintenance et le coût de rafraîchissement.

---

💡 **Conclusion** :

> Les vues matérialisées sont particulièrement efficaces pour **réduire la complexité des jointures et accélérer les recherches fréquentes** dans les applications et rapports analytiques. C’est une pratique très courante dans les environnements décisionnels et analytiques.

---

## 7. Contraintes générales des vues matérialisées / indexées

### Oracle (Materialized Views)

- **WITH clauses (CTE)** :

  - **Non supportées dans la plupart des cas** pour les vues matérialisées.
  - Les CTE (`WITH ... AS`) ne peuvent généralement pas être matérialisées directement. Il faut intégrer la requête CTE dans le `SELECT` principal ou créer une sous-requête inline.

- **Autres contraintes importantes** :

  - Les vues ne peuvent pas inclure **`DISTINCT` combiné avec certaines fonctions analytiques complexes** si Fast Refresh est utilisé.
  - Certaines fonctions comme `ROWNUM`, `SYSDATE`, ou des fonctions non déterministes rendent la vue non rafraîchissable en mode Fast Refresh.
  - Les vues matérialisées avec **joins complexes multi-niveaux** ou agrégations sur des colonnes calculées doivent parfois passer en **Complete Refresh**.

### SQL Server (Indexed Views)

- **WITH clauses (CTE)** :

  - **Pas supportées** dans une Indexed View. Les vues indexées doivent être basées sur des `SELECT` simples, avec des jointures et agrégations limitées.

- **Autres contraintes** :

  - Pas de `DISTINCT`, pas de sous-requêtes corrélées complexes dans la vue indexée.
  - Les fonctions non déterministes (`GETDATE()`, `NEWID()`, etc.) ne sont pas autorisées.
  - Les vues doivent être **SCHEMABINDING** (lié au schéma des tables sources).
  - Les agrégations doivent utiliser uniquement `COUNT_BIG`, `SUM`, `MIN`, `MAX`.

---

## 8. Règle pratique

💡 **En résumé** :

- Les vues matérialisées et indexées sont **puissantes mais rigides**.
- Elles **ne supportent pas** les constructions SQL complexes comme :

  - CTE (`WITH`),
  - sous-requêtes corrélées complexes,
  - fonctions non déterministes.

- Il faut **simplifier la requête** pour qu’elle soit compatible avec le moteur et le mode de rafraîchissement.

---

## 9. Bonnes pratiques

1. **Intégrer les CTE comme sous-requêtes inline** si possible.
2. **Éviter les fonctions non déterministes** dans les vues matérialisées ou indexées.
3. **Tester Fast Refresh / Indexed View** dès la conception pour vérifier la compatibilité.
4. **Documenter toutes les contraintes** pour les développeurs qui vont utiliser la vue.

---

## 10. Conclusion

- Les vues matérialisées sont des **outils puissants**, mais leur implémentation diffère entre Oracle et SQL Server.
- Oracle propose un **Fast Refresh** pour minimiser la latence et le coût des DML.
- SQL Server met à jour automatiquement les Indexed Views, mais **impacte les écritures**.
- **Ne jamais chercher à émuler le Fast Refresh Oracle dans SQL Server** : adaptez l’architecture à la philosophie du moteur.
- Pour de bonnes performances :

  - Index clusterisé pour la vue.
  - Index non clusterisé sur les **PK agrégées**, les **FK agrégées utilisées en recherche/jointure**, et seulement les colonnes nécessaires pour les filtres et jointures.
  - **Toujours tester l’impact** des index sur la latence et la maintenance.
