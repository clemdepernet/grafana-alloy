# TP — Déployer et monitorer un serveur Apache Tomcat avec Alloy

> Objectif : lancer **manuellement** (sans toucher au `docker-compose.yml`) un
> conteneur **Apache Tomcat**, sur le **même réseau** que le reste du lab, y
> déployer un `.WAR`, puis le **monitorer** en ajoutant une **nouvelle route**
> au pipeline Grafana **Alloy**.
>
> Prérequis : le lab du TP précédent tourne déjà (`docker compose up -d`), Alloy
> est fonctionnel (UI http://localhost:12345), Prometheus accepte le remote_write.

## Pourquoi le JMX Exporter ?

Tomcat **n'expose pas** de métriques au format Prometheus nativement. Ses
compteurs internes (requêtes, sessions, threads…) sont publiés via **JMX**
(Java Management Extensions). On greffe donc dans la JVM de Tomcat un
**javaagent** : le *JMX Prometheus Exporter*. Il lit les MBeans JMX et les
expose en HTTP sur `/metrics` (port `9404`). Alloy n'a plus qu'à le scraper.

## Architecture

### Avant
```
app, prometheus  ──> Alloy ──remote_write──> Prometheus
/var/log/*log    ──> Alloy ──push──────────> Loki
```

### Après (ajout de Tomcat)
```
app, prometheus  ──────────────┐
tomcat:9404 (JMX Exporter) ────┤──> Alloy ──remote_write──> Prometheus
/var/log/*log    ──────────────┘──> Alloy ──push──────────> Loki
```

> Tomcat tourne dans un conteneur lancé à la main, **branché sur le réseau du
> lab** : Alloy le joint par son nom DNS `tomcat`. Aucun volume partagé n'est
> nécessaire pour les **métriques** (tout passe par le réseau).

---

## Étape 0 — Prérequis & repérages

Vérifie que le lab tourne :

```bash
docker compose ps
```

Repère le **nom réel du réseau Docker**. Compose préfixe le réseau `grafana`
par le nom du projet (= nom du dossier). Le plus souvent :
`tutorial-environment_grafana`.

```bash
docker network ls | grep grafana
# ex. de sortie :  abc123  tutorial-environment_grafana  bridge  local
```

> Note la valeur exacte de la 2e colonne : on l'utilisera dans `--network`.
> Dans la suite on l'appelle `<RESEAU>`.

---

## Étape 1 — Récupérer le JMX Prometheus Exporter

Télécharge le `.jar` du javaagent dans `tomcat/jmx/` (le dossier et le
`config.yaml` de règles existent déjà dans ce repo) :

```bash
curl -L -o tomcat/jmx/jmx_prometheus_javaagent.jar \
  https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar

ls -lh tomcat/jmx/
# jmx_prometheus_javaagent.jar   +   config.yaml
```

---

## Étape 2 — Lancer Tomcat manuellement (`docker run -d`)

Depuis la racine du projet :

```bash
docker run -d \
  --name tomcat \
  --network <RESEAU> \
  -p 8888:8080 \
  -p 9404:9404 \
  -v "$PWD/tomcat/webapps:/usr/local/tomcat/webapps" \
  -v "$PWD/tomcat/jmx:/opt/jmx" \
  -e CATALINA_OPTS="-javaagent:/opt/jmx/jmx_prometheus_javaagent.jar=9404:/opt/jmx/config.yaml" \
  tomcat:9.0
```

Décortiquage de chaque option :

| Option | Rôle |
|---|---|
| `-d` | Détache (le conteneur tourne en arrière-plan). |
| `--name tomcat` | Nom du conteneur **et** alias DNS sur le réseau → Alloy joint `tomcat:9404`. |
| `--network <RESEAU>` | Branche Tomcat sur le **même réseau** que Alloy/Prometheus/Loki. |
| `-p 8888:8080` | Expose l'appli web (Tomcat écoute sur 8080 dans le conteneur). |
| `-p 9404:9404` | Expose le `/metrics` du JMX Exporter (pratique pour debug depuis l'hôte). |
| `-v .../webapps:/usr/local/tomcat/webapps` | Dossier de déploiement : on y dépose le `.WAR`. |
| `-v .../jmx:/opt/jmx` | Monte le `.jar` du javaagent + son `config.yaml`. |
| `-e CATALINA_OPTS=...` | Injecte le javaagent dans la JVM au démarrage : `=9404:<config>` ⇒ sert `/metrics` sur le port 9404. |
| `tomcat:9.0` | Image officielle Apache Tomcat 9. |

Vérifie qu'il démarre :

```bash
docker ps --filter name=tomcat
docker logs -f tomcat        # Ctrl-C pour quitter ; attends "Server startup in ... ms"
```

---

## Étape 3 — Se connecter dans le conteneur (`docker exec -it`)

```bash
# Ouvrir un shell interactif dans Tomcat
docker exec -it tomcat bash

# (dans le conteneur) explorer l'installation
ls /usr/local/tomcat/webapps      # apps déployées
ls /usr/local/tomcat/logs         # catalina.*, localhost_access_log.*.txt
cat /usr/local/tomcat/logs/catalina.$(date +%F).log | tail
ls /opt/jmx                       # le jar + config.yaml montés
exit
```

Commandes ponctuelles sans ouvrir de shell :

```bash
docker exec tomcat ls /usr/local/tomcat/webapps
docker exec tomcat tail -n 20 /usr/local/tomcat/logs/localhost_access_log.$(date +%F).txt
```

---

## Étape 4 — Déployer le `.WAR` manuellement

Deux méthodes, au choix.

**Méthode A — déposer le fichier (grâce au volume monté)** :

```bash
cp /chemin/vers/monappli.war tomcat/webapps/
# Tomcat auto-déploie : monappli.war -> http://localhost:8888/monappli/
```

**Méthode B — copier dans le conteneur (`docker cp`)** :

```bash
docker cp /chemin/vers/monappli.war tomcat:/usr/local/tomcat/webapps/
docker exec tomcat ls -l /usr/local/tomcat/webapps   # vérifier le dépôt + l'explosion du war
```

> `ROOT.war` se déploie à la racine (`http://localhost:8888/`).
> Teste l'appli :
> ```bash
> curl -I http://localhost:8888/monappli/
> ```

---

## Étape 5 — Vérifier l'export des métriques JMX

```bash
# Depuis l'hôte (port exposé 9404)
curl -s http://localhost:9404/metrics | grep -E '^tomcat_|^jvm_memory_heap' | head

# Ou depuis Alloy, par le réseau, exactement comme il scrapera
docker exec alloy wget -qO- http://tomcat:9404/metrics | head
```

Tu dois voir des séries `tomcat_requestcount_total`, `tomcat_threadpool_*`,
`tomcat_session_*`, `jvm_memory_heap_*`.

---

## Étape 6 — Ajouter la route dans Alloy

La nouvelle route est **déjà écrite** dans `alloy/config.alloy` (section 2.3) :

```alloy
prometheus.scrape "tomcat" {
  targets = [
    { __address__ = "tomcat:9404", job = "tomcat" },
  ]
  scrape_interval = "10s"
  forward_to      = [prometheus.remote_write.default.receiver]
}
```

Recharge Alloy pour prendre en compte la conf (sans modifier le compose) :

```bash
# Option 1 : recharge à chaud via SIGHUP
docker kill --signal=SIGHUP alloy

# Option 2 : redémarrage du service (ne modifie pas le fichier compose)
docker compose restart alloy
```

Contrôle dans l'UI Alloy → http://localhost:12345 → onglet **Graph** :
le composant `prometheus.scrape.tomcat` doit être **vert (healthy)**.

---

## Étape 7 — Vérifier dans Prometheus

http://localhost:9090 → requêtes PromQL :

```promql
# Le scrape fonctionne ?
up{job="tomcat"}

# Trafic HTTP cumulé sur le connecteur Tomcat
tomcat_requestcount_total

# Erreurs
tomcat_errorcount_total

# Threads occupés
tomcat_threadpool_currentthreadsbusy

# Heap utilisée
jvm_memory_heap_used
```

Toutes ces séries portent le label `job="tomcat"`. 🎉

---

## (Optionnel) Aller plus loin : les logs Tomcat

Pour ce TP, le monitoring se fait par **métriques** (réseau → aucun volume
partagé requis). Collecter aussi les **fichiers de logs** Tomcat avec Alloy
impose qu'Alloy **voie ces fichiers**, donc de partager un volume entre Tomcat
et Alloy — ce qui nécessiterait de modifier le `docker-compose.yml` (hors scope
de ce TP). Pour mémoire, la route Alloy ressemblerait à :

```alloy
// (à activer uniquement si Alloy monte le volume des logs Tomcat sur /var/log/tomcat)
local.file_match "tomcat_logs" {
  path_targets = [
    { __path__ = "/var/log/tomcat/*.log", job = "tomcat" },
    { __path__ = "/var/log/tomcat/*.txt", job = "tomcat" },  // access logs
  ]
}
loki.source.file "tomcat" {
  targets    = local.file_match.tomcat_logs.targets
  forward_to = [loki.write.default.receiver]
}
```

> En attendant, les logs restent consultables avec `docker logs tomcat` et
> `docker exec tomcat tail .../logs/...`.

---

## Nettoyage / rollback

```bash
docker rm -f tomcat                 # supprime le conteneur Tomcat
docker kill --signal=SIGHUP alloy   # (ou) retirer la section 2.3 de config.alloy puis recharger
```

Le `docker-compose.yml` n'ayant pas été touché, rien d'autre à défaire.

---

# Explication des fichiers

## `tomcat/jmx/config.yaml` (règles JMX → Prometheus)

- `lowercaseOutputName / lowercaseOutputLabelNames: true` → noms en minuscules
  (convention Prometheus).
- `rules:` → liste de motifs. Chaque `pattern` matche un **nom de MBean JMX** et
  le réécrit en métrique Prometheus :
  - **GlobalRequestProcessor** → `tomcat_requestcount_total`,
    `tomcat_errorcount_total`, `tomcat_bytessent_total`… (labels `protocol`,
    `port`), `type: COUNTER`.
  - **Manager** → `tomcat_session_*` (labels `host`, `context`) : sessions par appli.
  - **ThreadPool** → `tomcat_threadpool_*` : threads du connecteur.
  - **java.lang Memory** → `jvm_memory_heap_*` : mémoire de la JVM.

## Route Alloy `prometheus.scrape "tomcat"` (config.alloy §2.3)

- `targets = [{ __address__ = "tomcat:9404", job = "tomcat" }]` : cible le JMX
  Exporter via le DNS Docker ; `job="tomcat"` étiquette toutes les séries.
- `scrape_interval = "10s"` : fréquence de scrape.
- `forward_to = [prometheus.remote_write.default.receiver]` : réutilise le
  writer remote_write déjà en place → Prometheus stocke les métriques.

---

# Cheatsheet — Tomcat + JMX + Docker run

## `docker run` — options clés (réseau du lab)

| Flag | Effet |
|---|---|
| `--name <nom>` | Nom + alias DNS sur le réseau (résolution par les autres conteneurs). |
| `--network <RESEAU>` | Rejoindre un réseau existant (ici celui du compose). |
| `-d` | Mode détaché (arrière-plan). |
| `-p hôte:conteneur` | Publier un port vers l'hôte. |
| `-v hôte:conteneur` | Monter un dossier/volume. |
| `-e VAR=valeur` | Variable d'environnement. |
| `--rm` | Supprimer le conteneur à l'arrêt (utile pour tester). |

## `docker exec` — entrer / inspecter

```bash
docker exec -it tomcat bash         # shell interactif
docker exec tomcat <commande>       # commande ponctuelle
docker logs -f tomcat               # flux de logs (stdout = catalina)
docker cp fichier tomcat:/chemin/   # copier un fichier dans le conteneur
docker inspect tomcat --format '{{json .NetworkSettings.Networks}}'  # réseaux
```

## Arborescence Tomcat (dans le conteneur)

| Chemin | Contenu |
|---|---|
| `/usr/local/tomcat/webapps` | Apps déployées (déposer le `.WAR` ici). |
| `/usr/local/tomcat/logs` | `catalina.<date>.log`, `localhost_access_log.<date>.txt`, … |
| `/usr/local/tomcat/conf/server.xml` | Connecteurs, ports, AccessLogValve. |
| `/opt/jmx` | (notre montage) jar du javaagent + `config.yaml`. |

## JMX Prometheus Exporter

- Syntaxe javaagent : `-javaagent:<jar>=<port>:<config.yaml>`.
- Injecté via `CATALINA_OPTS` (ou `JAVA_OPTS`) au démarrage de Tomcat.
- Sert les métriques sur `http://<conteneur>:<port>/metrics`.
- Tourne **dans** la JVM → pas besoin d'ouvrir le JMX distant.

## PromQL utiles (job="tomcat")

```promql
up{job="tomcat"}                         # cible scrapée et UP ?
rate(tomcat_requestcount_total[5m])      # req/s
rate(tomcat_errorcount_total[5m])        # erreurs/s
tomcat_threadpool_currentthreadsbusy     # saturation du pool de threads
jvm_memory_heap_used / jvm_memory_heap_max   # taux d'occupation du heap
```

## Recharger Alloy après édition de `config.alloy`

```bash
docker kill --signal=SIGHUP alloy   # reload à chaud
# ou
docker compose restart alloy
```

## Pièges fréquents

- **`tomcat:9404` injoignable depuis Alloy** → mauvais `--network` : Tomcat doit
  être sur `<RESEAU>` (vérifier `docker network ls` et `docker inspect`).
- **`/metrics` vide ou 404** → `CATALINA_OPTS` mal formé, jar absent de
  `tomcat/jmx/`, ou mauvais chemin `=9404:/opt/jmx/config.yaml`.
- **WAR non déployé** → fichier pas dans `webapps`, ou extension ≠ `.war`.
- **`up{job="tomcat"} == 0`** → Alloy pas rechargé après édition de config.alloy.
- **Port 8888/9404 déjà pris** → changer le mapping hôte (`-p 8889:8080`).
