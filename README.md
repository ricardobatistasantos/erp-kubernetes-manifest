# ERP Modular Monolith — Kubernetes Manifests

Repositório de manifests Kubernetes para o ERP Modular Monolith. Contém toda a infraestrutura declarativa necessária para deploy em clusters locais (Minikube, k3d, kind).

---

## Sumário

- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Estrutura do Repositório](#estrutura-do-repositório)
- [Criação do Cluster Local](#criação-do-cluster-local)
- [Deploy Passo a Passo](#deploy-passo-a-passo)
- [Configuração de DNS Local](#configuração-de-dns-local)
- [Serviços Externos (Docker Compose)](#serviços-externos-docker-compose)
- [Descrição dos Componentes](#descrição-dos-componentes)
- [Variáveis de Ambiente](#variáveis-de-ambiente)
- [Segurança](#segurança)
- [Observabilidade](#observabilidade)
- [Autoscaling](#autoscaling)
- [Troubleshooting](#troubleshooting)
- [Adicionando um Novo Worker](#adicionando-um-novo-worker)

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                    Cluster Kubernetes                       │
│                                                             │
│  ┌─────────────────────┐     ┌──────────────────────┐       │
│  │  Namespace:         │     │  Namespace:          │       │
│  │  modular-monolith   │     │  mail-worker         │       │
│  │                     │     │                      │       │
│  │  ┌─────────────────┐│     │  ┌─────────────────┐ │       │
│  │  │ Deployment      ││     │  │ Deployment      │ │       │
│  │  │ modular-monolith││     │  │ mail-worker     │ │       │
│  │  │ :3000           ││     │  │ :3001           │ │       │
│  │  └─────────────────┘│     │  └─────────────────┘ │       │
│  └─────────────────────┘     └──────────────────────┘       │
│                                                             │
│  ┌─────────────────────┐                                    │
│  │  Namespace:         │                                    │
│  │  auth-worker        │                                    │
│  │                     │                                    │
│  │  ┌────────────────┐ │                                    │
│  │  │ Deployment     │ │                                    │
│  │  │ auth-worker    │ │                                    │
│  │  │ :3002          │ │                                    │
│  │  └────────────────┘ │                                    │
│  └─────────────────────┘                                    │
│                                                             │
│  ┌─────────────────────┐                                    │
│  │  Ingress (nginx)    │                                    │
│  │  erp-monolith.local │ ──▶ modular-monolith:3000          │
│  └─────────────────────┘                                    │
└─────────────────────────────────────────────────────────────┘
         │
         ▼ (host.minikube.internal)
┌─────────────────────────────────────────────────────────────┐
│                  Máquina Host                               │
│                                                             │
│  PostgreSQL :5432  │  MongoDB :27017  │  RabbitMQ :5672     │
│  Redis :6379       │  LocalStack :4566                      │
└─────────────────────────────────────────────────────────────┘
```

**Fluxo:**
1. Requisições HTTP chegam pelo Ingress em `erp-monolith.local`
2. O Ingress encaminha para o Service `modular-monolith` na porta 3000
3. A aplicação se comunica com serviços externos (Postgres, MongoDB, RabbitMQ) via `host.minikube.internal`
4. O `mail-worker` consome filas do RabbitMQ/Redis e dispara emails via SMTP
5. O `auth-worker` processa tarefas de autenticação consumindo filas (porta 3002)

---

## Pré-requisitos

| Ferramenta | Versão mínima | Finalidade |
|------------|---------------|------------|
| Docker | 20.10+ | Runtime de containers |
| Minikube / k3d / kind | Minikube 1.30+ | Cluster local |
| kubectl | 1.27+ | CLI para gerenciar o cluster |
| Docker Compose | 2.0+ | Serviços externos (Redis) |

### Instalação rápida (Ubuntu/Debian)

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

---

## Estrutura do Repositório

```
erp-kubernetes-manifest/
├── docker/                          # Docker Compose para serviços externos
│   ├── docker-compose.yml           # Redis
│   └── redis/
│       ├── Dockerfile
│       └── redis.conf
├── hosts-config/
│   └── etc/hosts                    # Entrada DNS local de referência
├── k8s/
│   ├── cluster/                     # Recursos compartilhados do cluster
│   │   ├── cluster-config/          # Configuração dos clusters locais
│   │   │   ├── k3d.yml
│   │   │   ├── kind.yml
│   │   │   └── minikube.md
│   │   ├── ingress/                 # Ingress do ERP
│   │   │   └── erp-service.yml
│   │   └── volume/                  # PV, PVCs e StorageClass
│   │       ├── storageclass-standard.yml
│   │       ├── volume-pv.yml
│   │       ├── volume-postgre-pvc.yml
│   │       └── pvc-rabbitmq.yml
│   ├── modular-monolith/            # API principal do ERP
│   │   ├── namespace/
│   │   ├── service-account/
│   │   ├── config-map/
│   │   ├── secrets/
│   │   ├── deployments/
│   │   ├── hpa/
│   │   ├── pdb/
│   │   ├── limit-range/
│   │   ├── resource-quota/
│   │   └── network-policy/
│   ├── email-worker/                # Worker de envio de emails
│   │   ├── namespace/
│   │   ├── service-account/
│   │   ├── config-map/
│   │   ├── secrets/
│   │   ├── deployments/
│   │   ├── hpa/
│   │   └── network-policy/
│   ├── auth-worker/                 # Worker de autenticação
│   │   ├── namespace/
│   │   ├── service-account/
│   │   ├── config-map/
│   │   ├── secrets/
│   │   ├── deployments/
│   │   ├── hpa/
│   │   └── volumes/
│   └── sped-php-worker/             # Worker SPED fiscal (scaffold)
└── README.md
```

---

## Criação do Cluster Local

Escolha **uma** das opções abaixo:

### Opção 1: Minikube (recomendado para desenvolvimento)

```bash
minikube start --profile=minikube-erp-modular-cluster --driver=docker

# Habilitar addons necessários
minikube addons enable ingress --profile=minikube-erp-modular-cluster
minikube addons enable metrics-server --profile=minikube-erp-modular-cluster
```

### Opção 2: k3d

```bash
k3d cluster create --config k8s/cluster/cluster-config/k3d.yml

# Instalar NGINX Ingress Controller (Traefik desabilitado no config)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### Opção 3: kind

```bash
kind create cluster --config k8s/cluster/cluster-config/kind.yml

# Instalar NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

---

## Deploy Passo a Passo

### 1. Subir serviços externos (Redis)

```bash
cd docker
docker compose up -d
cd ..
```

### 2. Carregar imagens Docker no cluster (Minikube)

```bash
# Construa as imagens do projeto (assumindo que estão em outro repo)
minikube image load modular-monolith:1.0.0 --profile minikube-erp-modular-cluster
minikube image load erp-modular/mail-worker:1.0.0 --profile minikube-erp-modular-cluster
minikube image load erp-modular/auth-worker:1.0.0 --profile minikube-erp-modular-cluster
```

### 3. Aplicar manifests na ordem correta

```bash
# --- Recursos de cluster ---
kubectl apply -f k8s/cluster/volume/storageclass-standard.yml
kubectl apply -f k8s/cluster/volume/volume-pv.yml

# --- Modular Monolith ---
kubectl apply -f k8s/modular-monolith/namespace/
kubectl apply -f k8s/modular-monolith/service-account/
kubectl apply -f k8s/modular-monolith/config-map/
kubectl apply -f k8s/modular-monolith/secrets/
kubectl apply -f k8s/modular-monolith/network-policy/
kubectl apply -f k8s/modular-monolith/limit-range/
kubectl apply -f k8s/modular-monolith/resource-quota/
kubectl apply -f k8s/modular-monolith/deployments/
kubectl apply -f k8s/modular-monolith/hpa/
kubectl apply -f k8s/modular-monolith/pdb/

# --- PVCs (dependem do namespace existir) ---
kubectl apply -f k8s/cluster/volume/volume-postgre-pvc.yml
kubectl apply -f k8s/cluster/volume/pvc-rabbitmq.yml

# --- Email Worker ---
kubectl apply -f k8s/email-worker/namespace/
kubectl apply -f k8s/email-worker/service-account/
kubectl apply -f k8s/email-worker/config-map/
kubectl apply -f k8s/email-worker/secrets/
kubectl apply -f k8s/email-worker/network-policy/
kubectl apply -f k8s/email-worker/deployments/
kubectl apply -f k8s/email-worker/hpa/

# --- Auth Worker ---
kubectl apply -f k8s/auth-worker/namespace/
kubectl apply -f k8s/auth-worker/service-account/
kubectl apply -f k8s/auth-worker/config-map/
kubectl apply -f k8s/auth-worker/secrets/
kubectl apply -f k8s/auth-worker/network-policy/
kubectl apply -f k8s/auth-worker/deployments/
kubectl apply -f k8s/auth-worker/hpa/

# --- Ingress ---
kubectl apply -f k8s/cluster/ingress/
```

### 4. Verificar o deploy

```bash
# Status dos pods
kubectl get pods -n modular-monolith
kubectl get pods -n mail-worker
kubectl get pods -n auth-worker

# Verificar se estão healthy
kubectl describe deployment modular-monolith -n modular-monolith
kubectl describe deployment mail-worker -n mail-worker
kubectl describe deployment auth-worker -n auth-worker

# Logs
kubectl logs -f deployment/modular-monolith -n modular-monolith
kubectl logs -f deployment/mail-worker -n mail-worker
kubectl logs -f deployment/auth-worker -n auth-worker
```

---

## Configuração de DNS Local

Para acessar o serviço via browser, adicione ao seu `/etc/hosts`:

```bash
# Obter IP do Minikube
minikube ip --profile=minikube-erp-modular-cluster

# Adicionar ao /etc/hosts (substitua <MINIKUBE_IP>)
echo "<MINIKUBE_IP> erp-monolith.local" | sudo tee -a /etc/hosts
```

Para k3d/kind com port-forward:
```bash
kubectl port-forward svc/modular-monolith 3000:3000 -n modular-monolith
# Acesse em http://localhost:3000
```

---

## Serviços Externos (Docker Compose)

O `docker-compose.yml` sobe o Redis que é utilizado pelo `mail-worker`.

```bash
cd docker
docker compose up -d    # Subir
docker compose down     # Parar
docker compose logs -f  # Acompanhar logs
```

**Serviços esperados no host (não gerenciados por este repo):**

| Serviço | Porta | Propósito |
|---------|-------|-----------|
| PostgreSQL | 5432 | Banco relacional do ERP |
| MongoDB | 27017 | Banco de documentos |
| RabbitMQ | 5672 | Message broker (eventos) |
| Redis | 6379 | Cache e fila do mail-worker |
| LocalStack | 4566 | Simulação AWS S3 local |

Esses serviços devem estar rodando na máquina host. O Kubernetes acessa via `host.minikube.internal`.

---

## Descrição dos Componentes

### modular-monolith

API principal do ERP. Expõe porta 3000 e se comunica com PostgreSQL, MongoDB, RabbitMQ e S3 (LocalStack).

| Recurso | Descrição |
|---------|-----------|
| Deployment | 1 réplica, imagem `modular-monolith:1.0.0` |
| Service | ClusterIP na porta 3000 |
| HPA | 1-5 réplicas (CPU 70%, Memory 80%) |
| PDB | Mínimo 1 pod disponível |
| LimitRange | Default: 500m CPU / 512Mi mem |
| ResourceQuota | Max: 2 CPU / 2Gi mem / 5 pods |
| NetworkPolicy | Deny-all + allow ingress:3000 + allow egress |
| ServiceAccount | `modular-monolith-sa` (sem auto-mount de token) |

### mail-worker

Worker que consome filas e envia emails via SMTP (Gmail). Expõe porta 3001 para health checks.

| Recurso | Descrição |
|---------|-----------|
| Deployment | 1 réplica, imagem `erp-modular/mail-worker:1.0.0` |
| Service | ClusterIP na porta 3001 |
| HPA | 1-3 réplicas (CPU 70%, Memory 80%) |
| NetworkPolicy | Deny-all + allow egress |
| ServiceAccount | `mail-worker-sa` (sem auto-mount de token) |

### auth-worker

Worker responsável pelo processamento de tarefas de autenticação. Expõe porta 3002 para health checks.

| Recurso | Descrição |
|---------|-----------|
| Deployment | 1 réplica, imagem `erp-modular/auth-worker:1.0.0` |
| Service | ClusterIP na porta 3002 |
| ServiceAccount | `auth-worker-sa` (sem auto-mount de token) |

### sped-php-worker

Scaffold preparado para futura implementação. Apenas estrutura de pastas com `.gitkeep`.

---

## Variáveis de Ambiente

### modular-monolith (ConfigMap)

| Variável | Valor | Descrição |
|----------|-------|-----------|
| NODE_PORT | 3000 | Porta da aplicação |
| NODE_ENV | local | Ambiente de execução |
| AMQP_EXCHANGE | erp.exchange | Exchange do RabbitMQ |
| POSTGRES_HOST | host.minikube.internal | Host do PostgreSQL |
| POSTGRES_PORT | 5432 | Porta do PostgreSQL |
| AWS_REGION | us-east-1 | Região AWS (LocalStack) |
| BUCKET_NAME | meu-bucket-local | Bucket S3 |
| AWS_S3_ENDPOINT | http://host.minikube.internal:4566 | Endpoint LocalStack |
| S3_PREFIX | erp | Prefixo dos objetos S3 |

### modular-monolith (Secrets)

| Variável | Descrição |
|----------|-----------|
| POSTGRES_USER | Usuário do PostgreSQL |
| POSTGRES_PASSWORD | Senha do PostgreSQL |
| POSTGRES_DB | Nome do banco |
| RABBITMQ_DEFAULT_USER | Usuário do RabbitMQ |
| RABBITMQ_DEFAULT_PASS | Senha do RabbitMQ |
| AMQP_URL | URL de conexão completa do RabbitMQ |
| MONGO_URL | URL de conexão completa do MongoDB |

### mail-worker (ConfigMap)

| Variável | Valor | Descrição |
|----------|-------|-----------|
| REDIS_HOST | host.minikube.internal | Host do Redis |
| REDIS_PORT | 6379 | Porta do Redis |
| GMAIL_HOST | smtp.gmail.com | Servidor SMTP |
| GMAIL_PORT | 587 | Porta SMTP (TLS) |

### mail-worker (Secrets)

| Variável | Descrição |
|----------|-----------|
| REDIS_PASSWORD | Senha de acesso ao Redis |
| GMAIL_USER | Email remetente |
| GMAIL_PASSWORD | App password do Gmail |
| GMAIL_SECURE | Flag de TLS (false = STARTTLS) |

---

## Segurança

### Práticas implementadas

- **SecurityContext no pod**: `runAsNonRoot: true`, `runAsUser: 1000`, `fsGroup: 1000`
- **SecurityContext no container**: `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`
- **ServiceAccount dedicada** por workload com `automountServiceAccountToken: false`
- **NetworkPolicies**: Default deny-all com allow explícito apenas para tráfego necessário
- **Secrets em base64**: Nenhuma credencial em texto puro nos manifests
- **Imagens com tag fixa**: Sem uso de `:latest` para garantir reprodutibilidade

### Recomendações adicionais para produção

- Usar um gerenciador de secrets externo (Vault, AWS Secrets Manager, Sealed Secrets)
- Habilitar RBAC granular por ServiceAccount
- Adicionar PodSecurityStandards (Restricted) no namespace
- Implementar scanning de vulnerabilidades nas imagens (Trivy, Snyk)
- Usar assinatura de imagens (Cosign/Sigstore)

---

## Observabilidade

### Prometheus

Os Services possuem annotations para scraping automático:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "3000"  # ou 3001 para mail-worker
```

### Health Checks

Ambos os deployments possuem probes configuradas:

| Probe | modular-monolith | mail-worker | auth-worker |
|-------|-----------------|-------------|-------------|
| startupProbe | GET / :3000 (fail: 30x, interval: 2s) | GET /health :3001 (fail: 30x, interval: 2s) | GET /health :3002 (fail: 30x, interval: 2s) |
| readinessProbe | GET / :3000 (delay: 5s, interval: 10s) | GET /health :3001 (delay: 5s, interval: 10s) | GET /health :3002 (delay: 5s, interval: 10s) |
| livenessProbe | GET / :3000 (delay: 15s, interval: 20s) | GET /health :3001 (delay: 15s, interval: 20s) | GET /health :3002 (delay: 15s, interval: 20s) |

### Logs

```bash
# Em tempo real
kubectl logs -f deployment/modular-monolith -n modular-monolith
kubectl logs -f deployment/mail-worker -n mail-worker
kubectl logs -f deployment/auth-worker -n auth-worker

# Últimas 100 linhas
kubectl logs --tail=100 deployment/modular-monolith -n modular-monolith
```

---

## Autoscaling

O HPA (Horizontal Pod Autoscaler) está configurado para ambos os workloads:

| Workload | Min | Max | CPU Threshold | Memory Threshold |
|----------|-----|-----|---------------|-----------------|
| modular-monolith | 1 | 5 | 70% | 80% |
| mail-worker | 1 | 3 | 70% | 80% |
| auth-worker | — | — | — | — |

**Requisito**: O `metrics-server` deve estar habilitado no cluster.

```bash
# Minikube
minikube addons enable metrics-server --profile=minikube-erp-modular-cluster

# Verificar HPA
kubectl get hpa -n modular-monolith
kubectl get hpa -n mail-worker
kubectl get hpa -n auth-worker
```

---

## Troubleshooting

### Pod em CrashLoopBackOff

```bash
# Ver eventos do pod
kubectl describe pod <pod-name> -n <namespace>

# Verificar logs da última execução
kubectl logs <pod-name> -n <namespace> --previous
```

**Causas comuns:**
- Serviço externo (Postgres, RabbitMQ) não está rodando no host
- Imagem não carregada no Minikube (`minikube image load`)
- Secret/ConfigMap não existe no namespace

### Pod em Pending

```bash
kubectl describe pod <pod-name> -n <namespace>
```

**Causas comuns:**
- ResourceQuota excedida (max 5 pods / 2 CPU)
- PVC não bound (verificar StorageClass)

### Ingress não responde

```bash
# Verificar se o ingress controller está rodando
kubectl get pods -n ingress-nginx

# Verificar se o ingress está configurado
kubectl describe ingress erp-modular-ingress -n modular-monolith

# Testar acesso direto ao service
kubectl port-forward svc/modular-monolith 3000:3000 -n modular-monolith
curl http://localhost:3000
```

### Serviço externo inacessível

```bash
# Testar conectividade de dentro do pod
kubectl exec -it deployment/modular-monolith -n modular-monolith -- sh
# Dentro do pod:
wget -qO- http://host.minikube.internal:5432 || echo "Postgres inacessível"
```

**Nota**: `host.minikube.internal` resolve para o IP da máquina host. Verifique se os serviços estão escutando em `0.0.0.0` (não apenas `127.0.0.1`).

---

## Adicionando um Novo Worker

Para criar um novo worker (ex: `auth-worker`), siga a estrutura padrão:

```bash
k8s/auth-worker/
├── namespace/auth-worker.yml
├── service-account/auth-worker-sa.yml
├── config-map/auth-worker-config.yml
├── secrets/auth-worker-secrets.yml
├── deployments/auth-worker.yml
├── hpa/hpa.yml
└── network-policy/deny-all.yml
```

**Checklist para cada novo worker:**

- [ ] Namespace com labels `app.kubernetes.io/name` e `environment: dev`
- [ ] ServiceAccount com `automountServiceAccountToken: false`
- [ ] Deployment com:
  - [ ] `securityContext` no pod e container
  - [ ] `envFrom` com configMapRef e secretRef
  - [ ] `resources` (requests + limits)
  - [ ] Health probes (startup, readiness, liveness)
  - [ ] Tag de imagem versionada (não usar `:latest`)
  - [ ] Labels `app.kubernetes.io/*`
- [ ] Service com annotation Prometheus
- [ ] HPA configurado
- [ ] NetworkPolicy deny-all + allow explícito
- [ ] Imagem carregada no cluster (`minikube image load`)

---

## Comandos Úteis

```bash
# Ver todos os recursos em um namespace
kubectl get all -n modular-monolith

# Deletar todos os recursos do projeto
kubectl delete -f k8s/cluster/ingress/
kubectl delete -f k8s/auth-worker/deployments/
kubectl delete -f k8s/email-worker/deployments/
kubectl delete -f k8s/modular-monolith/deployments/
kubectl delete ns modular-monolith mail-worker auth-worker

# Restart de um deployment
kubectl rollout restart deployment/modular-monolith -n modular-monolith

# Ver uso de recursos
kubectl top pods -n modular-monolith

# Verificar quotas
kubectl describe resourcequota -n modular-monolith
```

---

## Licença

Uso interno — ERP Modular Monolith.
