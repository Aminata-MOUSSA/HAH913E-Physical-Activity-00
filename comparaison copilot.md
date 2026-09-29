# Comparaison avec le code donné par copilot

## Code de copilot

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# -----------------------------
# 1. Charger les données
# -----------------------------
df = pd.read_csv("0_z.csv")

# Colonnes : t (sec), x, y, z (g)
t = df["t"].values
x = df["x"].values
y = df["y"].values
z = df["z"].values

# -----------------------------
# 2. Calcul ENMO
# -----------------------------
# Magnitude du vecteur
mag = np.sqrt(x**2 + y**2 + z**2)

# ENMO = max(mag - 1g, 0)
enmo = np.maximum(mag - 1, 0)

df["ENMO"] = enmo

# -----------------------------
# 3. Fonction pour intégrer ENMO par epoch
# -----------------------------
def integrate_enmo(df, epoch_length):
    """
    Intègre ENMO sur des epochs de epoch_length secondes.
    Retourne un DataFrame avec :
    - epoch_start
    - enmo_integrated
    """
    results = []
    t = df["t"].values
    enmo = df["ENMO"].values

    start = t[0]
    end = t[-1]

    epoch_start = start

    while epoch_start < end:
        epoch_end = epoch_start + epoch_length

        # Sélection des points dans l’epoch
        mask = (t >= epoch_start) & (t < epoch_end)
        enmo_epoch = enmo[mask]

        if len(enmo_epoch) > 0:
            # Intégration = somme(enmo * dt)
            # dt = différence moyenne entre points
            dt = np.mean(np.diff(t[mask]))
            enmo_int = np.sum(enmo_epoch * dt)

            results.append([epoch_start, enmo_int])

        epoch_start = epoch_end

    return pd.DataFrame(results, columns=["epoch_start", "enmo_integrated"])

# -----------------------------
# 4. Calcul pour 10s, 30s, 60s
# -----------------------------
epochs = {
    "10s": integrate_enmo(df, 10),
    "30s": integrate_enmo(df, 30),
    "60s": integrate_enmo(df, 60)
}

# -----------------------------
# 5. Tracer les résultats
# -----------------------------
plt.figure(figsize=(12, 6))

for label, ep_df in epochs.items():
    plt.plot(ep_df["epoch_start"], ep_df["enmo_integrated"], label=label)

plt.title("ENMO intégré par epoch")
plt.xlabel("Temps (s)")
plt.ylabel("ENMO intégré")
plt.legend()
plt.grid(True)
plt.show()

```
## Comparaison
