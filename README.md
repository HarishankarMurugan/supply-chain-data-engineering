# 📦 Pipeline de Données — Chaîne d'Approvisionnement & Logistique

Projet d'ingénierie des données de bout en bout : d'une base transactionnelle SQL Server jusqu'à un tableau de bord décisionnel Power BI, en passant par une extraction incrémentale SSIS, une ingestion orchestrée par Azure Data Factory, et une architecture Medallion (bronze / silver / gold) construite sur Microsoft Fabric.

> 📄 **Rapport complet** : le document [`docs/Rapport_Projet_Supply_Chain_Data_Engineering.pdf`](docs/Rapport_Projet_Supply_Chain_Data_Engineering.pdf) détaille chaque phase, chaque difficulté rencontrée et sa solution, le code complet et l'analyse des résultats. Ce README en est le résumé.

<img width="1920" height="1155" alt="image" src="https://github.com/user-attachments/assets/110788c4-86ca-4299-a8c2-cb4729802aff" />


---

## 🔎 Aperçu du projet

| | |
|---|---|
| 🏷️ **Domaine** | Chaîne d'approvisionnement et logistique |
| 📦 **Volume traité** | 180 519 lignes de commandes · 53 colonnes |
| 🗓️ **Période couverte** | 2015 — 2018 |
| 🌍 **Couverture géographique** | Mondiale |
| ✅ **Statut** | Projet terminé — pipeline validé de bout en bout |

---

## 🏗️ Architecture

```
SQL Server (SupplyChain_Staging)
        │
        │  SSIS — PKG_Extract_Orders.dtsx
        │  extraction incrémentale par filigrane + journal d'audit
        ▼
Table de staging stg.orders  (180 519 lignes)
        │
        │  Azure Data Factory — PL_02_Stage_To_Lake
        ▼
Azure Data Lake Storage Gen2 — conteneur bronze / orders.parquet (23,55 Mo)
        │
        │  Raccourci OneLake — virtualisation, aucune copie physique
        ▼
Microsoft Fabric — SupplyChain_Lakehouse
        │
        │  Notebook 01_bronze_to_silver (PySpark)
        ▼
Couche SILVER — silver_orders (table Delta, 180 519 lignes)
        │
        │  Notebook 02_silver_to_gold (PySpark)
        ▼
Couche GOLD — dim_supplier · dim_date · fact_shipment (modèle en étoile)
        │
        │  Requête inter-bases T-SQL
        ▼
SupplyChain_Warehouse — vue vw_shipment_kpis
        │
        ▼
Power BI — Tableau de bord décisionnel (DirectQuery)
```

⚙️ **Orchestration** : le pipeline natif Fabric `PL_Orchestrate_Silver_Gold` enchaîne automatiquement les deux notebooks (silver → gold) sous une seule exécution, avec dépendance conditionnelle sur succès.

---

## 🧰 Pile technologique

| Couche | Technologie |
|---|---|
| Source | Microsoft SQL Server |
| Extraction (ETL) | SQL Server Integration Services (SSIS) |
| Orchestration cloud | Azure Data Factory |
| Stockage brut | Azure Data Lake Storage Gen2 |
| Plateforme analytique | Microsoft Fabric (Lakehouse, OneLake) |
| Transformation | PySpark / Apache Spark |
| Format de table | Delta Lake |
| Entrepôt | Fabric Warehouse (T-SQL) |
| Orchestration Fabric | Fabric Data Pipeline |
| Restitution | Power BI Desktop / Service (DAX, DirectQuery) |

**Langages** : Python (PySpark), Spark SQL, T-SQL, SQL, DAX.

---

## 🗂️ Les sept phases du projet

### 1️⃣ Phase 1 — Mise en place de l'environnement
Installation de SQL Server, SSMS, Visual Studio (SSIS), création du groupe de ressources Azure, du compte de stockage ADLS Gen2, de la capacité Fabric et de l'espace de travail `SupplyChain_DE`.

### 2️⃣ Phase 2 — Base de données de staging
Base `SupplyChain_Staging` avec la table `stg.orders` (53 colonnes, 180 519 lignes) et trois tables techniques : `stg.watermark` (filigrane), `stg.audit_log` (journal d'exécution), `stg.error_log` (journal d'erreurs).

### 3️⃣ Phase 3 — Extraction incrémentale avec SSIS
Package `PKG_Extract_Orders.dtsx` : trois tâches enchaînées (Read Watermark → Audit Log → Update Watermark). Seules les lignes nouvelles depuis la dernière exécution sont extraites.

### 4️⃣ Phase 4 — Ingestion vers Azure Data Lake avec ADF
Pipeline `PL_02_Stage_To_Lake`, activité `Copy_SQLServer_To_Bronze`, connexion à la source via runtime d'intégration auto-hébergé. Écriture au format Parquet dans le conteneur `bronze` (23,55 Mo).

