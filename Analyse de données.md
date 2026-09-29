# Analyse des données

Ce document décrit les étapes de préparation des données d’accélérométrie : importation du fichier, vérification des mesures et calcul de la fréquence d’échantillonnage.

## Présentation des données

Selon l’objectif indiqué dans le notebook, les données ont été enregistrées avec un accéléromètre Axivity AX3 porté au poignet non dominant. Le fichier `0_z.csv` lui-même indique seulement que les accélérations sont en g et contient les colonnes `t`, `x`, `y` et `z`. Chaque ligne correspond à une mesure.

- `t` : le temps de la mesure, interprété en secondes dans l’analyse.
- `x`, `y` et `z` : l’accélération mesurée selon les trois axes, en g.

Après l’importation, on vérifiera que ces colonnes sont bien numériques et qu’elles ne contiennent pas de valeurs manquantes.

## 1. Importer les bibliothèques

**Objectif :** charger les outils nécessaires pour lire et analyser les données.

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
```

`pandas` sert à lire et manipuler les tableaux. `numpy` sert aux calculs numériques. `matplotlib` sera utilisé plus tard pour les graphiques.

## 2. Importer les données

**Objectif :** lire le fichier CSV et vérifier rapidement son contenu.

```python
df = pd.read_csv("0_z.csv", comment="#")
df.head()
```

Le fichier commence par une ligne de commentaire qui indique l’unité des mesures. L’option `comment="#"` demande à pandas de l’ignorer. Les données sont chargées dans `df`, un tableau appelé DataFrame. `df.head()` affiche ses cinq premières lignes.

## 3. Vérifier les données

### Étape 1 : regarder la taille et les colonnes

**Objectif :** comprendre combien de mesures contient le fichier et identifier les colonnes.

```python
print("Number of rows:", df.shape[0])
print("Number of columns:", df.shape[1])
print("Column names:", list(df.columns))
df.head()
```

`df.shape[0]` donne le nombre de lignes, `df.shape[1]` le nombre de colonnes, et `df.columns` leurs noms.

### Étape 2 : vérifier les colonnes nécessaires

**Objectif :** s’assurer que le temps (`t`) et les trois axes (`x`, `y`, `z`) sont présents.

```python
required_columns = ["t", "x", "y", "z"]
missing_columns = [column for column in required_columns if column not in df.columns]

if missing_columns:
	raise ValueError(f"Missing required columns: {missing_columns}")

print("All required columns are present:", required_columns)
```

Si une colonne manque, `raise ValueError` arrête le code et affiche un message d’erreur.

### Étape 3 : vérifier les types et les valeurs manquantes

**Objectif :** vérifier que les mesures sont numériques et qu’aucune valeur ne manque.

```python
print("Original data types:")
print(df[required_columns].dtypes)

df[required_columns] = df[required_columns].apply(pd.to_numeric, errors="coerce")
missing_values = df[required_columns].isna().sum()

print("Missing or non-numeric values:")
print(missing_values)

if missing_values.any():
	raise ValueError("Some required values are missing or non-numeric.")
```

`pd.to_numeric()` convertit les valeurs en nombres. Avec `errors="coerce"`, une valeur impossible à convertir devient manquante. `isna().sum()` compte les valeurs manquantes dans chaque colonne.

Pour ce fichier, les colonnes `t`, `x`, `y` et `z` sont toutes de type `float64`. Cela signifie que les valeurs sont des nombres à virgule, comme `0.02`, enregistrés avec une précision de 64 bits. Le contrôle trouve **0 valeur manquante** dans chacune des quatre colonnes.

### Étape 4 : vérifier l’ordre du temps

**Objectif :** vérifier que les temps sont dans l’ordre et ne sont pas répétés.

```python
time_is_increasing = df["t"].is_monotonic_increasing
duplicate_times = df["t"].duplicated().sum()

print("Time is increasing:", time_is_increasing)
print("Duplicate time values:", duplicate_times)

if not time_is_increasing or duplicate_times > 0:
	raise ValueError("The time column must increase without duplicate values.")
```

`is_monotonic_increasing` vérifie que les temps avancent dans l’ordre. `duplicated().sum()` compte les temps répétés.

## 4. Calculer la fréquence d’échantillonnage

**Objectif :** déterminer le nombre de mesures enregistrées chaque seconde et vérifier la régularité des mesures.

### Étape 1 : calculer le temps entre deux mesures

```python
time_intervals = df["t"].diff().dropna()

if time_intervals.empty:
	raise ValueError("At least two time measurements are required.")

time_intervals.head()
```

`diff()` calcule l’écart de temps entre deux mesures consécutives. `dropna()` retire le premier résultat, car il n’a pas de mesure précédente.

### Étape 2 : trouver l’intervalle typique

```python
dt = time_intervals.median()
print(f"Typical time interval: {dt:.4f} s")
```

La médiane des intervalles est stockée dans `dt`. Pour notre fichier, elle vaut **0,02 seconde**.

### Étape 3 : calculer la fréquence

```python
fs = 1 / dt
print(f"Sampling frequency: {fs:.2f} Hz")
```

La fréquence est l’inverse de l’intervalle : $fs = 1 / dt$. Notre fréquence est de **50 Hz**, soit 50 mesures par seconde.

### Étape 4 : vérifier la régularité

```python
irregular_count = int(
	(~np.isclose(time_intervals, dt, rtol=1e-3, atol=1e-9)).sum()
)
print(f"Irregular intervals: {irregular_count} / {len(time_intervals)}")
```

`np.isclose()` compare chaque intervalle à l’intervalle typique et tolère de petits écarts dus aux arrondis. Le fichier contient **0 intervalle irrégulier sur 20 999**.

À 50 Hz, les epochs de 10, 30 et 60 secondes contiennent respectivement **500, 1 500 et 3 000 mesures**.
