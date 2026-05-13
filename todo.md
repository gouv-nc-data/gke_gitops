# TODO — Monitoring/alerting Argo Workflows + CronJobs & exposition UI Argo WF

> Ce fichier décrit les ressources à créer. Il est destiné à être lu par un LLM qui produira le code correspondant.

## Décisions actées

- **Monitoring** : Cloud Monitoring uniquement (pas de Prometheus self-hosted, pas de Grafana à opérer).
- **Sources** :
  - **Logs (Cloud Logging)** pour tous les événements de fail.
  - **GMP (Google Managed Prometheus)** uniquement si l'un des cas d'usage de la section "Faut-il GMP" est confirmé.
- **Alerting** : `google_monitoring_alert_policy` branchées sur les 3 notification channels email **déjà existants** dans `hub-vpc-gitops/main.tf` (`primary_contact`, `secondary_contact`, `org_admin_contact`).
- **Exposition UI Argo WF** : `Service type=LoadBalancer` interne (Internal Passthrough Network LB L4) avec IP réservée dans le subnet GKE existant. Pas d'Ingress, pas de cert-manager pour ce premier jet.
- **DNS** : nom interne uniquement à ce stade (pas de publication sous `*.gouv.nc`). Sera revu quand Teleport sera en prod on-premise.

---

## Référence : à quoi sert GMP dans ce combo ?

| Alerte                                       | Source nécessaire                                       | Pourquoi pas les logs ?                                                                  |
| -------------------------------------------- | ------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Workflow Argo Failed                         | **Logs**                                                | Événement discret, parfait                                                               |
| Job K8s en échec                             | **Logs**                                                | Idem                                                                                     |
| Pod OOMKilled / CrashLoop                    | **Logs** (events)                                       | Idem                                                                                     |
| Argo controller crash                        | **Logs**                                                | Idem                                                                                     |
| **CronJob qui n'a pas tourné** à l'heure prévue | **GMP** (`kube_cronjob_status_last_schedule_time`)      | Pas de log = silence radio, on ne peut pas alerter sur l'absence d'un événement de manière fiable |
| **Queue Argo qui sature** (>N workflows en attente) | **GMP** (`argo_workflows_queue_depth_count`)            | C'est une jauge instantanée, pas un événement                                            |
| **Workflow trop long** (durée > seuil)       | **GMP** (`argo_workflows_operation_duration_seconds`)   | Calculable depuis des logs mais tordu ; trivial en PromQL                                |

## Référence : faut-il GMP, oui ou non ?

Deux questions à trancher :

1. Y a-t-il des CronJobs critiques où "ne pas avoir tourné" est un incident ? (ex : backup nocturne, job de réplication horaire)
2. Risque-t-on un controller Argo saturé (beaucoup de workflows concurrents) ?

| Réponses                       | Recommandation                                                                            |
| ------------------------------ | ----------------------------------------------------------------------------------------- |
| **Non aux deux**               | **Pas de GMP**. Logs only. ~0 €/mois. Setup ½ j. Activable plus tard sans rejouer le reste. |
| **Oui à au moins une**         | **GMP activé**, limité au namespace `argo` + kube-state-metrics. ~2-5 €/mois.             |

**Statut** : à confirmer par l'équipe. Par défaut, démarrer en **logs only**, ajouter GMP plus tard si besoin.

---

## Ressources à créer

### A. Repo `hub-vpc-gitops` — Cloud Monitoring (Terraform)

Cible : nouveau fichier `monitoring.tf` (ou `monitoring/` si on veut séparer) dans le projet `prj-dinum-p-hub-vpc` *(à confirmer : peut aussi atterrir dans le projet qui héberge le cluster GKE — privilégier le projet du cluster pour que les métriques resource.* `k8s_container` *soient dans le bon scope)*.

#### A.1 — Log-based metrics (`google_logging_metric`)

Créer une métrique pour chacun des événements suivants. Filtre Cloud Logging à adapter au besoin après vérification du format réel des logs émis par le cluster.

1. **`argo_workflow_failed`**
   - Filtre :
     ```
     resource.type="k8s_container"
     resource.labels.namespace_name="argo"
     resource.labels.container_name="workflow-controller"
     severity>=ERROR
     (jsonPayload.message=~"Failed" OR jsonPayload.message=~"Errored")
     ```
   - `metric_descriptor` : `DELTA` / `INT64`.
   - Labels extraits si possible : `workflow_name`, `workflow_namespace`.