### 5️⃣ Phase 5 — Transformation dans Microsoft Fabric (Medallion)
- **Raccourci OneLake** vers le conteneur bronze (virtualisation, pas de copie).
- **Notebook `01_bronze_to_silver`** : profilage du schéma, transtypage de 12 colonnes (string → double), suppression de la colonne sensible `Customer_Password`, dédoublonnage, 4 contrôles de qualité automatisés, écriture en table Delta `silver_orders`.
- **Notebook `02_silver_to_gold`** : construction du modèle en étoile — `dim_supplier` (11 lignes, proxy sur Department), `dim_date` (1 127 jours, dimension calendaire continue), `fact_shipment` (180 519 lignes, avec les mesures dérivées `lead_time_days`, `delay_days`, `is_late`).
- **Entrepôt `SupplyChain_Warehouse`** : vue `vw_shipment_kpis` par requête inter-bases T-SQL.
- **Validation de bout en bout** : correspondance exacte des volumes entre la table de staging SQL Server et la table de faits finale.

### 6️⃣ Phase 6 — Orchestration et automatisation
Approche initialement prévue (ADF → API REST Fabric via principal de service) bloquée par une restriction de sécurité du locataire (erreur 401, création d'inscriptions d'applications réservée aux administrateurs). Solution retenue : pipeline natif Fabric `PL_Orchestrate_Silver_Gold`, exécuté sous l'identité de l'utilisateur, sans authentification externe.

### 7️⃣ Phase 7 — Tableau de bord Power BI
Connexion DirectQuery à l'entrepôt Fabric, modèle en étoile avec deux relations, six mesures DAX (`Ventes Totales`, `Taux de Livraison à Temps`, `Délai Moyen de Livraison`, `Total des Expéditions`, `Retard Moyen`, `Bénéfice Total`). Tableau de bord sur une page unique : 4 cartes KPI, segment temporel, graphique à barres, graphique en secteurs, carte géographique, nuage de points, graphique en ruban, graphique en courbes.

---

## 📊 Résultats clés

| Indicateur | Valeur |
|---|---|
| 💶 Ventes totales | 36,78 M€ |
| 🚚 Taux de livraison à temps | 43 % |
| ⏱️ Délai moyen d'expédition | 3,50 jours |
| 📦 Total des expéditions | 181 K |
| 🏢 Départements analysés | 11 |

💡 **Constats principaux** : le taux de service est structurellement faible (moins d'une livraison sur deux respecte le planning), avec une dégradation visible sur la fin de la période observée. Les ventes se concentrent fortement sur l'Europe de l'Ouest et l'Amérique centrale.

---

## 🛠️ Difficultés rencontrées

| Difficulté | Solution |
|---|---|
| Colonnes numériques typées en chaîne de caractères | Transtypage explicite dans le notebook silver |
| Session Spark expirée → `NameError` sur les variables | Cellule de rechargement systématique en tête de notebook |
| Colonne `Customer_Password` dans le flux analytique | Suppression dès la couche silver (minimisation des données) |
| Blocage de création de principal de service (erreur 401) | Pipeline natif Fabric à la place d'ADF Web Activity |
| Mesure DAX affichant 0 au lieu de 43 % | Correction du format d'affichage (pourcentage) |
| Absence d'entité fournisseur dans les données | Utilisation de Department_Id / Department_Name comme proxy documenté |

Le détail complet de chaque incident et de son diagnostic est dans le rapport (section 13).

---

## 📁 Contenu du dépôt

```
├── docs/                     → rapport complet (PDF/Word) et schéma d'architecture
├── screenshots/               → captures d'écran de chaque étape du pipeline
```

Les captures sont organisées par phase et illustrent : la base SQL Server et le contrôle de volumétrie, le flux de contrôle SSIS après exécution, le pipeline et les services liés ADF, les conteneurs Azure Data Lake, l'explorateur du Lakehouse Fabric, les notebooks PySpark en exécution, les contrôles de qualité, le pipeline d'orchestration Fabric, et le tableau de bord Power BI final.

---

## 🎯 Compétences mobilisées

ETL traditionnel (SSIS) · Intégration cloud (Azure Data Factory) · Stockage objet (ADLS Gen2) · Traitement distribué (PySpark) · Delta Lake · Architecture Medallion · Modélisation dimensionnelle (schéma en étoile) · SQL analytique (T-SQL) · Qualité des données · Orchestration · Power BI / DAX · Conformité et minimisation des données personnelles.

---

## ✍️ Auteur

Harishankar Murugan — Septembre 2026
