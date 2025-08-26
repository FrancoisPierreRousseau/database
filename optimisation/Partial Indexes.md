## Documentation sur les Index Partiels dans Oracle et SQL Server

### 1. Définition d'un index partiel

Un **index partiel** (ou index filtré) est un index qui ne couvre **qu'un sous-ensemble des lignes d'une table**, défini par une condition (`WHERE` clause).

**Objectif :**

- Réduire le scan complet de la table pour les requêtes ciblant ce sous-ensemble.
- Optimiser les **lectures** sur des sous-ensembles fréquemment interrogés.
- Réduire le coût des **écritures** en excluant les lignes rarement utilisées.
- Limiter l'espace disque utilisé par l'index.

**Exemple conceptuel :**

```sql
-- Index sur les dossiers non archivés
CREATE INDEX idx_non_archive
ON dossiers(id_dossier)
WHERE est_archive = 0; -- Oracle/SQL Server
```

---

### 2. Index partiel sur des sous-ensembles déséquilibrés

- Table volumineuse avec des valeurs très déséquilibrées (ex: statut A=20%, B=30%, C=40%, D=10%).
- Les index partiels sont **plus efficaces que les index complets** pour les sous-ensembles minoritaires :

  - Lecture rapide car l’index ne contient que les lignes pertinentes.
  - Moins de surcharge pour les insertions ou mises à jour hors sous-ensemble.

**Exemple sur statut rare :**

```sql
-- SQL Server
CREATE INDEX idx_statut_D
ON dossiers(id_dossier)
WHERE statut = 'D';
```

**Avantage pour les agrégations :**

- Les fonctions COUNT, SUM, GROUP BY sur un sous-ensemble minoritaire utilisent un index réduit → plus rapide.

- Même pour des sous-ensembles volumineux ou équitablement distribués, un index partiel **reste pertinent et n’est pas nuisible**. Il équivaut à un index complet mais permet d’améliorer la lecture et l’agrégation lorsqu’un sous-ensemble est disproportionnellement ciblé par des requêtes, tout en restant performant pour les écritures.

---

### 3. Performance en lecture et écriture

| Opération                         | Impact avec index partiel                                               | Remarques                                                             |
| --------------------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Lecture sur sous-ensemble ciblé   | Très rapide                                                             | L’index contient uniquement les lignes pertinentes                    |
| Lecture globale                   | Pas d’impact si les sous-ensembles non indexés sont utilisés            | Peut nécessiter scan séquentiel pour lignes hors index                |
| Insertion sur lignes hors index   | Rapide                                                                  | L’index n’est pas mis à jour                                          |
| Insertion sur lignes indexées     | Coût normal                                                             | L’index doit être mis à jour                                          |
| Update modifiant le sous-ensemble | Ligne ajoutée ou retirée de l’index                                     | Petit coût supplémentaire                                             |
| Delete                            | Ligne retirée automatiquement de l’index si elle satisfait la condition | `ON DELETE CASCADE` peut être combiné pour la cohérence référentielle |

💡 **Règle générale :**

- Gain maximal lorsque la majorité des lignes **n’appartiennent pas au sous-ensemble indexé**, mais un index partiel **reste bénéfique même pour de grands sous-ensembles**.

---

### 4. Cas spécifiques : ENUM ou colonnes ciblant une langue

- **ENUM / statut minoritaire :** Index partiel très efficace si la valeur est minoritaire mais fréquemment interrogée.
- **Champ langue :** Si les requêtes filtrent souvent sur une langue précise, un index partiel sur cette langue réduit la taille de l’index et accélère les lectures.
- **Distribution équilibrée :** L’index partiel **peut toujours être utilisé** et reste équivalent à un index complet tout en offrant des avantages pour certaines requêtes ciblées.

---

### 5. Gestion des dossiers archivés

- Table `dossiers` avec `est_archive = 0/1`.
- Créer un index partiel sur **les 10% non archivés** est souvent suffisant :

  - Lecture rapide sur dossiers actifs
  - Écriture sur dossiers archivés (90%) **non impactée par l’index**

- Créer un index partiel sur **les 90% archivés** :

  - Lecture sur dossiers archivés rapide
  - Écriture sur ces lignes implique une mise à jour de l’index → gain d’écriture réduit par rapport à l’absence d’index

- **Conclusion :** on peut créer des index partiels sur les deux sous-ensembles si les deux sont fréquemment interrogés. Cela reste performant et peut améliorer les requêtes et agrégations sur les sous-ensembles.

---

### 6. Comparaison avec l’index complet

| Critère                                | Index complet                                  | Index partiel                                                                 |
| -------------------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| Taille                                 | Grande                                         | Petite ou équivalente à un index complet selon le sous-ensemble               |
| Lecture sur sous-ensemble minoritaire  | Moins efficace                                 | Très rapide                                                                   |
| Écriture sur lignes hors sous-ensemble | Mise à jour nécessaire                         | Non impactée                                                                  |
| Maintenance                            | Plus simple mais plus coûteuse sur gros volume | Plus complexe à gérer si plusieurs index partiels, mais gain sur performances |

---

### 7. Particularités Oracle et SQL Server

#### Oracle

- Index partiel = **Index avec WHERE clause** depuis Oracle 12c (appelé `Function-Based Index` ou `Filtered Index`).
- Syntaxe exemple :

```sql
CREATE INDEX idx_non_archive
ON dossiers(id_dossier)
WHERE est_archive = 0;
```

- Oracle met à jour automatiquement l’index lors d’insertion ou modification.
- Avantages : optimisation des requêtes filtrées, moins de surcharge sur les lignes hors index.

#### SQL Server

- Appelé **Filtered Index**.
- Syntaxe exemple :

```sql
CREATE INDEX idx_non_archive
ON dbo.dossiers(id_dossier)
WHERE est_archive = 0;
```

- Permet des lectures ciblées rapides et réduit le coût des écritures sur les lignes exclues.
- Peut être combiné avec `INCLUDE` pour ajouter des colonnes non-clés à l’index.

---

### 8. Bonnes pratiques

1. **Indexer les sous-ensembles fréquemment interrogés**, même grands ou équilibrés. Un index partiel reste utile et performant.
2. **Analyser le flux d’écriture** : si une majorité de lignes est exclue, gain sur les écritures, sinon l’impact est comparable à un index complet.
3. **Combiner avec `ON DELETE CASCADE`** si l’on souhaite maintenir la cohérence référentielle sur les grappes de données.
4. **Créer plusieurs index partiels seulement si nécessaire** pour des sous-ensembles fréquemment interrogés pour limiter la maintenance.
