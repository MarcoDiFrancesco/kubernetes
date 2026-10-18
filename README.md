# Kubernetes Cluster Architecture

A single-node **K3s** cluster running on a **Raspberry Pi 5** (`pi5`), backed by high-speed NVMe storage (`/mnt/nvme`). Inbound traffic is tunneled via Cloudflare Zero Trust into Traefik, which routes traffic across isolated namespaces hosting media, monitoring, AI proxying, container registry, torrenting, and automation services.

---

## Cluster Overview Diagram

```mermaid
graph TD
    %% Color Definitions
    classDef cloudflare fill:#A0D18C,stroke:#70A65A,color:#000;
    classDef traefik fill:#24A1C1,stroke:#176E84,color:#FFF;
    classDef headlamp fill:#2E7D32,stroke:#1B5E20,color:#FFF;
    classDef grafana fill:#F29B38,stroke:#C87B2A,color:#000;
    classDef prometheus fill:#E6522C,stroke:#B0391A,color:#FFF;
    classDef telegraf fill:#2B7BB9,stroke:#1E5580,color:#FFF;
    classDef postgres fill:#336791,stroke:#234A69,color:#FFF;
    classDef redis fill:#DC382D,stroke:#9E241C,color:#FFF;
    classDef ai fill:#10A37F,stroke:#0B6B53,color:#FFF;
    classDef film fill:#00A4DC,stroke:#00759E,color:#FFF;
    classDef finance fill:#059669,stroke:#047857,color:#FFF;
    classDef litellm fill:#7C3AED,stroke:#5B21B6,color:#FFF;
    classDef photos fill:#4F46E5,stroke:#3730A3,color:#FFF;
    classDef portainer fill:#13BEBB,stroke:#0E8C89,color:#000;
    classDef registry fill:#2496ED,stroke:#1565C0,color:#FFF;
    classDef torrent fill:#3B82F6,stroke:#1D4ED8,color:#FFF;
    classDef cronjob fill:#C084FC,stroke:#9333EA,color:#000;

    User((User)) -->|HTTPS| CF_EDGE(("Cloudflare Edge<br>[*.marcodifrancesco.com]"))
    CF_EDGE -->|Tunnel| CF_TUNNEL(["Cloudflare Tunnel<br>[cloudflared]"])
    CF_TUNNEL -->|HTTP| TRAEFIK_SVC

    subgraph "Namespace: kube-system"
        subgraph "Traefik Ingress Controller"
            TRAEFIK_SVC{{"Service<br>[traefik]<br>(LoadBalancer)"}}
            TRAEFIK_DEPLOY(["Deployment<br>[traefik]"])
            TRAEFIK_METRICS_SVC{{"Service<br>[traefik-metrics]<br>(ClusterIP)"}}

            TRAEFIK_SVC -.-> TRAEFIK_DEPLOY
            TRAEFIK_METRICS_SVC -.-> TRAEFIK_DEPLOY
        end
    end

    %% Ingress Routing
    TRAEFIK_SVC -->|ai.marcodifrancesco.com| AI_SVC
    TRAEFIK_SVC -->|dashboard.marcodifrancesco.com| GRAF_SVC
    TRAEFIK_SVC -->|headlamp.marcodifrancesco.com| HEADLAMP_SVC
    TRAEFIK_SVC -->|film.marcodifrancesco.com| FILM_SVC
    TRAEFIK_SVC -->|finance.marcodifrancesco.com/api| FIN_SVC
    TRAEFIK_SVC -->|litellm.marcodifrancesco.com| LLM_SVC
    TRAEFIK_SVC -->|photos.marcodifrancesco.com| PHOTO_SVC
    TRAEFIK_SVC -->|portainer.marcodifrancesco.com| PORT_SVC
    TRAEFIK_SVC -->|registry.marcodifrancesco.com| REG_SVC
    TRAEFIK_SVC -->|prowlarr.marcodifrancesco.com| TOR_PROW_SVC
    TRAEFIK_SVC -->|radarr.marcodifrancesco.com| TOR_RAD_SVC
    TRAEFIK_SVC -->|qbittorrent.marcodifrancesco.com| TOR_QBIT_SVC

    subgraph "Namespace: ai"
        subgraph "Open WebUI"
            AI_SVC{{"Service<br>[open-webui]<br>(ClusterIP)"}}
            AI_DEPLOY(["Deployment<br>[open-webui]"])

            AI_SVC -.-> AI_DEPLOY
        end

        subgraph "SearXNG"
            AI_SEARX_SVC{{"Service<br>[searxng]<br>(ClusterIP)"}}
            AI_SEARX_DEPLOY(["Deployment<br>[searxng]"])

            AI_SEARX_SVC -.-> AI_SEARX_DEPLOY
        end

        AI_DEPLOY -. "web search" .-> AI_SEARX_SVC
    end

    subgraph "Namespace: monitoring"
        subgraph "Headlamp"
            HEADLAMP_SVC{{"Service<br>[headlamp]<br>(ClusterIP)"}}
            HEADLAMP_DEPLOY(["Deployment<br>[headlamp]"])

            HEADLAMP_SVC -.-> HEADLAMP_DEPLOY
        end

        subgraph "Grafana"
            GRAF_SVC{{"Service<br>[grafana]<br>(ClusterIP)"}}
            GRAF_DEPLOY(["Deployment<br>[grafana]"])

            GRAF_SVC -.-> GRAF_DEPLOY
        end

        subgraph "Prometheus"
            PROM_SVC{{"Service<br>[prometheus]<br>(ClusterIP)"}}
            PROM_DEPLOY(["Deployment<br>[prometheus]"])

            PROM_SVC -.-> PROM_DEPLOY
        end

        subgraph "Telegraf"
            TELE_SVC{{"Service<br>[telegraf]<br>(ClusterIP)"}}
            TELE_DEPLOY(["Deployment<br>[telegraf]"])

            TELE_SVC -.-> TELE_DEPLOY
        end

        GRAF_DEPLOY -. "queries" .-> PROM_SVC
        PROM_DEPLOY -. "scrapes" .-> TELE_SVC
    end

    PROM_DEPLOY -. "scrapes" .-> TRAEFIK_METRICS_SVC
    PROM_DEPLOY -. "scrapes" .-> LLM_SVC

    subgraph "Namespace: film"
        subgraph "Jellyfin"
            FILM_SVC{{"Service<br>[jellyfin]<br>(ClusterIP)"}}
            FILM_DEPLOY(["Deployment<br>[jellyfin]"])

            FILM_SVC -.-> FILM_DEPLOY
        end
    end

    subgraph "Namespace: finance"
        subgraph "Finance API"
            FIN_SVC{{"Service<br>[finance-api]<br>(ClusterIP)"}}
            FIN_DEPLOY(["Deployment<br>[finance-api]"])

            FIN_SVC -.-> FIN_DEPLOY
        end
    end

    subgraph "Namespace: litellm"
        LLM_CJ["CronJob<br>[litellm-restarter]<br>(0 2 * * *)"]

        subgraph "LiteLLM Proxy"
            LLM_SVC{{"Service<br>[litellm]<br>(ClusterIP)"}}
            LLM_DEPLOY(["Deployment<br>[litellm]"])

            LLM_SVC -.-> LLM_DEPLOY
        end

        subgraph "LiteLLM PostgreSQL"
            LLM_PG_SVC{{"Service<br>[postgres]<br>(ClusterIP)"}}
            LLM_PG_DEPLOY(["Deployment<br>[postgres]"])

            LLM_PG_SVC -.-> LLM_PG_DEPLOY
        end

        LLM_DEPLOY -.-> LLM_PG_SVC
        LLM_CJ -. "rollout restart" .-> LLM_DEPLOY
    end

    subgraph "Namespace: photos"
        subgraph "Immich Server"
            PHOTO_SVC{{"Service<br>[immich-server]<br>(ClusterIP)"}}
            PHOTO_DEPLOY(["Deployment<br>[immich-server]"])

            PHOTO_SVC -.-> PHOTO_DEPLOY
        end

        subgraph "Immich Machine Learning"
            PHOTO_ML_SVC{{"Service<br>[immich-machine-learning]<br>(ClusterIP)"}}
            PHOTO_ML_DEPLOY(["Deployment<br>[immich-machine-learning]"])

            PHOTO_ML_SVC -.-> PHOTO_ML_DEPLOY
        end

        subgraph "Immich PostgreSQL"
            PHOTO_PG_SVC{{"Service<br>[postgres]<br>(ClusterIP)"}}
            PHOTO_PG_DEPLOY(["Deployment<br>[postgres-vectorchord]"])

            PHOTO_PG_SVC -.-> PHOTO_PG_DEPLOY
        end

        subgraph "Immich Redis"
            PHOTO_REDIS_SVC{{"Service<br>[redis]<br>(ClusterIP)"}}
            PHOTO_REDIS_DEPLOY(["Deployment<br>[valkey]"])

            PHOTO_REDIS_SVC -.-> PHOTO_REDIS_DEPLOY
        end

        PHOTO_DEPLOY -.-> PHOTO_PG_SVC
        PHOTO_DEPLOY -.-> PHOTO_REDIS_SVC
        PHOTO_DEPLOY -.-> PHOTO_ML_SVC
    end

    subgraph "Namespace: portainer"
        subgraph "Portainer CE"
            PORT_SVC{{"Service<br>[portainer]<br>(ClusterIP)"}}
            PORT_DEPLOY(["Deployment<br>[portainer]"])

            PORT_SVC -.-> PORT_DEPLOY
        end
    end

    subgraph "Namespace: registry"
        subgraph "Docker Registry"
            REG_SVC{{"Service<br>[registry]<br>(ClusterIP)"}}
            REG_DEPLOY(["Deployment<br>[registry]"])

            REG_SVC -.-> REG_DEPLOY
        end
    end

    subgraph "Namespace: torrent"
        subgraph "Prowlarr"
            TOR_PROW_SVC{{"Service<br>[prowlarr]<br>(ClusterIP)"}}
            TOR_PROW_DEPLOY(["Deployment<br>[prowlarr]"])

            TOR_PROW_SVC -.-> TOR_PROW_DEPLOY
        end

        subgraph "FlareSolverr"
            TOR_FLARE_SVC{{"Service<br>[flaresolverr]<br>(ClusterIP)"}}
            TOR_FLARE_DEPLOY(["Deployment<br>[flaresolverr]"])

            TOR_FLARE_SVC -.-> TOR_FLARE_DEPLOY
        end

        subgraph "Radarr"
            TOR_RAD_SVC{{"Service<br>[radarr]<br>(ClusterIP)"}}
            TOR_RAD_DEPLOY(["Deployment<br>[radarr]"])

            TOR_RAD_SVC -.-> TOR_RAD_DEPLOY
        end

        subgraph "qBittorrent"
            TOR_QBIT_SVC{{"Service<br>[qbittorrent]<br>(ClusterIP)"}}
            TOR_QBIT_DEPLOY(["Deployment<br>[qbittorrent]"])

            TOR_QBIT_SVC -.-> TOR_QBIT_DEPLOY
        end

        TOR_PROW_DEPLOY -. "captchas" .-> TOR_FLARE_SVC
        TOR_PROW_DEPLOY -. "syncs" .-> TOR_RAD_SVC
        TOR_RAD_DEPLOY -. "downloads" .-> TOR_QBIT_SVC
    end

    %% Apply Colors
    class CF_EDGE,CF_TUNNEL cloudflare;
    class TRAEFIK_SVC,TRAEFIK_METRICS_SVC,TRAEFIK_DEPLOY traefik;
    class HEADLAMP_SVC,HEADLAMP_DEPLOY headlamp;
    class GRAF_SVC,GRAF_DEPLOY grafana;
    class PROM_SVC,PROM_DEPLOY prometheus;
    class TELE_SVC,TELE_DEPLOY telegraf;
    class AI_SVC,AI_DEPLOY,AI_SEARX_SVC,AI_SEARX_DEPLOY ai;
    class FILM_SVC,FILM_DEPLOY film;
    class FIN_SVC,FIN_DEPLOY finance;
    class LLM_SVC,LLM_DEPLOY litellm;
    class LLM_PG_SVC,LLM_PG_DEPLOY,PHOTO_PG_SVC,PHOTO_PG_DEPLOY postgres;
    class PHOTO_SVC,PHOTO_DEPLOY,PHOTO_ML_SVC,PHOTO_ML_DEPLOY photos;
    class PHOTO_REDIS_SVC,PHOTO_REDIS_DEPLOY redis;
    class PORT_SVC,PORT_DEPLOY portainer;
    class REG_SVC,REG_DEPLOY registry;
    class TOR_PROW_SVC,TOR_PROW_DEPLOY,TOR_RAD_SVC,TOR_RAD_DEPLOY,TOR_QBIT_SVC,TOR_QBIT_DEPLOY,TOR_FLARE_SVC,TOR_FLARE_DEPLOY torrent;
    class LLM_CJ cronjob;
```

