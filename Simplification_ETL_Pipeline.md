# Simplification du Pipeline ETL

Ce document présente une approche simplifiée pour implémenter le même pipeline ETL avec moins de code complexe et une meilleure organisation des fichiers.

## Principes de simplification

1. **Modularité sans complexité excessive** - Créer des modules réutilisables sans surcharger la structure
2. **Réduction du code dupliqué** - Utiliser l'héritage et les fonctions utilitaires
3. **Configuration centralisée** - Un seul fichier de configuration pour tout le pipeline
4. **Automatisation des tâches répétitives** - Utiliser des décorateurs et des fonctions génériques

## Structure de fichiers proposée

```
jupyter_etl_pipeline/
│
├── config.json                  # Configuration centralisée
├── etl_pipeline.ipynb           # Notebook principal orchestrant le pipeline
│
├── notebooks/                   # Notebooks individuels pour chaque étape
│   ├── 1_extraction.ipynb
│   ├── 2_transformation.ipynb
│   ├── 3_schema_preparation.ipynb
│   └── 4_loading.ipynb
│
├── etl/                         # Modules Python réutilisables
│   ├── __init__.py
│   ├── utils.py                 # Fonctions utilitaires communes
│   │
│   ├── extractors/
│   │   ├── __init__.py
│   │   └── base.py              # Extracteur de base avec fonctionnalités communes
│   │
│   ├── transformers/
│   │   ├── __init__.py
│   │   └── base.py              # Transformateur de base avec fonctionnalités communes
│   │
│   ├── schema/
│   │   ├── __init__.py
│   │   └── models.py            # Définition des modèles de données
│   │
│   └── loaders/
│       ├── __init__.py
│       └── base.py              # Chargeur de base avec fonctionnalités communes
│
└── data/                        # Données d'entrée et de sortie
    ├── raw/                     # Données brutes
    ├── processed/               # Données transformées
    └── output/                  # Résultats finaux
```

## Approche simplifiée

### 1. Configuration centralisée

Utiliser un fichier de configuration unique (`config.json`) pour toutes les étapes du pipeline:

```json
{
  "input_dir": "data/raw",
  "output_dir": "data/processed",
  "database": {
    "host": "localhost",
    "port": 3306,
    "user": "root",
    "password": "",
    "database": "epiviz",
    "batch_size": 1000
  },
  "datasets": {
    "covid": ["covid_19_clean_complete.csv", "worldometer_coronavirus_daily_data.csv"],
    "monkeypox": ["owid-monkeypox-data.csv"]
  }
}
```

### 2. Module utilitaire central

Créer un module `utils.py` avec des fonctions communes:

```python
import os
import json
import pandas as pd
import pickle

def load_config(config_path='config.json'):
    """Charge la configuration à partir d'un fichier JSON"""
    try:
        with open(config_path, 'r') as f:
            config = json.load(f)
        return config
    except FileNotFoundError:
        # Configuration par défaut
        return {...}

def save_dataframes(dataframes, filename, output_dir=None):
    """Sauvegarde une liste de DataFrames"""
    config = load_config()
    output_dir = output_dir or config.get("output_dir", "data/processed")
    os.makedirs(output_dir, exist_ok=True)
    
    with open(os.path.join(output_dir, filename), 'wb') as f:
        pickle.dump(dataframes, f)

def load_dataframes(filename, input_dir=None):
    """Charge une liste de DataFrames"""
    config = load_config()
    input_dir = input_dir or config.get("output_dir", "data/processed")
    
    try:
        with open(os.path.join(input_dir, filename), 'rb') as f:
            return pickle.load(f)
    except FileNotFoundError:
        return []

def log_step(step_name):
    """Décorateur pour logger les étapes du pipeline"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            print(f"=== Début de l'étape: {step_name} ===")
            result = func(*args, **kwargs)
            print(f"=== Fin de l'étape: {step_name} ===")
            return result
        return wrapper
    return decorator
```

### 3. Classes de base avec héritage

#### Extracteur de base (`etl/extractors/base.py`):

```python
import pandas as pd
import os
from ..utils import log_step

class BaseExtractor:
    """Classe de base pour tous les extracteurs"""
    
    def __init__(self, config=None):
        """Initialise l'extracteur avec la configuration"""
        from ..utils import load_config
        self.config = config or load_config()
        self.input_dir = self.config.get("input_dir", "data/raw")
    
    @log_step("Extraction de fichier")
    def extract_file(self, file_path):
        """Extrait les données d'un fichier CSV"""
        try:
            df = pd.read_csv(os.path.join(self.input_dir, file_path))
            print(f"Extraction réussie: {file_path}, {len(df)} lignes")
            return df
        except Exception as e:
            print(f"Erreur lors de l'extraction de {file_path}: {e}")
            return pd.DataFrame()
    
    def extract_all(self, file_list=None):
        """Extrait les données de tous les fichiers spécifiés"""
        if file_list is None:
            # Utiliser tous les fichiers CSV du répertoire d'entrée
            file_list = [f for f in os.listdir(self.input_dir) if f.endswith('.csv')]
        
        dataframes = []
        for file_path in file_list:
            df = self.extract_file(file_path)
            if not df.empty:
                dataframes.append((file_path, df))
        
        return dataframes
```

#### Transformateur de base (`etl/transformers/base.py`):

```python
import pandas as pd
from ..utils import log_step

class BaseTransformer:
    """Classe de base pour tous les transformateurs"""
    
    def __init__(self, config=None):
        """Initialise le transformateur avec la configuration"""
        from ..utils import load_config
        self.config = config or load_config()
    
    @log_step("Transformation de données")
    def transform(self, dataframes):
        """Méthode abstraite à implémenter par les sous-classes"""
        raise NotImplementedError("Les sous-classes doivent implémenter cette méthode")
    
    def standardize_columns(self, df, column_mapping):
        """Standardise les noms de colonnes selon un mapping"""
        return df.rename(columns=column_mapping)
    
    def clean_numeric_columns(self, df, columns):
        """Nettoie les colonnes numériques (valeurs manquantes, etc.)"""
        for col in columns:
            if col in df.columns:
                df[col] = df[col].fillna(0).astype(int)
        return df
```

