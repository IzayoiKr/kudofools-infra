# kudofools-infra

Kubernetes cluster infrastructure managed by Flux CD on a single-node k3s (Raspberry Pi 5, Ubuntu 24.04, 8GB RAM).

## Architecture

```mermaid
graph LR
    %% External Network
    subgraph Ext [External]
        direction TB
        REPO[This Git Repo]
        CF[Cloudflare Tunnel]
    end

    %% K3s Cluster Boundary
    subgraph Cluster [K3s / Raspberry Pi Cluster]
        direction LR

        %% Core GitOps Logic
        subgraph GitOps [Continuous Delivery]
            FLUX[Flux CD]
            K8S_Res[(K8s Resources)]
        end

        %% Security / Secret Syncing
        subgraph Security [Secret Engine]
            direction TB
            OB[OpenBao Store] -->|Fetch| ESO[External Secrets]
            ESO -->|Sync| K8S_Sec[(K8s Secrets)]
        end

        %% Routing Layer
        TR[Traefik Ingress]

        %% Target Applications Group
        subgraph Apps [Applications]
            direction LR
            FJ[Forgejo]
            WP[Woodpecker CI]
            REG[Registry]
            BK[BuildKit]
        end
    end

    %% --- CONNECTIONS ---

    %% GitOps Automation Paths
    REPO -->|Reconcile| FLUX
    FLUX --> K8S_Res
    FLUX -->|Deploy| OB

    %% Network Routing Paths
    CF --> TR
    TR --> Apps

    %% Secret Consumptions
    K8S_Sec -.->|Inject| Apps
    K8S_Sec -.->|Auth| OB
```

## Prerequisites

- Device with k3s installed
- Domain with DNS pointing to the node (via Cloudflare Tunnel)
- Forgejo + Woodpecker already running

## Repo structure

```
clusters/default/
├── flux-system/             # Flux bootstrap (auto-generated)
├── forgejo-infra.yaml       # Kustomization: syncs infra/
├── forgejo-eso.yaml         # Kustomization: syncs platform/eso-resources/
├── infra/                   # Applied by forgejo-infra
│   ├── system/              # LimitRange, NetworkPolicy, PVCs
│   ├── platform/
│   │   ├── ingress/         # Traefik Ingress rules + middlewares
│   │   └── eso/             # External Secrets HelmRelease
│   └── apps/
│       ├── openbao/         # Secrets engine (Vault-compatible)
│       ├── forgejo/         # Git server + CI webhooks
│       ├── registry/        # Internal Docker registry
│       ├── woodpecker/      # CI server + agent + buildkitd
│       └── cloudflared/     # Cloudflare Tunnel
└── platform/
    └── eso-resources/       # ClusterSecretStore + ExternalSecrets
```

## Docs

- [Setup guide](./SETUP.md) — full setup steps
- [Operations](./OPERATIONS.md) — maintenance tasks
