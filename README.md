# ETLV3

## Description

ETLV3 est une solution ETL (Extract, Transform, Load) pour la gestion des données de pandémies développée pour l'OMS. Cette solution permet de traiter, nettoyer et charger des données sur les pandémies comme COVID-19 et Monkeypox dans une base de données structurée.

## Architecture

Le projet suit une architecture modulaire avec les composants suivants :

- **Extracteurs** : Extraction des données depuis différentes sources CSV
- **Transformateurs** : Nettoyage, agrégation et préparation des données
- **Chargeurs** : Sauvegarde des données transformées vers CSV et base de données
- **Utilitaires** : Fonctions communes et configuration

## Sources de données

- COVID-19 (covid_19_clean_complete.csv)
- Monkeypox/Variole du singe (owid-monkeypox-data.csv)
- Données quotidiennes coronavirus (worldometer_coronavirus_daily_data.csv)

## Configuration

La base de données utilisée est MySQL/MariaDB avec les paramètres suivants :

- Nom de la base de données : epiviz
- Utilisateur : root
- Mot de passe : (aucun)
- Hôte : localhost
- Port : 3306

## Installation

```bash
# Cloner le dépôt
git clone https://github.com/CYSTCloud/ETLV3.git

# Installer les dépendances
pip install -r requirements.txt
```

## Utilisation

```bash
# Exécuter le pipeline ETL complet
python etl_pipeline.py
```
