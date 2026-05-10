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

    (votre réponse ici)


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

    (votre réponse ici)


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

    (votre réponse ici)


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

    (votre réponse ici)


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

    (votre réponse ici)


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

    (votre réponse ici)

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

    (votre réponse ici)


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

    (votre réponse ici)

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