2. **`k8s_job_failed`**
   - Filtre sur events :
     ```
     resource.type="k8s_cluster"
     jsonPayload.reason="BackoffLimitExceeded"
     ```
     (ajuster selon ce qui remonte réellement — alternative : `protoPayload.methodName` sur les events K8s, ou `jsonPayload.involvedObject.kind="Job"` + `jsonPayload.type="Warning"`).
   - Labels : `job_name`, `namespace`.

3. **`pod_oomkilled`**
   - Filtre :
     ```
     resource.type="k8s_pod"
     jsonPayload.reason="OOMKilling"
     ```
     ou alternative via `k8s_node` / events.
   - Labels : `pod_name`, `namespace`.

4. **`pod_crashloopbackoff`**
   - Filtre sur events K8s reason `BackOff` avec `involvedObject.kind="Pod"`.
   - Labels : `pod_name`, `namespace`.

5. **`argo_controller_error`** (optionnel, redondant avec #1 mais sépare le controller des workflows)
   - Filtre :
     ```
     resource.type="k8s_container"
     resource.labels.container_name="workflow-controller"
     severity=ERROR
     ```

#### A.2 — Alert policies (`google_monitoring_alert_policy`)

Une policy par métrique ci-dessus, toutes branchées sur les 3 channels existants :

```hcl
notification_channels = [
  google_monitoring_notification_channel.primary_contact.id,
  google_monitoring_notification_channel.secondary_contact.id,
  google_monitoring_notification_channel.org_admin_contact.id,
]
```

Convention :
- `combiner = "OR"`
- Condition `condition_threshold` : `comparison = "COMPARISON_GT"`, `threshold_value = 0`, `duration = "60s"` (ou 0s pour fail immédiat).
- `alert_strategy.auto_close = "1800s"` (incidents fermés auto au bout de 30 min sans nouveau hit).
- `documentation.content` : courte phrase markdown expliquant l'alerte + lien Cloud Logging pré-filtré (paramètre `documentation.mime_type = "text/markdown"`).

Liste des policies à produire :

| # | Display name                          | Source                         | Seuil                | Durée |
| - | ------------------------------------- | ------------------------------ | -------------------- | ----- |
| 1 | "Argo Workflow failed"                | `argo_workflow_failed`         | > 0                  | 0s    |
| 2 | "Kubernetes Job failed (BackoffLimit)"| `k8s_job_failed`               | > 0                  | 0s    |
| 3 | "Pod OOMKilled"                       | `pod_oomkilled`                | > 0                  | 0s    |
| 4 | "Pod CrashLoopBackOff"                | `pod_crashloopbackoff`         | > 3 occurrences en 10 min | 600s  |
| 5 | "Argo workflow-controller errors"     | `argo_controller_error`        | > 5 en 5 min         | 300s  |

#### A.3 — (Optionnel, si décision "GMP = oui") Activation GMP + alertes métriques

Conditionner ces ressources à une variable Terraform `enable_gmp` (default `false`) :

- Activer GMP sur le cluster GKE (flag cluster ou via `google_container_cluster.monitoring_config.managed_prometheus.enabled = true`).
- Déployer côté `gke_gitops` (manifest k8s, voir section B) :
  - Un **`PodMonitoring`** pour le `workflow-controller` (port `:9090`, path `/metrics`).
  - **kube-state-metrics** s'il n'est pas déjà installé (Helm chart ou manifest), avec un `PodMonitoring` ciblant son endpoint.
- Trois alertes Cloud Monitoring supplémentaires basées sur les métriques GMP (`prometheus.googleapis.com/...`) :

  | # | Display name                            | Métrique PromQL équivalente                                                                 | Seuil          |
  | - | --------------------------------------- | ------------------------------------------------------------------------------------------- | -------------- |
  | 6 | "CronJob missed schedule"               | `time() - kube_cronjob_status_last_schedule_time > 2 * cron_interval`                       | true 5 min     |
  | 7 | "Argo workflow queue saturated"         | `argo_workflows_queue_depth_count > 50`                                                     | true 10 min    |
  | 8 | "Argo workflow long running"            | `histogram_quantile(0.95, rate(argo_workflows_operation_duration_seconds_bucket[5m])) > 1800` | true 10 min    |

  Le seuil exact de #6 doit être paramétrable par cronjob (label `cronjob`) — démarrer avec un seuil global de 1 h puis affiner.

---

### B. Repo `gke_gitops` (ce repo) — exposition UI Argo WF

#### B.1 — Réserver une IP interne fixe

Repo cible : `hub-vpc-gitops/main.tf` (là où sont déjà les `module "addresses"`), ou bien créer un nouveau `google_compute_address` dans le projet du cluster.

```hcl
resource "google_compute_address" "argowf_ilb" {
  name         = "argowf-ui-internal"
  project      = <project du cluster GKE>
  region       = var.region
  subnetwork   = <self_link du subnet des nodes GKE>
  address_type = "INTERNAL"
  # Laisser GCP allouer l'IP, ou fixer une valeur si on veut un mapping DNS stable.
}
```

À exporter en output pour récupération côté GitOps.

#### B.2 — Patcher `apps/data/argo_workflow/values.yaml`

Modifier la section `server.service` :

```yaml
server:
  # ... (conserver l'existant : secure, extraArgs, extraEnv, resources)
  service:
    type: LoadBalancer
    annotations:
      networking.gke.io/load-balancer-type: "Internal"
      networking.gke.io/internal-load-balancer-allow-global-access: "true"
    loadBalancerIP: "<IP allouée par google_compute_address.argowf_ilb>"
```

**Note auth** : tant que le service est joignable depuis le réseau interne, **passer `--auth-mode=client`** (et retirer `--auth-mode=server`) dans `server.extraArgs` pour exiger un token. Le mode `server` rend l'UI accessible anonymement à tout ce qui peut atteindre l'IP.

#### B.3 — Firewall rule GCP

Autoriser le trafic depuis les ranges VPN (`172.20.10.0/24`, `172.20.11.128/25` — cf. `hub-vpc-gitops/main.tf` `local.vpn_remote_ranges`) vers les nodes GKE sur le port du service Argo WF (par défaut `2746`).

```hcl
resource "google_compute_firewall" "allow_argowf_ui_from_onprem" {
  name      = "allow-argowf-ui-from-onprem"
  project   = <project du cluster GKE>
  network   = <network du cluster>
  direction = "INGRESS"
  source_ranges = ["172.20.10.0/24", "172.20.11.128/25"]
  target_tags   = [<tag des nodes GKE>] # ou target_service_accounts
  allow {
    protocol = "tcp"
    ports    = ["2746"]
  }
}
```

#### B.4 — Vérifications post-déploiement (manuelles, non automatisables ici)

- [ ] Le LB est bien créé en interne (`gcloud compute forwarding-rules list --filter="loadBalancingScheme=INTERNAL"`).
- [ ] L'IP est bien dans le subnet attendu.
- [ ] Tester depuis une machine on-prem : `curl -k https://<IP>:2746/`.
- [ ] Vérifier que Palo Alto laisse passer le flux dans ce sens (point régulièrement oublié).
- [ ] Vérifier que `--auth-mode=client` est bien actif et qu'on est challengé par un token.

---

### C. Étapes ultérieures (hors scope ce ticket, à tracer)

- DNS interne : entrée `argowf.dinum.<zone interne>` → IP du LB, à coordonner avec l'équipe DNS gouv.nc.
- TLS valide (cert-manager + ICA interne) si on veut éviter le `-k` curl/`certificat non valide` navigateur.
- Bascule vers Teleport Application Access dès qu'il est en prod on-premise → permettra de supprimer l'ILB et la firewall rule (Argo WF repassé en ClusterIP).
- Mutualiser un Internal HTTP(S) LB + Ingress si plus de 2 UI à exposer (ArgoCD, Grafana managé, etc.).

---

## Ordre d'implémentation suggéré

1. A.1 + A.2 (alertes log-based — gain immédiat, ~0 €).
2. B.1 → B.3 (exposition Argo WF UI via ILB).
3. B.4 (vérifs).
4. A.3 (GMP) **seulement après décision** sur les 2 questions de la section "Faut-il GMP".
5. C (DNS, TLS, Teleport) — itérations suivantes.
