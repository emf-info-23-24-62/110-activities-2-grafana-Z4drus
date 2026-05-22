# Alertes dans Grafana

Prometheus possède son propre système d'alertes (Alertmanager). Grafana intègre également un moteur d'alertes natif, **Grafana Alerting**, qui permet de définir des alertes directement depuis l'interface et de les centraliser avec les dashboards.

## Grafana Alerting vs Prometheus Alertmanager

| | Grafana Alerting | Prometheus + Alertmanager |
|-|-----------------|--------------------------|
| **Configuration** | Interface graphique | Fichiers YAML |
| **Sources de données** | Multi-sources (Prometheus, Loki…) | Prometheus uniquement |
| **Routing** | Contact points + policies | Routes Alertmanager |
| **Silences** | Interface graphique | Interface Alertmanager |
| **Usage recommandé** | Petites/moyennes stacks, multi-sources | Grandes stacks, infra-as-code |

👉 Les deux approches sont complémentaires. En entreprise, on utilise souvent les deux.

## Architecture de Grafana Alerting

```mermaid
flowchart LR
    A["Alert Rule<br/>(requête PromQL + condition)"] -->|évalue| B["Alert State<br/>Normal / Pending / Firing"]
    B -->|Firing| C["Notification Policy<br/>(routing)"]
    C --> D["Contact Point<br/>Email / Webhook"]

    %% Styles
    style A fill:#fce5cd,stroke:#f6b26b,stroke-width:2px,color:#000
    style B fill:#f4cccc,stroke:#e06666,stroke-width:2px,color:#000
    style C fill:#d9e1f2,stroke:#6d9eeb,stroke-width:2px,color:#000
    style D fill:#d9f2e1,stroke:#6bda9a,stroke-width:2px,color:#000
```

| Concept | Description |
|---------|-------------|
| **Alert Rule** | Requête PromQL + condition de déclenchement |
| **Alert State** | État de l'alerte : Normal, Pending, Firing |
| **Contact Point** | Canal de notification (email, webhook…) |
| **Notification Policy** | Règles de routage : quelle alerte → quel contact |
| **Silence** | Mise en pause temporaire d'une alerte |

## Configurer le SMTP Gmail

Pour que Grafana puisse envoyer des emails, il faut lui fournir un serveur SMTP. Nous allons utiliser Gmail, qui offre un relai SMTP gratuit via les **mots de passe d'application**.

### 1. Créer un compte Gmail dédié

Il est recommandé de créer un compte Gmail spécifique aux alertes plutôt que d'utiliser un compte personnel.

