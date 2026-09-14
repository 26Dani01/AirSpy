# AIRSPY — Multi-Protocol Scanner

**Version 1.0**

Airspy est un projet scolaire réalisé par deux étudiants en cybersécurité à l'ESME.
Il s'agit d'outil en ligne de commande (Python) destiné à l'audit de réseaux sans fil. Il regroupe en un seul utilitaire le scan Wi-Fi, le scan Bluetooth Low Energy et la réception de signaux radio ISM (via RTL-SDR / `rtl_433`), avec un mode d'audit complet automatisé.

> ⚠️ **Usage légal uniquement.** Cet outil manipule des interfaces réseau en mode moniteur et peut envoyer des trames de désauthentification Wi-Fi. Ces opérations sont encadrées par la loi dans la plupart des pays (en France notamment, l'article 323-1 du Code pénal). N'utilisez Airspy que sur des réseaux et équipements dont vous êtes propriétaire ou pour lesquels vous disposez d'une autorisation écrite explicite (pentest contractuel, CTF, labo personnel isolé, etc.).

---

## Sommaire

- [Fonctionnalités](#fonctionnalités)
- [Architecture du projet](#architecture-du-projet)
- [Prérequis](#prérequis)
- [Installation](#installation)
- [Utilisation](#utilisation)
  - [Scan Wi-Fi](#scan-wi-fi)
  - [Attaque de désauthentification](#attaque-de-désauthentification)
  - [Scan Bluetooth](#scan-bluetooth)
  - [Scan radio (RTL-SDR / rtl_433)](#scan-radio-rtl-sdr--rtl_433)
  - [Audit complet](#audit-complet)
- [Options en ligne de commande](#options-en-ligne-de-commande)
- [Formats de sortie](#formats-de-sortie)
- [Limitations connues](#limitations-connues)
- [Avertissement légal](#avertissement-légal)

---

## Fonctionnalités

- **Scan Wi-Fi** : détection des points d'accès et des clients associés via `airodump-ng`, avec filtres par SSID, canal ou puissance de signal minimale, et résolution du fabricant (vendor) à partir de l'adresse MAC.
- **Analyse de canaux Wi-Fi** : recommandation du canal le moins encombré, à partir des réseaux détectés.
- **Attaque de désauthentification** : envoi de trames deauth via `aireplay-ng`, en mode ciblé (station précise) ou broadcast.
- **Scan Bluetooth (BLE)** : découverte asynchrone des périphériques via `bleak`, avec résolution du fabricant et tri par puissance de signal (RSSI).
- **Scan radio ISM (rtl_433)** : réception et décodage de transmissions radio (capteurs météo, télécommandes, etc.) via un récepteur RTL-SDR, en mode ponctuel ou en surveillance continue (`--live-sdr`).
- **Audit complet automatisé** : enchaîne un scan Wi-Fi, un scan Bluetooth et deux scans radio (433 MHz et 868 MHz), puis consigne l'ensemble des résultats dans un fichier `audit.txt` horodaté.
- **Interface graphique** (`AirspyGUI.py`) : une interface basée sur PySide6 / qt-material est présente dans le dépôt en complément du CLI.

## Architecture du projet

```
.
├── Airspy.py          # Point d'entrée CLI — parsing des arguments, orchestration
├── AirspyGUI.py        # Interface graphique (PySide6 / qt-material)
├── audit.py            # Mode audit complet (Wi-Fi + Bluetooth + RTL433) → audit.txt
├── bluetooth.py         # Scan Bluetooth (bleak)
├── wifi.py             # Scan Wi-Fi, analyse de canaux, attaque deauth
├── rtl.py              # Scan radio ponctuel et surveillance live (rtl_433)
├── color.py             # Constantes ANSI pour la coloration du terminal
└── requirements.txt     # Dépendances Python
```

Chaque module expose des fonctions dédiées à un protocole et partage une fonction utilitaire `get_mac_vendor()` (interrogeant l'API `api.macvendors.com`) pour résoudre le constructeur associé à une adresse MAC.

## Prérequis

### Matériel
- Une carte Wi-Fi compatible **mode moniteur** (pour le scan et le deauth).
- Un récepteur **RTL-SDR** (pour les fonctions radio via `rtl_433`).
- Un adaptateur Bluetooth compatible BLE.

### Logiciels système
- Python 3.9+ recommandé.
- `aircrack-ng` (fournit `airodump-ng` et `aireplay-ng`).
- `rtl_433` compilé avec le support SoapySDR (le code appelle `rtl_433 -d soapy ...`).
- Droits `sudo` pour les opérations nécessitant le mode moniteur (`airodump-ng`, `aireplay-ng`, `iwconfig`).
- Testé dans un contexte **Linux** (les commandes `sudo`, `iwconfig`, `wlan0mon` supposent un environnement Linux).

### Dépendances Python

Listées dans `requirements.txt` :

```
PySide6
qt-material
bleak
pyshark
pywifi
matplotlib
networkx
numpy
requests
```

> Note : `requirements.txt` liste aussi `asyncio`, `subprocess`, `json`, `time`, `csv`, `select`, `argparse`, `warnings`, `re`, qui sont des modules de la bibliothèque standard Python et n'ont pas besoin d'être installés via pip. Il est conseillé de les retirer du fichier pour éviter une erreur d'installation avec certains gestionnaires de paquets.

## Installation

```bash
# Cloner le dépôt
git clone <url-du-depot>
cd airspy

# (Optionnel mais recommandé) créer un environnement virtuel
python3 -m venv venv
source venv/bin/activate

# Installer les dépendances Python
pip install -r requirements.txt

# Installer les outils système requis (exemple Debian/Ubuntu)
sudo apt install aircrack-ng rtl-sdr
# rtl_433 avec support SoapySDR peut nécessiter une compilation depuis les sources
```

Avant tout scan ou toute attaque Wi-Fi, l'interface doit être placée en mode moniteur (nommée `wlan0mon` dans le code) :

```bash
sudo airmon-ng start wlan0
```

## Utilisation

Le point d'entrée principal est `Airspy.py`.

```bash
python3 Airspy.py [options]
```

Si aucune option n'est fournie, l'aide (`--help`) s'affiche automatiquement.

### Scan Wi-Fi

```bash
python3 Airspy.py -w -T 20
```

Options de filtrage disponibles :

```bash
python3 Airspy.py -w --filter-ssid "MonReseau" -T 15
python3 Airspy.py -w --filter-channel 1-6 -T 15
python3 Airspy.py -w --min-signal -60 -T 15
python3 Airspy.py -w --wifi-channels -T 15   # + recommandation de canal
```

Le scan repose sur `airodump-ng`, écrit un CSV temporaire (`/tmp/airodump-01.csv`), le parse, puis le supprime en fin d'exécution.

### Attaque de désauthentification

```bash
python3 Airspy.py -d -a <BSSID_cible> -T 10
python3 Airspy.py -d -a <BSSID_cible> -c <MAC_station> -T 10   # ciblage d'un client précis
```

- `-a` / `--bssid` (obligatoire) : adresse MAC du point d'accès visé.
- `-c` / `--station` (optionnel) : adresse MAC d'un client précis ; en son absence, l'attaque est envoyée en broadcast à tous les clients associés.
- Le script vérifie que l'interface `wlan0mon` est bien en mode moniteur avant de lancer `aireplay-ng` ; sinon il s'arrête avec un message d'erreur.

⚠️ **N'utilisez cette fonctionnalité que sur des équipements vous appartenant ou dans le cadre d'une mission d'audit autorisée par écrit.** Envoyer des trames de désauthentification sur un réseau tiers sans autorisation est une infraction pénale dans la plupart des juridictions.

### Scan Bluetooth

```bash
python3 Airspy.py -b -T 15
```

Scan asynchrone via `bleak.BleakScanner`, résultats triés par RSSI décroissant, avec résolution du fabricant.

### Scan radio (RTL-SDR / rtl_433)

Scan ponctuel (durée limitée par `-T`) :

```bash
python3 Airspy.py -f 433.92M -T 30 --gain auto --output json
python3 Airspy.py -f 868M --protocol 40 --output csv
```

- `-f` / `--frequency` : fréquence cible (par défaut `433.92M` si l'option est utilisée sans valeur).
- `--gain` : gain du récepteur (ex. `auto`, `40`).
- `--protocol` : filtre sur un décodeur `rtl_433` spécifique (ex. `40` pour Acurite).
- `--output` : format de sortie transmis à `rtl_433` (`json`, `csv`, `log`, `mqtt`, `influx`).

Mode surveillance continue (Ctrl+C pour arrêter) :

```bash
python3 Airspy.py -f 433.92M --live-sdr
```

`--live-sdr` nécessite obligatoirement `-f` ; sans quoi le programme affiche une erreur et s'arrête.

### Audit complet

```bash
python3 Airspy.py --audit
```

Enchaîne automatiquement :
1. Scan Wi-Fi (10 s)
2. Scan Bluetooth (10 s)
3. Scan radio à 433,92 MHz (30 s)
4. Scan radio à 868 MHz (30 s)

Les résultats sont consignés, horodatés, dans un fichier `audit.txt` créé (et réinitialisé) à la racine du projet à chaque exécution.

## Options en ligne de commande

| Option | Argument | Description |
|---|---|---|
| `-w`, `--wifi` | — | Lance un scan Wi-Fi |
| `-d`, `--deauth` | — | Lance une attaque de désauthentification (requiert `-a`) |
| `-a`, `--bssid` | MAC | BSSID cible pour le deauth |
| `-c`, `--station` | MAC | Station cible pour le deauth (optionnel) |
| `--filter-ssid` | texte | Filtre les résultats Wi-Fi par SSID |
| `--filter-channel` | ex. `1-6` | Filtre les résultats Wi-Fi par canal |
| `--min-signal` | dBm (int) | Filtre les réseaux sous ce seuil de signal |
| `--wifi-channels` | — | Analyse les canaux et recommande le moins encombré |
| `-b`, `--bluetooth` | — | Lance un scan Bluetooth |
| `-f`, `--frequency` | ex. `433.92M` | Active le scan RTL433 sur cette fréquence |
| `--gain` | ex. `auto`, `40` | Gain du récepteur SDR |
| `--protocol` | code protocole | Décodeur `rtl_433` spécifique |
| `--output` | `json`\|`csv`\|`log`\|`mqtt`\|`influx` | Format de sortie `rtl_433` |
| `--live-sdr` | — | Surveillance radio continue (requiert `-f`) |
| `-T`, `--timeout` | secondes (défaut : 10) | Durée maximale du scan ou de l'attaque |
| `--audit` | — | Lance l'audit complet automatisé |

## Formats de sortie

- **Wi-Fi / Bluetooth** : affichage direct dans le terminal, avec coloration ANSI (`color.py`) pour lisibilité.
- **RTL433** : suit le format choisi via `--output` (JSON par défaut).
- **Audit** : rapport texte structuré et horodaté, ajouté au fichier `audit.txt`.

## Limitations connues

- Le code suppose un environnement **Linux** avec les binaires `sudo`, `airodump-ng`, `aireplay-ng`, `iwconfig`, `rtl_433` disponibles dans le `PATH`.
- Le nom de l'interface moniteur (`wlan0mon`) est codé en dur dans `wifi.py` et `audit.py` ; il faudra l'adapter si votre interface porte un autre nom.
- La résolution de fabricant (`get_mac_vendor`) dépend de l'API publique `api.macvendors.com`, qui applique des limites de requêtes ; en cas d'indisponibilité ou de dépassement de quota, le fabricant est affiché comme `Unknown`.
- `Airspy.py` importe `pywifi` mais ne s'en sert pas directement dans les fonctions actuelles (le scan Wi-Fi s'appuie sur `airodump-ng`) ; l'import peut être superflu selon l'évolution du code.
- Le mode `--live-sdr` tourne indéfiniment jusqu'à interruption manuelle (Ctrl+C) et ignore le `-T` fourni.

## Avertissement légal

Airspy manipule des équipements radio et réseau et peut interférer avec des communications tierces (scan passif ou attaque active de désauthentification). L'utilisation de cet outil doit se limiter à :

- vos propres équipements et réseaux ;
- des environnements de test isolés (labo, CTF) ;
- des missions d'audit ou de pentest couvertes par une autorisation écrite explicite du propriétaire du réseau.

L'auteur/la documentation ne saurait être tenu responsable d'un usage non autorisé de cet outil. Renseignez-vous sur la réglementation applicable dans votre pays avant toute utilisation.
