# Créer des dashboards Grafana

Grafana est connecté à Prometheus. Il est maintenant temps de construire des dashboards pour visualiser les métriques de votre serveur Ubuntu et de votre application NodeJS.

## Comprendre les concepts

| Concept | Description |
|---------|-------------|
| **Dashboard** | Ensemble de panels regroupés sur une page |
| **Panel** | Widget individuel (graphique, jauge, stat…) |
| **Query** | Requête PromQL envoyée à Prometheus |
| **Time range** | Fenêtre temporelle affichée (dernières 1h, 24h…) |
| **Refresh** | Intervalle de rafraîchissement automatique |

## Importer un dashboard communautaire

Avant de créer un dashboard from scratch, la bonne pratique est de chercher si un dashboard existe déjà sur **Grafana Labs**. La communauté publie des centaines de dashboards prêts à l'emploi.

### Importer le dashboard Node Exporter Full

C'est le dashboard le plus utilisé pour monitorer un serveur Linux avec Node Exporter.

1. Dans le menu de gauche :  
   👉 **Dashboards → Import**

2. Dans le champ **"Import via grafana.com"**, entrez l'ID :  
   ```
   1860
   ```

3. Cliquez sur **Load**

4. Dans le champ **Prometheus**, sélectionnez votre datasource `Prometheus`

5. Cliquez sur **Import**

Vous avez maintenant un dashboard complet avec CPU, RAM, disque et réseau de votre serveur Ubuntu.

👉 Explorez les différents panels : passez votre souris dessus, regardez les requêtes PromQL utilisées en cliquant sur les trois points d'un panel → **Edit**.

## Créer un dashboard depuis zéro

### 1. Créer un nouveau dashboard

Dans le menu de gauche :  
👉 **Dashboards → New → New dashboard → Add visualization**

Sélectionnez la datasource `Prometheus`.

### 2. Anatomie de l'éditeur de panel

```
┌─────────────────────────────────────────────────────┐
│  Visualisation (graphique, jauge, stat, table…)     │
├─────────────────────────────────────────────────────┤
│  Query Editor                                       │
│  [A]  PromQL : up                                   │
├─────────────────────────────────────────────────────┤
│  Options du panel (titre, unité, légendes…)         │
└─────────────────────────────────────────────────────┘
```

- **Zone de visualisation** : aperçu en temps réel
- **Query Editor** : saisie des requêtes PromQL
- **Panel options** (colonne droite) : titre, description, unités, couleurs

### 3. Les types de visualisation

| Type | Usage |
|------|-------|
| **Time series** | Courbes temporelles (CPU, requêtes/s…) |
| **Stat** | Valeur unique mise en avant (uptime, version…) |
| **Gauge** | Jauge circulaire (% RAM, % disque…) |
| **Bar chart** | Comparaison entre séries |
| **Table** | Données tabulaires |
| **Heatmap** | Distribution de valeurs dans le temps |

## Exercice : Dashboard de monitoring complet

Créez un dashboard nommé **"Monitoring complet"** contenant les panels suivants.

### Panel 1 — Statut des instances

**Type de visualisation :** Stat

**Requête PromQL :**
```promql
up
```

**Réglages :**
- Title : `Instances actives`
- Dans *Value options*, choisissez **Last** (dernière valeur)
- Dans *Thresholds* : rouge si `0`, vert si `1`

**Réponse — Quel résultat observez-vous ?**

    Les 4 cibles scrapées par Prometheus retournent `1` (UP) :
    - `prometheus:9090` (job prometheus)
    - `alertmanager:9093` (job alertmanager)
    - `85.217.163.213:9100` (job node-exporter)
    - `85.217.163.213:3000` (job node-app)
    Tous les panels affichent donc une valeur verte « 1 ». Si un service tombe, la stat passe à 0 et le seuil rouge se déclenche.


### Panel 2 — Utilisation CPU

**Type de visualisation :** Time series

**Requête PromQL :**
```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Réglages :**
- Title : `CPU Usage (%)`
- Dans *Standard options*, Unit : `Percent (0-100)`

**Réponse — Quel est le pourcentage de CPU moyen observé ?**

    Environ **0,7 %** sur `srv-web-110` (la seule instance exposant node_exporter).
    La VM est très peu chargée — les pics correspondent aux bursts de trafic curl sur l'app NodeJS et au scrape Prometheus (toutes les 15 s).


### Panel 3 — Mémoire disponible

**Type de visualisation :** Gauge

**Requête PromQL :**
```promql
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

**Réglages :**
- Title : `RAM disponible (%)`
- Unit : `Percent (0-100)`
- Min : `0`, Max : `100`
- Thresholds : rouge < 10%, orange < 30%, vert sinon

**Réponse — Quel est le pourcentage de RAM disponible ?**

    Environ **76,4 %** de RAM disponible sur `srv-web-110`.
    La jauge est largement dans la zone verte (seuils orange < 30 %, rouge < 10 %).


### Panel 4 — Espace disque utilisé

**Type de visualisation :** Gauge

