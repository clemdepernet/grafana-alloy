# TP — Implémenter Grafana Alloy dans le lab

> Objectif : remplacer **Promtail** (collecte de logs) **et** le scraping interne
> de **Prometheus** (collecte de métriques) par un agent unique : **Grafana Alloy**.
> À la fin, le lab fonctionne exactement comme avant — mêmes métriques `tns_*`,
> mêmes logs `{job="varlogs"}` — mais via un seul collecteur moderne.

## Architecture

### Avant

```
app ──/metrics──> Prometheus (scrape)
app ──/var/log──> Promtail ──push──> Loki
```

### Après

```
app:80 /metrics ─┐
prometheus:9090 ─┤──> Alloy ──remote_write──> Prometheus  (stockage métriques)
/var/log/*log ───┘──> Alloy ──push────────────> Loki        (stockage logs)
```

Alloy devient le point d'entrée unique de la collecte. Prometheus ne fait plus
que **stocker** (il reçoit les métriques en `remote_write`), Loki ne change pas.

---

## Étape 0 — Prérequis

- Docker + Docker Compose installés.
- Être à la racine du projet (`tutorial-environment/`).

```bash
docker compose down      # repartir propre si le lab tournait déjà
```

---

## Étape 1 — Créer le dossier et le fichier de configuration

```bash
mkdir -p alloy
```

Crée `alloy/config.alloy` (voir le fichier déjà fourni dans ce repo). Sa
structure est détaillée dans la section **« Explication ligne par ligne »**.

---

## Étape 2 — Activer la réception remote_write sur Prometheus

Dans `docker-compose.yml`, service `prometheus`, ajoute le flag :

```yaml
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.enable-remote-write-receiver"   # <-- AJOUT
```

Sans ce flag, Prometheus refuse les `POST /api/v1/write` envoyés par Alloy.

---

## Étape 3 — Désactiver le scraping interne de Prometheus

C'est désormais Alloy qui scrape. Si on laisse les `scrape_configs`, on aura des
métriques **en double**. Dans `prometheus/prometheus.yml`, commente le bloc
`scrape_configs` (déjà fait dans ce repo).

---

## Étape 4 — Remplacer Promtail par Alloy dans docker-compose

Commente le service `promtail` et ajoute le service `alloy` :

```yaml
  alloy:
    image: grafana/alloy:latest
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy
      - app_data:/var/log            # même volume que l'app -> lecture des logs
    command:
      - "run"
      - "--server.http.listen-addr=0.0.0.0:12345"
      - "/etc/alloy/config.alloy"
    ports:
      - 12345:12345                  # UI de debug d'Alloy
    networks:
      - grafana
```

> Le volume `app_data` est la clé : l'app écrit dans `/var/log/tns-app.log`
> (cf. son Dockerfile), Alloy lit le même volume — exactement comme Promtail.

---

## Étape 5 — Démarrer le lab

```bash
docker compose up -d
docker compose ps          # tous les services doivent être "running"
```

---

## Étape 6 — Vérifications

### 6.1 L'UI Alloy

Ouvre http://localhost:12345 → onglet **Graph**. Tu vois le graphe de composants.
Chaque composant doit être **vert (healthy)**. Clique dessus pour voir ses
entrées/sorties.

### 6.2 Générer du trafic

```bash
curl -s http://localhost:8081/ > /dev/null     # génère métriques + logs
```

### 6.3 Les métriques arrivent dans Prometheus

http://localhost:9090 → requête :

```promql
tns_request_duration_seconds_count
```

Tu dois retrouver les labels `job="tns_app"` et `job="prometheus"`.

### 6.4 Les logs arrivent dans Loki