### 4. Notebooks simplifiés

#### Notebook principal (`etl_pipeline.ipynb`):

```python
# Importation des modules
import os
import sys
import json
import time
from datetime import datetime
from IPython.display import display, HTML

# Ajout du répertoire parent au path pour l'importation des modules
sys.path.append('..')

# Importation des modules ETL
from etl.utils import load_config, log_step

# Chargement de la configuration
config = load_config()

# Fonction pour exécuter un notebook
def execute_notebook(notebook_path, timeout=600):
    """Exécute un notebook Jupyter"""
    import nbformat
    from nbconvert.preprocessors import ExecutePreprocessor
    
    start_time = time.time()
    print(f"\n{'='*80}")
    print(f"Exécution de {notebook_path}")
    print(f"{'='*80}\n")
    
    try:
        with open(notebook_path, 'r', encoding='utf-8') as f:
            nb = nbformat.read(f, as_version=4)
        
        ep = ExecutePreprocessor(timeout=timeout, kernel_name='python3')
        ep.preprocess(nb, {'metadata': {'path': os.path.dirname(notebook_path)}})
        
        execution_time = time.time() - start_time
        print(f"\n{'='*80}")
        print(f"Exécution de {notebook_path} terminée en {execution_time:.2f} secondes")
        print(f"{'='*80}\n")
        
        return True
    except Exception as e:
        print(f"\nErreur lors de l'exécution de {notebook_path}: {e}")
        return False

# Liste des notebooks à exécuter
notebooks = [
    "notebooks/1_extraction.ipynb",
    "notebooks/2_transformation.ipynb",
    "notebooks/3_schema_preparation.ipynb",
    "notebooks/4_loading.ipynb"
]

# Exécution des notebooks
results = {}
start_time = time.time()

print(f"\n{'='*80}")
print(f"Démarrage du pipeline ETL à {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
print(f"{'='*80}\n")

for notebook in notebooks:
    results[notebook] = execute_notebook(notebook)

# Calcul du temps total d'exécution
total_time = time.time() - start_time

# Affichage du résumé
print("\n=== RÉSUMÉ DE L'EXÉCUTION DU PIPELINE ETL ===")
print(f"Date d'exécution: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
print(f"Temps total d'exécution: {total_time:.2f} secondes")
print(f"Étapes réussies: {sum(results.values())}/{len(notebooks)}")

# Affichage du statut de chaque étape
print("\nStatut des étapes:")
for notebook, status in results.items():
    status_text = "Succès" if status else "Échec"
    color = "green" if status else "red"
    display(HTML(f"<div style='display: flex; align-items: center;'><div style='width: 300px;'>{notebook}</div><div style='background-color: {color}; color: white; padding: 5px 10px; border-radius: 5px;'>{status_text}</div></div>"))
```

#### Notebook d'extraction (`notebooks/1_extraction.ipynb`):

```python
# Importation des modules
import os
import sys
import pandas as pd

# Ajout du répertoire parent au path pour l'importation des modules
sys.path.append('..')

# Importation des modules ETL
from etl.utils import load_config, save_dataframes
from etl.extractors.base import BaseExtractor

# Chargement de la configuration
config = load_config()

# Initialisation de l'extracteur
extractor = BaseExtractor(config)

# Extraction des données
file_list = []
for dataset_type, files in config.get("datasets", {}).items():
    file_list.extend(files)

print(f"Fichiers à extraire: {file_list}")
raw_dataframes = extractor.extract_all(file_list)

# Affichage des résultats
print(f"\nNombre de fichiers extraits: {len(raw_dataframes)}")
for file_name, df in raw_dataframes:
    print(f"  - {file_name}: {len(df)} lignes, {len(df.columns)} colonnes")

# Sauvegarde des données extraites
save_dataframes(raw_dataframes, 'raw_dataframes.pkl')
```

## Avantages de cette approche

1. **Réduction du code** - Moins de duplication grâce aux classes de base et aux utilitaires
2. **Meilleure maintenabilité** - Code plus modulaire et plus facile à comprendre
3. **Flexibilité** - Facilité d'ajout de nouveaux extracteurs, transformateurs, etc.
4. **Automatisation** - Utilisation de décorateurs pour les tâches répétitives
5. **Centralisation** - Configuration unique pour tout le pipeline

## Comparaison avec l'approche précédente

| Aspect | Approche précédente | Approche simplifiée |
|--------|---------------------|---------------------|
| Lignes de code | ~1500 | ~600 |
| Nombre de fichiers | 10+ | 8-10 |
| Duplication | Élevée | Faible |
| Extensibilité | Moyenne | Élevée |
| Maintenabilité | Moyenne | Élevée |
| Lisibilité | Moyenne | Élevée |

## Conclusion

Cette approche simplifiée permet d'obtenir les mêmes résultats avec moins de code et une meilleure organisation. Elle utilise l'héritage, les décorateurs et les fonctions utilitaires pour réduire la duplication et améliorer la maintenabilité.

Les principes clés sont:
- Centraliser la configuration
- Créer des classes de base réutilisables
- Utiliser des fonctions utilitaires communes
- Automatiser les tâches répétitives avec des décorateurs
- Organiser le code de manière logique et cohérente

Cette structure est également plus facile à étendre pour ajouter de nouvelles fonctionnalités ou prendre en charge de nouveaux types de données.