---

## Namespaces & Workloads

| Namespace | Service / Deployment | Ingress Host |
|---|---|---|
| **`kube-system`** | `traefik`<br>`traefik-metrics` | `192.168.1.101` |
| **`ai`** | `open-webui`<br>`searxng` | `ai.marcodifrancesco.com` |
| **`monitoring`** | `grafana`<br>`headlamp`<br>`prometheus`<br>`telegraf` | `dashboard.marcodifrancesco.com`<br>*(redirect: `grafana.`)*<br>`headlamp.marcodifrancesco.com` |
| **`film`** | `jellyfin` | `film.marcodifrancesco.com`<br>*(redirect: `jellyfin.`)* |
| **`finance`** | `finance-api` | `finance.marcodifrancesco.com/api` |
| **`litellm`** | `litellm`<br>`postgres`<br>`litellm-restarter` | `litellm.marcodifrancesco.com` |
| **`photos`** | `immich-server`<br>`immich-machine-learning`<br>`postgres`<br>`redis` | `photos.marcodifrancesco.com`<br>*(redirect: `immich.`)* |
| **`portainer`** | `portainer` | `portainer.marcodifrancesco.com` |
| **`registry`** | `registry` | `registry.marcodifrancesco.com` |
| **`torrent`** | `prowlarr`<br>`flaresolverr`<br>`radarr`<br>`qbittorrent` | `prowlarr.marcodifrancesco.com`<br>`radarr.marcodifrancesco.com`<br>`qbittorrent.marcodifrancesco.com` |

---

## Architecture Details

### Ingress & Network Flow
1. **Cloudflare Zero Trust**: Public requests to `*.marcodifrancesco.com` terminate on Cloudflare Edge and pass through an encrypted tunnel (`cloudflared`) directly to the node.
2. **Traefik Ingress Controller**: Serves traffic on ports `80` and `443`, evaluating Kubernetes `Ingress` resources and applying custom middlewares.

### Persistent Storage
- Persistent volumes are backed by NVMe storage mounted on `/mnt/nvme`.
- Application configuration and runtime state reside in `/mnt/nvme/k8s/data/<namespace>/`.

### Resource Limits
- Every namespace enforces a baseline `LimitRange` standardizing default CPU/memory limits (`2 CPUs`, `1-2 GiB` RAM) and default requests (`100m CPU`, `128-256 MiB` RAM).
