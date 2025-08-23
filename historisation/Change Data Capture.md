# Documentation : Change Data Capture (CDC)

## Sommaire

- [Introduction au CDC](#introduction-au-cdc)
- [Rôle strictement technique du CDC](#role-strictement-technique-du-cdc)
- [Ce pour quoi le CDC **n’est pas destiné**](#ce-pour-quoi-le-cdc-nest-pas-destine)
- [Limites et précautions](#limites-et-precautions)
- [Conclusion](#conclusion)

---

## Introduction au CDC

Le **Change Data Capture (CDC)** est une fonctionnalité des bases de données permettant de **capturer les modifications (INSERT, UPDATE, DELETE)** sur des tables source.

Objectifs principaux :

- Auditer les changements de données
- Alimenter des systèmes d’intégration ou de reporting (ETL, data warehouse)
- Historiser les modifications pour analyse ou conformité

⚠️ Le CDC **n’est pas un mécanisme de déclenchement de traitement métier**.

---

## Rôle strictement technique du CDC

Le CDC est conçu pour :

1. **Capturer des changements dans le journal des transactions**

   - Il lit le log transactionnel pour identifier toutes les modifications validées.

2. **Stocker ces changements dans des tables CDC**

   - Par exemple `cdc.<nom_table>_CT` dans SQL Server.
   - Ces tables contiennent les valeurs **avant/après** pour chaque modification.

3. **Servir de source pour la réplication ou l’alimentation downstream**

   - ETL, data warehouse, analytics, audit.
   - Lecture en batch périodique pour ne traiter que les données modifiées.

---

## Ce pour quoi le CDC **n’est pas destiné**

- **Ne pas déclencher de traitements métier** :

  - Facturation, notifications, workflows métiers → CDC ne doit pas être utilisé directement.

- **Ne pas filtrer par logique métier** :

  - CDC capture **tous les changements**, il ne sait pas quelles colonnes ou valeurs sont pertinentes pour un processus métier.

- **Ne pas écrire dans les tables sources suivies par CDC** :

  - Écrire dans une table CDC suivie entraîne des boucles potentielles et surcharge du système.

- **Ne pas servir de moteur temps réel métier** :

  - Latence inhérente (lecture périodique du log, tables CDC).
  - Impossibilité de cibler précisément les événements métier.

---

## Limites et précautions

1. **Volume de données** :

   - Tables CDC et logs peuvent grossir rapidement.

2. **Latence** :

   - Les données ne sont pas disponibles immédiatement, lecture en batch nécessaire.

3. **Boucles et surcharge** :

   - Tout traitement qui écrit dans une table suivie par CDC peut générer une cascade d’événements.

4. **Filtrage métier impossible en natif** :

   - Le CDC ne sait pas qu’un changement de statut doit déclencher la facturation ou un workflow spécifique.

---

## Conclusion

- Le **CDC est un outil technique** fiable pour **capturer et historiser les modifications**.
- Il est **strictement destiné** à :

  - Audit, réplication, ETL, reporting, data warehouse.

- Il **n’est pas adapté** pour :

  - Déclencher des traitements métier, workflows, notifications ou processus temps réel.

- Toute tentative d’utiliser CDC comme moteur métier **peut entraîner des boucles infinies, surcharge et incohérences**.
