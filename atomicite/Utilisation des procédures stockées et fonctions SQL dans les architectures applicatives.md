# Documentation : Utilisation des procédures stockées et fonctions SQL dans les architectures applicatives

## Introduction

Dans le développement d’applications, la gestion de la logique métier peut s’effectuer à différents niveaux : directement dans le code applicatif, ou bien au niveau de la base de données via des procédures stockées et fonctions SQL. Ces dernières sont particulièrement utiles dans certains contextes, notamment dans les architectures monolithiques, où plusieurs applications doivent partager et centraliser une même logique métier.

Cette documentation explique pourquoi et comment utiliser les procédures stockées et fonctions SQL, illustre leur usage avec un cas concret, et met en perspective cette approche avec les architectures modernes comme les microservices.

---

## 1. Utilisation des procédures stockées et fonctions SQL : principe et avantages

### 1.1 Qu’est-ce qu’une procédure stockée et une fonction SQL ?

- **Procédure stockée** : un ensemble d’instructions SQL pré-compilées et stockées directement dans la base de données, pouvant être exécutées à la demande avec des paramètres. Elle **peut modifier l’état de la base** (insertions, mises à jour, suppressions).
- **Fonction SQL** : un bloc de code encapsulé qui retourne une valeur ou un jeu de résultats, mais **sans modifier les données** (pas d’effets de bord). Elle est donc **pure**, dans le sens de la programmation fonctionnelle.

### 1.2 Programmation fonctionnelle et base de données : pureté vs effets de bord

La programmation fonctionnelle repose sur des fonctions **pures**, c’est-à-dire des fonctions dont la sortie dépend uniquement de leurs entrées et qui **n’ont aucun effet de bord** (ne modifient pas d’état externe).

- **Fonctions SQL = fonctions pures**  
  Elles ne doivent **pas muter les données** en base. Leur rôle est de calculer, transformer ou retourner une valeur à partir des données existantes, sans changer l’état de la base.
- **Procédures stockées = fonctions impures**  
  Elles peuvent **muter les données** (insert, update, delete), gérer des transactions, et donc ont des effets de bord.

Cette séparation est essentielle pour garantir la robustesse, la prévisibilité et la maintenabilité de la logique métier. Elle permet aussi d’utiliser les fonctions dans des requêtes complexes ou des vues, en toute sécurité.

### 1.3 Pourquoi utiliser cette distinction ?

- **Sécurité et cohérence** : les fonctions pures ne modifient pas la base, donc peuvent être invoquées sans risque d’effets inattendus.
- **Optimisation** : les moteurs SQL peuvent optimiser l’exécution des fonctions pures (ex. : mise en cache, parallélisation).
- **Clarté de conception** : on sait d’avance quelles parties du code modifient la base et lesquelles se contentent de calculer des valeurs.

### 1.4 Pourquoi utiliser des procédures stockées et fonctions SQL ?

- **Centralisation de la logique métier** : regrouper les règles de gestion dans la base de données, garantissant une seule source de vérité.
- **Performance** : les procédures stockées sont pré-compilées, ce qui réduit le temps d’exécution comparé à des requêtes dynamiques.
- **Sécurité** : contrôle d’accès plus fin, possibilité de limiter les droits d’exécution sans exposer directement les tables.
- **Réduction du trafic réseau** : en exécutant des blocs logiques complexes côté base, on limite les allers-retours entre application et base.
- **Facilité de maintenance** : la modification de la logique métier dans une procédure évite la mise à jour simultanée de plusieurs applications.

---

## 2. Cas d’usage typique : architecture monolithique avec plusieurs applications

### 2.1 Contexte

Dans certaines organisations, plusieurs applications doivent accéder à la même base de données, partageant ainsi une partie importante de la logique métier. Par exemple :

- Un système d’import de données automatisé qui traite et enrichit les informations.
- Une application frontale ou un service métier démarrant certains processus sur la même base.
- Plusieurs applications internes ou externes qui doivent effectuer des opérations complexes sur les mêmes données.

### 2.2 Exemple concret

Imaginons un système où :

