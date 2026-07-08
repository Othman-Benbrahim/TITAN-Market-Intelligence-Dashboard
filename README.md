# TITAN — Market Intelligence Dashboard

> **Statut : projet archivé** (juillet 2026). Fonctionnel au moment de l'archivage.
> Développé et hébergé sur alwaysdata — anciennement en ligne sur `visiondufutur.com/bourse-titan/`.

TITAN est un tableau de bord d'analyse de marchés financiers combinant collecte de données temps réel, indicateurs propriétaires, prédictions par machine learning et briefings quotidiens générés par LLM avec notification Telegram.

---

## ✨ Fonctionnalités

- **Suivi multi-actifs** : SPY, QQQ, BTC/USDT, Or, VIX, DXY, US10Y et les 11 secteurs SPDR (XLK, XLF, XLV…)
- **Indicateurs propriétaires** :
  - *Oracle* — projection de tendance pondérée
  - *Echo* — analyse fractale de similarité historique (jusqu'à 1 000 points)
  - *GEM Matrix* — lecture synthétique du régime de marché
  - *Kill Zones* — niveaux de support/résistance majeurs
- **Sentiment & macro** : analyse NLP des flux d'actualités, régime macro DXY/taux
- **Machine learning** :
  - Grid search PHP quotidien sur les poids Oracle/Echo (table `titan_cortex`)
  - Modèle XGBoost (Python) avec module d'explicabilité (XAI)
- **Briefings IA nocturnes** : rapports par actif générés via l'API Cerebras, publiés en HTML statique et poussés sur Telegram
- **Auto-Grader** : comparaison automatique prédictions vs prix réels (table `ai_predictions`)
- **Backtest** : module de test de stratégie sur l'historique
- **Vue publique** : dashboard allégé accessible sans authentification

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│  SOURCES EXTERNES                                       │
│  Binance API · Yahoo Finance v8 · Finnhub · Flux RSS    │
└──────────────────┬──────────────────────────────────────┘
                   │ cron */5 min
                   ▼
┌─────────────────────────────────────────────────────────┐
│  COLLECTE (PHP)                                         │
│  fetch_data.php → MySQL (market_data, market_history)   │
└──────────┬──────────────────────────┬───────────────────┘
           │                          │
           ▼                          ▼
┌────────────────────┐    ┌──────────────────────────────┐
│  CORTEX PHP        │    │  CORTEX PYTHON               │
│  train_ai.php      │    │  /home/othman/api_python/    │
│  (poids Oracle/    │    │  · train_xgboost.py (cron)   │
│   Echo, cron 00h)  │    │  · API /predict/{symbol}     │
└────────┬───────────┘    │  · API /generate_brief       │
         │                └──────────┬───────────────────┘
         ▼                           │
┌─────────────────────────────────────────────────────────┐
│  COUCHE API (PHP, JSON)                                 │
│  api_asset · api_history · api_orderbook · api_predict  │
│  api_smartmoney · api_macro · api_news · api_rss        │
│  api_performance · api_export · api_public              │
└──────────────────┬──────────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────────┐
│  FRONTEND (PHP + JS vanilla)                            │
│  index.php (dashboard) · public.php · backtest.php      │
│  briefing_*.html (générés par cron_cerebras.php, 05h)   │
│  → notifications Telegram                               │
└─────────────────────────────────────────────────────────┘
```

## 📦 Stack technique

| Couche | Technologie |
|---|---|
| Frontend | PHP 8, JavaScript vanilla, HTML/CSS |
| Backend / API | PHP 8 (PDO), JSON |
| ML | PHP (grid search) + Python (XGBoost, XAI) |
| LLM | API Cerebras (briefings) |
| Base de données | MySQL / MariaDB |
| Notifications | Bot Telegram |
| Hébergement | alwaysdata (mutualisé) |

## 🗄️ Base de données

8 tables MySQL :

| Table | Rôle |
|---|---|
| `market_data` | Snapshot temps réel par actif |
| `market_history` | Historique de prix (rétention 180 jours) |
| `titan_history` | Historique du stress global |
| `titan_cortex` | Poids ML Oracle/Echo par actif |
| `titan_performance` | Suivi du win rate |
| `ai_predictions` | Prédictions vs prix réels (Auto-Grader) |
| `alerts` | Alertes déclenchées |
| `titan_winrate_log` | Journal des performances |

## 🚀 Installation

### Prérequis
- Hébergement PHP ≥ 8.0 avec extensions PDO MySQL et cURL
- Base MySQL / MariaDB
- Python 3 (pour le Cortex XGBoost — optionnel, le site fonctionne en mode dégradé sans lui)
- Clé API Cerebras + bot Telegram (pour les briefings — optionnels)

### Étapes

1. **Déployer le code PHP** dans le répertoire web (contenu de `bourse-titan-production.zip`)
2. **Créer la base** et importer le dump : `mysql -u user -p ma_base < dump.sql`
3. **Configurer `db.php`** avec les identifiants de la nouvelle base
4. **Configurer `cron_cerebras.php`** : clé Cerebras, token Telegram, chat IDs, et **mettre à jour l'URL en dur** vers `api_asset.php` si le domaine a changé
5. **Mettre à jour `api_predict.php` et `api_llm.php`** si l'API Python change d'adresse
6. **Déployer l'API Python** (dossier `api_python/`) et installer ses dépendances
7. **Recréer les tâches cron** (voir ci-dessous)
8. Lancer une première collecte : `https://votre-domaine/chemin/fetch_data.php?force=1`

### Tâches cron

```cron
# Collecte des données de marché
*/5 * * * *  wget -q -O /dev/null "https://DOMAINE/bourse-titan/fetch_data.php?force=1"

# Réentraînement des poids Oracle/Echo
0 0 * * *    wget -q -O /dev/null "https://DOMAINE/bourse-titan/train_ai.php"

# Réentraînement du modèle XGBoost (Python)
0 3 * * *    python /chemin/vers/api_python/train_xgboost.py

# Briefings LLM + push Telegram
0 5 * * *    curl -s "https://DOMAINE/bourse-titan/cron_cerebras.php" > /dev/null
```

## 📁 Structure des fichiers

```
bourse-titan/
├── index.php              # Dashboard principal
├── public.php             # Vue publique
├── backtest.php           # Module de backtest
├── about.php / doc.php    # Présentation & documentation API
├── db.php                 # Connexion PDO (⚠️ identifiants)
├── api_*.php              # 11 endpoints JSON (voir architecture)
├── fetch_data.php         # Collecteur (cron 5 min)
├── train_ai.php           # Entraînement poids ML (cron 00h)
├── cron_cerebras.php      # Briefings LLM + Telegram (cron 05h, ⚠️ clés API)
├── briefing_*.html        # 5 briefings générés (SPY, QQQ, BTC, GOLD, VIX)
└── last_update.txt        # Verrou anti-flood
```

## ⚠️ Sécurité — points connus

- Les identifiants MySQL (`db.php`) et les clés API Cerebras/Telegram (`cron_cerebras.php`) sont **en clair dans le code**. Avant toute remise en ligne : régénérer tous les secrets et les externaliser (fichier de config hors racine web ou variables d'environnement).
- Protéger `fetch_data.php`, `train_ai.php` et `cron_cerebras.php` par un token secret ou une règle `.htaccess`.
- Réactiver la vérification SSL (`CURLOPT_SSL_VERIFYPEER`) dans `api_predict.php` et `cron_cerebras.php`.
- Ne jamais redéployer les scripts de maintenance (`reset.php`, `fix_db.php`, `clean.php`, `debug.php`, tests, seeds) sur un serveur public.

## ⚖️ Avertissement

TITAN est un projet expérimental d'apprentissage et d'exploration technique. Ses indicateurs et prédictions **ne constituent en aucun cas des conseils en investissement** et ne doivent pas servir de base à des décisions de trading réelles.

## 📚 Kit d'archivage

Le projet complet est préservé en 4 pièces :

1. `bourse-titan-production.zip` — code PHP (26 fichiers)
2. `dump.sql` — structure et données de la base
3. `crontab-titan.txt` — tâches planifiées
4. `api_python/` — Cortex Python (modèles + API), récupéré par SFTP