**Requête PromQL :**
```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)
```

**Réglages :**
- Title : `Disque utilisé (%)`
- Unit : `Percent (0-100)`

**Réponse — Quel est le pourcentage d'espace disque utilisé ?**

    Environ **32,2 %** d'espace disque utilisé sur la partition `/` de `srv-web-110`.
    Largement sous les seuils d'alerte — la VM dispose donc encore d'environ 68 % d'espace libre.


### Panel 5 — Requêtes HTTP par seconde (app NodeJS)

**Type de visualisation :** Time series

**Requête PromQL :**
```promql
sum by(route) (rate(http_requests_total[1m]))
```

**Réglages :**
- Title : `Requêtes HTTP/s par route`
- Unit : `requests/sec`

**Réponse — Quelles routes génèrent le plus de trafic ?**

    Lorsque le script de génération de trafic tourne, les routes `/`, `/health`, `/users`, `/notfound` et `/orders` reçoivent le même volume (~0,4 req/s chacune avec 100 itérations) car le générateur les appelle de manière équivalente.
    La route `/metrics` apparaît également (~0,067 req/s) : c'est Prometheus qui la scrape toutes les 15 s.
    En l'absence de trafic, seule `/metrics` reste active.


### Panel 6 — Taux d'erreurs HTTP

**Type de visualisation :** Time series

**Requête PromQL :**
```promql
sum(rate(http_requests_total{status_code=~"5.."}[1m]))
```

**Réglages :**
- Title : `Erreurs HTTP 5xx/s`
- Unit : `errors/sec`
- Threshold : orange si > 0.1, rouge si > 1

**Réponse — Y a-t-il des erreurs ? Sur quelle route ?**

    Oui. La route `POST /orders` retourne aléatoirement un code `500` avec une probabilité de ~20 % (voir `index.js` de l'app NodeJS) : on observe environ **0,08 req/s en 5xx** sur cette route.
    La route `/error` renvoie systématiquement 500 mais elle n'est pas appelée par le générateur par défaut.
    La route `/notfound` génère des erreurs `404` (~0,4 req/s) qui n'apparaissent pas dans ce panel car le filtre est `5..`.

### Panel 7 — Latence P95 des requêtes

**Type de visualisation :** Time series

**Requête PromQL :**
```promql
histogram_quantile(0.95, sum by(le, route) (rate(http_request_duration_seconds_bucket[5m])))
```

**Réglages :**
- Title : `Latence P95 par route`
- Unit : `seconds`

**Réponse — Quelle route a la latence la plus élevée ? Pourquoi ?**

    En conditions normales, toutes les routes rapides (`/`, `/health`, `/users`, `/orders`, `/notfound`, `/metrics`) ont une P95 quasi-identique d'environ **4,75 ms** — le premier bucket de l'histogramme (5 ms) capture l'essentiel des requêtes.
    Dès qu'on appelle `/slow`, c'est elle qui prend la tête : le handler ajoute un `setTimeout` aléatoire entre 300 ms et 1 800 ms, donc sa P95 monte typiquement à **~1,5 s**. C'est attendu : la latence reflète directement le délai artificiel injecté côté serveur.


### Panel 8 — Utilisateurs actifs

**Type de visualisation :** Stat

**Requête PromQL :**
```promql
app_active_users
```

**Réglages :**
- Title : `Utilisateurs actifs`
- Choisissez une couleur adaptée

**Réponse — Comment évolue cette valeur dans le temps ?**

    `app_active_users` est une **Gauge** : sa valeur est réécrite à chaque appel de `GET /users` avec un entier aléatoire entre 1 et 100.
    Conséquence : la courbe est en dents de scie, sans tendance, et reste constante entre deux appels. La dernière valeur observée est de **61 utilisateurs actifs**.
    En production, ce serait un compteur réellement représentatif (sessions actives, websockets ouverts, etc.).

## Sauvegarder le dashboard

Une fois tous les panels créés :
1. Cliquez sur **Save dashboard** (icône disquette en haut à droite)
2. Donnez-lui le nom `Monitoring complet`
3. Cliquez sur **Save**

## Générer du trafic pour remplir les graphiques

Pour voir des données intéressantes dans vos panels, générez du trafic sur l'application NodeJS :

```bash
# Générer des requêtes en boucle
while true; do
  curl -s http://localhost:3000/ > /dev/null
  curl -s http://localhost:3000/health > /dev/null
  curl -s http://localhost:3000/slow > /dev/null
  curl -s http://localhost:3000/error > /dev/null
  curl -s -X POST http://localhost:3000/orders > /dev/null
  curl -s http://localhost:3000/users > /dev/null
  sleep 1
done
```

👉 Laissez ce script tourner pendant quelques minutes, puis observez l'évolution de vos graphiques.

## Résultat

Vous avez créé un dashboard complet qui permet de surveiller en un coup d'œil :
- l'état de votre infrastructure (CPU, RAM, disque)
- le comportement de votre application (requêtes, erreurs, latence)

C'est exactement ce qu'un ingénieur SRE surveille en production.