- **Application A** : une application web qui affiche et manipule les données métier.
- **Application B** : un service d’import automatisé chargé d’injecter des données provenant de sources externes.

Ces deux applications utilisent la **même base de données**. Pour éviter la duplication de la logique métier (règles de validation, calculs, mises à jour complexes), on développe des procédures stockées qui encapsulent cette logique.

Par exemple, une procédure `sp_ImportData` va :

- Valider les données entrantes.
- Appliquer les règles métiers spécifiques.
- Mettre à jour plusieurs tables dans une transaction sécurisée.

L’application d’import appelle simplement cette procédure. De même, l’application web utilise une fonction SQL `fn_CalculateStatus` pour afficher le statut des données.

### 2.3 Avantages dans ce contexte

- **Cohérence garantie** : les deux applications passent par la même logique, réduisant les erreurs.
- **Maintenance simplifiée** : une seule modification dans la procédure suffit.
- **Déploiement indépendant** : les applications peuvent évoluer sans impacter la logique métier stockée en base.
- **Interopérabilité** : différentes technologies (Java, .NET, Python...) peuvent interagir avec la même base en utilisant les mêmes procédures.

---

## 3. Limites et alternatives selon les architectures applicatives

### 3.1 En architecture monolithique

L’approche centralisée via procédures stockées est pertinente car :

- La base de données est un point unique de coordination.
- Les applications consommatrices sont souvent déployées ensemble ou gérées par la même équipe.
- La logique métier évolue moins fréquemment et doit être partagée uniformément.

### 3.2 En architecture microservices

Dans une architecture microservices :

- Chaque service est responsable de sa propre logique métier et souvent de sa propre base de données.
- La logique est encapsulée dans le service, généralement écrite dans le code applicatif.
- Le découplage et la flexibilité sont privilégiés.
- Le recours aux procédures stockées pour centraliser la logique métier disparaît.

### 3.3 Pourquoi plus besoin des procédures stockées dans le microservice ?

- Les services sont autonomes, donc la logique métier est décentralisée.
- Les bases de données sont spécifiques à chaque service, limitant le besoin de partager des procédures.
- La communication interservices se fait via APIs ou messages, pas via appels directs à la base.
- L’agilité et le déploiement indépendant sont optimisés, au détriment parfois d’une duplication partielle de logique.

---

## 4. Conclusion

L’utilisation de procédures stockées et fonctions SQL est un choix stratégique selon l’architecture de votre application :

| Architecture  | Usage recommandé des procédures stockées            | Avantages principaux                |
| ------------- | --------------------------------------------------- | ----------------------------------- |
| Monolithique  | Oui, centralisation de la logique métier            | Cohérence, performance, maintenance |
| Microservices | Non, logique métier décentralisée dans les services | Agilité, découplage, évolutivité    |

Pour des systèmes où plusieurs applications doivent impérativement partager une même base et logique métier, les procédures stockées restent un outil très pratique et performant. En revanche, dans les architectures modernes orientées microservices, elles deviennent inutiles, voire contre-productives.

---

## Annexes

### Exemple simple de procédure stockée

```sql
CREATE PROCEDURE sp_ImportData
  @DataId INT,
  @DataValue NVARCHAR(100)
AS
BEGIN
  -- Exemple simple de logique métier
  IF EXISTS (SELECT 1 FROM DataTable WHERE Id = @DataId)
  BEGIN
    UPDATE DataTable SET Value = @DataValue WHERE Id = @DataId;
  END
  ELSE
  BEGIN
    INSERT INTO DataTable (Id, Value) VALUES (@DataId, @DataValue);
  END
END;
```

### Exemple de fonction SQL

```sql
CREATE FUNCTION fn_CalculateStatus (@Id INT)
RETURNS NVARCHAR(50)
AS
BEGIN
  DECLARE @Status NVARCHAR(50);

  SELECT @Status = CASE
    WHEN SomeCondition THEN 'Active'
    ELSE 'Inactive'
  END
  FROM DataTable WHERE Id = @Id;

  RETURN @Status;
END;
```