Dans Grafana (http://localhost:3000) → Explore → source Loki → requête :

```logql
{job="varlogs"}
```

Tu dois voir les logs de l'app, comme avant avec Promtail.

---

## Rollback

```bash
docker compose down
```

Puis : réactive les `scrape_configs` dans `prometheus.yml`, décommente le service
`promtail`, retire le flag remote-write et le service `alloy`. Le `git diff` te
montre tout ce qui a été touché.

---

# Explication ligne par ligne de `config.alloy`

Une configuration Alloy est écrite en **River** : un graphe de **composants**.
Un composant = `type "label" { arguments... }`. On connecte la sortie d'un
composant à l'entrée d'un autre via le champ `forward_to` et les exports
(ex : `loki.write.default.receiver`).

## Bloc LOGS

### `local.file_match "varlogs"`
Découvre les fichiers à suivre.
- `path_targets` : liste de cibles (dictionnaires de labels).
- `__path__ = "/var/log/*log"` : motif glob des fichiers (labels `__...__` =
  internes/spéciaux). Reproduit le comportement par défaut de Promtail.
- `job = "varlogs"` : label métier conservé sur chaque log → les requêtes
  `{job="varlogs"}` continuent de fonctionner.

### `loki.source.file "varlogs"`
Lit (tail) les fichiers découverts.
- `targets = local.file_match.varlogs.targets` : branche la sortie de la
  découverte en entrée.
- `forward_to = [loki.write.default.receiver]` : envoie les lignes lues vers le
  writer Loki.

### `loki.write "default"`
Écrit vers Loki.
- `endpoint { url = "http://loki:3100/loki/api/v1/push" }` : endpoint de push.
  `loki` = nom du service Docker, résolu par le DNS du réseau `grafana`.

## Bloc MÉTRIQUES

### `prometheus.scrape "tns_app"`
Scrape l'application.
- `targets = [{ __address__ = "app:80", job = "tns_app" }]` : cible + label
  `job` (le chemin par défaut scrapé est `/metrics`).
- `scrape_interval = "5s"` : reproduit l'override du `prometheus.yml`.
- `forward_to = [prometheus.remote_write.default.receiver]` : transmet les
  métriques au composant d'envoi.

### `prometheus.scrape "prometheus"`
Scrape Prometheus lui-même.
- `__address__ = "prometheus:9090"` : avant, Prometheus se scrapait via
  `localhost:9090` ; ici c'est Alloy qui le scrape → on cible le service Docker
  `prometheus`.
- `scrape_interval = "15s"` : intervalle global d'origine.

### `prometheus.remote_write "default"`
Pousse les métriques vers Prometheus.
- `endpoint { url = "http://prometheus:9090/api/v1/write" }` : endpoint
  remote-write (nécessite `--web.enable-remote-write-receiver`).

---

# Cheatsheet Grafana Alloy

## Concepts

| Terme | Définition |
|---|---|
| **Composant** | Brique unitaire : `type "label" { ... }`. |
| **River** | Le langage de config d'Alloy (proche HCL/Terraform). |
| **Argument** | Entrée d'un composant (ex : `targets`, `url`). |
| **Export** | Sortie d'un composant, réutilisable (ex : `.receiver`, `.targets`). |
| **`forward_to`** | Connecte la sortie d'un composant vers le(s) suivant(s). |
| **Pipeline** | Chaîne de composants : découverte → lecture → traitement → écriture. |
| **Référence** | `type.label.export`, ex : `loki.write.default.receiver`. |

## Familles de composants utiles

| Préfixe | Rôle | Exemples |
|---|---|---|
| `discovery.*` | Découverte dynamique de cibles | `discovery.docker`, `discovery.kubernetes` |
| `local.file_match` | Découverte de fichiers (glob) | logs sur disque |
| `prometheus.*` | Métriques | `prometheus.scrape`, `prometheus.remote_write`, `prometheus.relabel` |
| `loki.*` | Logs | `loki.source.file`, `loki.process`, `loki.relabel`, `loki.write` |
| `otelcol.*` | Pipeline OpenTelemetry | traces, metrics, logs OTLP |
| `discovery.relabel` | Réécriture de labels de cibles | filtrage / renommage |

## Labels spéciaux (commencent par `__`)

| Label | Sens |
|---|---|
| `__address__` | hôte:port de la cible à scraper |
| `__path__` | chemin/glob des fichiers de logs |
| `__metrics_path__` | chemin HTTP des métriques (défaut `/metrics`) |
| `__scheme__` | `http` / `https` |

> Les labels `__...__` sont internes (non exposés sur la donnée finale).
> Les autres clés (ex : `job`, `instance`) deviennent des labels persistés.

## Commandes

```bash
# Démarrer Alloy avec une conf
alloy run config.alloy

# Vérifier/formatter la syntaxe sans démarrer
alloy fmt config.alloy

# Avec l'UI de debug
alloy run --server.http.listen-addr=0.0.0.0:12345 config.alloy

# Via docker-compose dans ce lab
docker compose up -d alloy
docker compose logs -f alloy        # voir les erreurs de conf / runtime
docker compose restart alloy        # recharger après modif de config.alloy
```

## UI de debug (http://localhost:12345)

- **Graph** : visualise le graphe de composants et leur santé (vert = healthy).
- Clic sur un composant → ses arguments, exports, et l'état en direct.
- Indispensable pour debugger : un composant rouge = erreur à corriger.

## Endpoints du lab

| Service | URL | Usage |
|---|---|---|
| Alloy UI | http://localhost:12345 | Debug du pipeline |
| Prometheus | http://localhost:9090 | Requêtes PromQL |
| Loki (via Grafana) | http://localhost:3000 → Explore | Requêtes LogQL |
| App TNS | http://localhost:8081 | Génère métriques + logs |

## Pièges fréquents

- **Métriques en double** → oublie de commenter les `scrape_configs` de Prometheus.
- **remote_write refusé (400/404)** → flag `--web.enable-remote-write-receiver` manquant.
- **Pas de logs** → volume `app_data:/var/log` non monté sur le service `alloy`.
- **`connection refused`** → utiliser les noms de services Docker (`loki`, `prometheus`,
  `app`), pas `localhost`, car chaque conteneur a son propre réseau.