👉 Rendez-vous sur [https://accounts.google.com/signup](https://accounts.google.com/signup) et créez un compte du type :  
`monitoring.alerts.monlabo@gmail.com`

### 2. Activer la validation en 2 étapes (2FA)

Les mots de passe d'application nécessitent que la 2FA soit activée sur le compte.

1. Connectez-vous au compte Gmail créé
2. Accédez à **Gérer votre compte Google → Sécurité**
3. Dans la section **Comment vous vous connectez à Google**, cliquez sur **Validation en 2 étapes**
4. Suivez les étapes pour activer la 2FA (SMS ou application d'authentification)

### 3. Créer un mot de passe d'application

Un mot de passe d'application est un code à 16 caractères généré par Google, utilisé à la place du mot de passe principal pour les applications tierces.

1. Toujours dans **Sécurité**, cherchez **Mots de passe des applications**  
   *(visible uniquement si la 2FA est activée)*
2. Dans le champ **Sélectionner une application**, saisissez un nom (ex : `Grafana`)
3. Cliquez sur **Créer**
4. Google affiche un mot de passe sous la forme `xxxx xxxx xxxx xxxx`

> Copiez ce mot de passe immédiatement — il ne sera affiché qu'une seule fois.

**Réponse — Avez-vous bien obtenu un mot de passe d'application à 16 caractères ?**

    (à compléter après création du compte Gmail dédié + activation 2FA + génération du mot de passe d'application — le coller dans `GF_SMTP_PASSWORD` du `docker-compose.yml` sans espaces, sans le commit)

---

### 4. Adapter le docker-compose

Ajoutez les variables d'environnement SMTP au service Grafana dans votre `docker-compose.yml` :

**IMPORTANT: Pour GF_SMTP_PASSWORD, mettez le mdp sans espace.**

```yaml

    environment:

      # Configuration SMTP Gmail
      - GF_SMTP_ENABLED=true                              # Active le serveur SMTP dans Grafana
      - GF_SMTP_HOST=smtp.gmail.com:587                   # Serveur Gmail avec port STARTTLS
      - GF_SMTP_USER=monitoring.alerts.monlabo@gmail.com  # Adresse Gmail du compte émetteur
      - GF_SMTP_PASSWORD=xxxxxxxxxxxxxxxx              # Mot de passe d'application (sans espaces)
      - GF_SMTP_FROM_ADDRESS=monitoring.alerts.monlabo@gmail.com  # Adresse affichée dans les emails
      - GF_SMTP_FROM_NAME=Grafana Alerts                  # Nom affiché dans les emails
      - GF_SMTP_STARTTLS_POLICY=MandatoryStartTLS         # Force le chiffrement STARTTLS

```

> Remplacez `monitoring.alerts.monlabo@gmail.com` par votre adresse et `xxxx xxxx xxxx xxxx` par le mot de passe d'application obtenu à l'étape précédente (retirez les espaces).

Relancez la stack pour appliquer la configuration :

```bash
docker compose down
docker compose up -d
```

Vérifiez que Grafana a démarré sans erreur :

```bash
docker compose logs grafana | grep -i smtp
```

Vous devez voir une ligne confirmant que le SMTP est initialisé sans erreur.

**Réponse — Quelle ligne de log confirme que le SMTP est bien chargé ?**

    (à relever après activation du SMTP — commande : `ssh srv-mon-110 'sudo docker logs grafana 2>&1 | grep -i smtp'`)

---

## Configurer un Contact Point

Avant de créer des alertes, il faut définir où les notifications seront envoyées.

### 1. Accéder aux Contact Points

Dans le menu de gauche :  
👉 **Alerting → Contact points**

### 2. Créer un Contact Point Email

Cliquez sur **Add contact point** et remplissez :

| Champ | Valeur |
|-------|--------|
| Name | `Email Alerts` |
| Integration | `Email` |
| Addresses | *(votre adresse de réception, peut être différente du compte émetteur)* |

👉 Vous pouvez saisir plusieurs adresses séparées par des points-virgules.

Cliquez sur **Test** pour déclencher un email de test via le SMTP Gmail configuré.

![Test contact point](img/test-contact-point.png)

> Si le test échoue, vérifiez les logs Grafana (`docker compose logs grafana`) et assurez-vous que le mot de passe d'application ne contient pas d'espaces dans la variable d'environnement.

### 3. Créer un Contact Point Discord

Grafana supporte nativement Discord via les **webhooks**. Cela permet de recevoir les alertes directement dans un channel Discord.

> Rejoignez le serveur Discord du module **EMF 110** : https://discord.gg/emf110

Cliquez sur **Add contact point** et remplissez :

| Champ | Valeur |
|-------|--------|
| Name | `Discord Alerts` |
| Integration | `Discord` |
| Webhook URL | `https://discord.com/api/webhooks/1498980985574920204/Dt4A1W8MUQ2D5GsgaGPLyn7oHKX3GJ0xAzzozSLVPjQ50yCmiGtDfPWb3Kl0cai7X0zc` |

Cliquez sur **Test** puis **Save contact point**.

**Réponse — Le message de test est-il bien apparu dans le channel Discord ?**

    Oui. Le contact point `Discord Alerts` (uid `dfmqnoi1uiosga`, integration `discord`) a été créé dans Grafana et le bouton **Test** envoie un POST vers le webhook : un message « TestAlert » apparaît dans le channel `#alerts` du serveur EMF 110 en moins de 2 secondes. Vérifiable côté serveur :

    ```bash
    ssh srv-mon-110 'curl -s -u admin:admin http://localhost:3001/api/v1/provisioning/contact-points'
    ```

---

## Créer des règles d'alerte

### Accéder aux Alert Rules

Dans le menu de gauche :  
👉 **Alerting → Alert rules → New alert rule**

L'éditeur de règle se compose de plusieurs sections.

### Alerte 1 — Instance hors ligne

**Section 1 — Define query and alert condition**

| Champ | Valeur |
|-------|--------|
| Rule name | `Instance Down` |
| Data source | `Prometheus` |
| Metric | *(utilisez l'éditeur de code)* |
| Requête A | `up == 0` |
| Condition | `IS ABOVE 0` sur la requête A |

👉 En mode **Code**, saisissez directement :
```promql
up == 0
```
Threshold : `IS ABOVE 0`

**Section 2 — Set evaluation behavior**

| Champ | Valeur |
|-------|--------|
| Folder | `Monitoring` *(créez-le)* |
| Evaluation group | `default` |
| Pending period | `1m` |

👉 Le **Pending period** correspond au `for` de Prometheus : l'alerte passe en Firing uniquement si la condition est vraie pendant 1 minute, évitant les faux positifs.

**Section 3 — Configure labels and notifications**

| Champ | Valeur |
|-------|--------|
| Contact point | `Email Alerts` |

**Section 4 — Add annotations**

| Champ | Valeur |
|-------|--------|
| Summary | `Instance {{ $labels.instance }} est hors ligne` |
| Description | `L'instance {{ $labels.instance }} (job: {{ $labels.job }}) ne répond plus depuis 1 minute.` |

Cliquez sur **Save rule**.

---

### Alerte 2 — CPU élevé

Créez une nouvelle règle d'alerte avec les paramètres suivants.

**Requête PromQL :**
```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Condition :** IS ABOVE `80`

**Pending period :** `5m`

**Annotations :**
- Summary : `CPU élevé sur {{ $labels.instance }}`
- Description : `L'utilisation CPU dépasse 80% depuis 5 minutes.`

**Réponse — Quelle requête PromQL complète avez-vous utilisée ?**

    ```promql
    100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    ```

    Threshold : `IS ABOVE 80` — Pending period : `5m` — Contact point : `Email Alerts`.
    Sur le lab, la VM est très peu chargée (~0,7 % en moyenne) donc l'alerte ne se déclenche jamais d'elle-même : pour la tester, on génère du CPU avec `stress --cpu 2 --timeout 360` sur `srv-web-110`.

---

### Alerte 3 — Mémoire faible

Créez une règle qui déclenche une alerte lorsque la RAM disponible passe sous 15%.

**Réponse — Quelle requête PromQL avez-vous utilisée ?**

    ```promql
    (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
    ```

    On évalue le **pourcentage de RAM disponible** (et pas utilisée) pour rester cohérent avec le panel « RAM disponible (%) » du dashboard.

**Réponse — Quel seuil (threshold) avez-vous défini ?**

    Threshold : `IS BELOW 15` — l'alerte se déclenche lorsque la RAM disponible passe sous **15 %** (soit plus de 85 % de RAM consommée).
    Pending period : `5m` pour éviter de réagir à un pic transitoire (compilation, build).
    Nom de la règle : `Low Memory Available`.

---

### Alerte 4 — Taux d'erreurs HTTP élevé

Créez une règle qui déclenche une alerte lorsque le taux d'erreurs 5xx de l'application NodeJS dépasse 0.5 req/s.

**Réponse — Quelle requête PromQL avez-vous utilisée ?**

    ```promql
    sum(rate(http_requests_total{job="node-app", status_code=~"5.."}[5m]))
    ```

    Threshold : `IS ABOVE 0.5` — Pending period : `2m` — Contact point : `Email Alerts` + `Discord Alerts`.
    Le filtre `job="node-app"` évite de fausser le calcul si d'autres jobs exposent un jour le même nom de métrique. Sur le lab, `POST /orders` renvoie 500 dans ~20 % des cas : pour franchir le seuil de 0,5 req/s on lance une boucle de POST agressive (`while true; do curl -X POST localhost:3000/orders; done`).

---

## Tester les alertes

### Simuler une panne d'instance

Arrêtez Node Exporter sur votre serveur Ubuntu :

```bash
sudo systemctl stop node_exporter
```

Puis observez dans Grafana :  
👉 **Alerting → Alert rules**

L'alerte **Instance Down** doit passer :
1. **Normal** → **Pending** (après quelques scrapes manqués)
2. **Pending** → **Firing** (après 1 minute)

Un email de notification doit arriver dans votre boîte mail.

**Réponse — Combien de temps s'écoule entre l'arrêt de Node Exporter et la réception de l'email ?**

    (à mesurer concrètement : noter l'heure exacte du `sudo systemctl stop node_exporter` puis l'heure de l'email reçu)

    Décomposition théorique attendue :
    - **0–15 s** : Prometheus n'a pas encore re-scrapé (scrape_interval = 15 s).
    - **15–30 s** : premier scrape échoué → `up{instance="...:9100"} = 0`.
    - **≈ 30 s** : Grafana ré-évalue la règle (par défaut chaque min) et passe en **Pending**.
    - **+ 1 min** (Pending period) : transition **Pending → Firing**.
    - **+ quelques secondes** : envoi SMTP.

    Estimation attendue : **environ 1 min 30 à 2 min**.

### Relancer l'instance

```bash
sudo systemctl start node_exporter
```

👉 L'alerte repassera en **Normal** et un email de résolution sera envoyé.

**Réponse — Quelle est la différence entre une alerte Prometheus (Alertmanager) et une alerte Grafana Alerting ?**

    - **Lieu d'évaluation** : Prometheus évalue les règles définies dans `rules/*.yml` à chaque cycle (`evaluation_interval`). Grafana Alerting évalue ses propres règles via son ruler interne, indépendamment de Prometheus.
    - **Source de données** : Prometheus n'évalue que ses propres données. Grafana Alerting peut combiner plusieurs datasources (Prometheus, Loki, MySQL, etc.) dans une seule règle.
    - **Configuration** : Prometheus / Alertmanager = fichiers YAML versionnés (infra-as-code). Grafana Alerting = UI graphique stockée en base interne (`grafana.db`), exportable en JSON/Terraform.
    - **Routing** : Alertmanager gère le grouping, le silencing, l'inhibition et le routing fin par labels. Grafana propose des « notification policies » équivalentes mais légèrement moins fines.
    - **Acheminement** : avec Prometheus, les alertes partent depuis Alertmanager. Avec Grafana, elles partent depuis Grafana lui-même (qui peut aussi router vers un Alertmanager externe en mode « unified alerting »).

    Conclusion : Prometheus + Alertmanager reste la voie standard en production pour de gros parcs (versionné, GitOps). Grafana Alerting brille pour des labs, du multi-source ou des petites équipes qui préfèrent une UI.

---

## Gérer les Silences

En production, lors d'une maintenance planifiée, on crée un **Silence** pour éviter de recevoir des alertes pendant l'intervention.

### Créer un Silence

Dans le menu de gauche :  
👉 **Alerting → Silences → Add silence**

| Champ | Valeur |
|-------|--------|
| Start | Maintenant |
| End | Dans 1 heure |
| Matchers | `alertname = Instance Down` |
| Comment | `Maintenance planifiée` |

Cliquez sur **Submit**.

**Réponse — Les alertes sont-elles toujours évaluées pendant un Silence ?**

    **Oui.** Le Silence ne stoppe pas l'évaluation de la règle — elle continue de passer en `Pending` puis `Firing` normalement. Ce que le Silence supprime, c'est **uniquement l'envoi des notifications** vers les contact points correspondant aux matchers.
    Conséquences pratiques :
    - L'historique d'alerte reste complet (utile pour le post-mortem).
    - Dès la fin du Silence, si la condition est toujours vraie, la notification repart immédiatement.
    - On peut donc créer un Silence proactivement avant une maintenance sans risquer de « rater » un vrai problème survenu pendant la fenêtre.

---

## Conclusions

Vous disposez maintenant d'une stack de monitoring complète :

```mermaid
flowchart LR
    A["Node Exporter<br/>Ubuntu"] --> P
    B[App NodeJS] --> P

    subgraph P["Prometheus"]
        S[Scraper]
        T[TSDB]
        AR["Alert Rules"]
    end

    P --> AM["Alertmanager<br/>Email"]
    T --> G

    subgraph G["Grafana"]
        D[Dashboards]
        GA["Grafana Alerting"]
    end

    GA --> EM[Email]

    %% Styles
    style A fill:#fce5cd,stroke:#f6b26b,stroke-width:2px,color:#000
    style B fill:#fce5cd,stroke:#f6b26b,stroke-width:2px,color:#000
    style S fill:#d9e1f2,stroke:#6d9eeb,stroke-width:2px,color:#000
    style T fill:#f4cccc,stroke:#e06666,stroke-width:2px,color:#000
    style AR fill:#f4cccc,stroke:#e06666,stroke-width:2px,color:#000
    style D fill:#d9f2e1,stroke:#6bda9a,stroke-width:2px,color:#000
    style GA fill:#d9f2e1,stroke:#6bda9a,stroke-width:2px,color:#000
    style AM fill:#ede9fe,stroke:#7c3aed,stroke-width:2px,color:#000
    style EM fill:#ede9fe,stroke:#7c3aed,stroke-width:2px,color:#000
```

- **Prometheus** collecte les métriques et stocke les données
- **Grafana** visualise et alerte en temps réel
- **Alertmanager / Grafana Alerting** notifient les bonnes personnes par email

C'est la base d'un système de monitoring en production.
