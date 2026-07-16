# 🚀 Tech Challenge Fase 3 — GitOps com ArgoCD

> ⚠️ **PROJETO DIDÁTICO** — Este projeto foi desenvolvido como parte do Tech Challenge Fase 3 da Pós-Tech FIAP em Arquitetura Cloud e DevOps.

---

## 📋 Sobre o Projeto

Este projeto representa o escopo de **entrega contínua via GitOps** da Fase 3 do Tech Challenge, implementando o padrão GitOps com **ArgoCD** sobre um cluster **Kubernetes**.

O repositório Git é tratado como a **única fonte da verdade** do estado do cluster. Qualquer mudança nos manifests Kubernetes, ao ser enviada para a branch `main`, é detectada e aplicada automaticamente pelo ArgoCD — sem intervenção manual.

### Não fazem parte do escopo deste repositório

- Código-fonte das aplicações
- Build de imagens Docker
- Infraestrutura AWS (VPC, EKS, ECR)
- Pipelines CI/CD de build

---

## 🛠️ Tecnologias Utilizadas

| Tecnologia | Uso |
|---|---|
| **ArgoCD** | Operador GitOps — sincroniza o repositório Git com o cluster |
| **Kubernetes** | Orquestração dos microsserviços |
| **AWS EKS** | Cluster Kubernetes gerenciado na AWS |
| **AWS ECR** | Registry de imagens Docker |
| **Docker** | Containerização das aplicações |
| **GitHub** | Hospedagem do repositório monitorado pelo ArgoCD |

---

## 📁 Estrutura do Repositório

```
fiap-tech-challenge-fase-3-gitops/
│
├── argocd/
│   ├── root.yaml                   ← único Application aplicado manualmente (bootstrap)
│   ├── root/                       ← Application "avó": ordena a sincronização via sync-wave
│   │   ├── root-infra.yaml          (wave -2 — sincroniza antes)
│   │   └── root-applications.yaml   (wave -1)
│   ├── infra/                      ← Applications de plataforma (chart público)
│   │   ├── application-ingress-nginx.yaml
│   │   └── application-keda.yaml
│   └── applications/               ← Applications de negócio (uma por microsserviço)
│       ├── application-auth-service.yaml
│       ├── application-flag-service.yaml
│       ├── application-targeting-service.yaml
│       ├── application-evaluation-service.yaml
│       └── application-analytics-service.yaml
│
├── helm/                          ← Um Helm chart independente por microsserviço
│   ├── auth-service/
│   ├── flag-service/
│   ├── targeting-service/
│   ├── evaluation-service/
│   └── analytics-service/         ← inclui KEDA (ScaledObject + TriggerAuthentication)
│
├── .github/workflows/             ← Workflows reutilizáveis (chamados pelo repo services)
│   ├── update-image.yaml
│   └── update-infra-values.yaml
│
├── RELATORIO.md
└── README.md
```

---

## 🔧 Configuração do ArgoCD

Cada microsserviço tem seu próprio recurso `Application`, apontando para o respectivo Helm chart em `helm/<serviço>`. Isso permite sincronizar, promover ou reverter cada serviço de forma independente.

| Parâmetro | Valor |
|---|---|
| Nome das aplicações | `auth-service`, `flag-service`, `targeting-service`, `evaluation-service`, `analytics-service` |
| Namespace do ArgoCD | `argocd` |
| Repositório monitorado | `https://github.com/nascied/fiap-tech-challenge-fase-3-gitops` |
| Branch monitorada | `main` |
| Pasta do chart | `helm/<serviço>` |
| Cluster de destino | `https://kubernetes.default.svc` |
| Namespace de deploy | `fiap-tc-f4` |
| Sincronização automática | ativada |
| Auto-correção de desvios (`selfHeal`) | ativada |
| Remoção de recursos deletados (`prune`) | ativada |
| Criação automática do namespace | ativada |

Para registrar tudo de uma vez, aplique só a Application raiz — o padrão **App of Apps** cuida do resto:

```bash
kubectl apply -f argocd/root.yaml
```

---

## 🌳 Estrutura App of Apps

Existem três níveis de `Application`:

1. **`argocd/root.yaml`** — único recurso aplicado manualmente. Aponta para a pasta `argocd/root/`.
2. **`argocd/root/`** — duas Applications "avó" com `sync-wave` para garantir ordem: `root-infra` (wave `-2`, sincroniza primeiro) e `root-applications` (wave `-1`, depois). Isso é necessário porque `sync-wave` só é respeitado entre recursos de uma mesma operação de sync — por isso elas precisam estar sob o mesmo `root.yaml`, e não serem aplicadas soltas via `kubectl`.
3. **`argocd/infra/`** e **`argocd/applications/`** — as Applications reais: `ingress-nginx`/`keda` (plataforma, apontam para charts públicos) e os 5 microsserviços (apontam para `helm/<serviço>`).

## ⚙️ Componentes de Plataforma

Além dos 5 microsserviços, este repositório também gerencia dois componentes de infraestrutura do cluster, como charts públicos (sem vendorizar nada):

| Application | Chart | Namespace | Observação |
|---|---|---|---|
| `ingress-nginx` | `kubernetes.github.io/ingress-nginx` | `ingress-nginx` | Controller que atende os 5 `Ingress` dos microsserviços |
| `keda` | `kedacore.github.io/charts` | `keda` | Necessário para o `ScaledObject` do `analytics-service`; usa `ServerSideApply=true` porque o CRD `ScaledJob` é grande demais para o `kubectl apply` padrão (estoura o limite de 256KB de annotation) |

## ⚙️ Workflows Reutilizáveis (`workflow_call`)

Este repositório também expõe dois workflows do GitHub Actions que **não rodam sozinhos** — eles só existem para serem chamados via `uses:` pelas pipelines do repositório `fiap-tech-challenge-fase-3-services`. Por isso, eles nunca aparecem com execução própria na aba *Actions* deste repositório: a run inteira acontece no contexto de quem chama.

| Workflow | Chamado por | O que faz |
|---|---|---|
| `update-image.yaml` | Job `update-gitops` de cada `ci-<serviço>.yaml` (repo `services`), a cada build de imagem aprovado | Atualiza `image.repository`/`image.tag` no `values.yaml` do serviço |
| `update-infra-values.yaml` | `sync-infra-values.yaml` (repo `services`), disparado por `repository_dispatch` vindo do `terraform apply` (repo `fase3`) | Aplica um conjunto dinâmico de campos (`caminho: valor`) no `values.yaml` do serviço, via `yq` — usado para `DATABASE_URL`, endpoint do Redis/SQS/DynamoDB e credenciais AWS |

Ambos exigem o secret `GITOPS_PUSH_TOKEN` (PAT com permissão de escrita neste repositório), configurado no repositório **chamador** (`services`), não aqui — já que é lá que o job efetivamente executa.

---

## 🧩 Microsserviços

Cada pasta dentro de `helm/` é um chart Helm completo (Deployment, Service, ConfigMap, Secret, HPA e, no caso do `analytics-service`, `ScaledObject`/`TriggerAuthentication` do KEDA para escalonamento por fila SQS). A tag da imagem de cada serviço fica em `helm/<serviço>/values.yaml` (`image.repository` / `image.tag`).

| Serviço | Porta | Autoscaling |
|---|---|---|
| `auth-service` | 8001 | HPA (CPU/memória) |
| `flag-service` | 8002 | HPA (CPU/memória) |
| `targeting-service` | 8003 | HPA (CPU/memória) |
| `evaluation-service` | 8004 | HPA (CPU/memória) |
| `analytics-service` | 8005 | KEDA (fila SQS) |

---

## 🔄 Como Funciona o Deploy Automático

```
Desenvolvedor edita deployment.yaml (ex: nova versão de imagem)
        │
        │  git push origin main
        ▼
GitHub — branch main atualizada
        │
        │  ArgoCD detecta divergência (polling a cada ~3 min)
        ▼
Status: OutOfSync
        │
        │  kubectl apply automático
        ▼
Status: Synced ✅ — cluster atualizado sem nenhuma intervenção manual
```

---

## 🧪 Como Demonstrar o GitOps em Funcionamento

**1. Instalar o ArgoCD no cluster**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Aguarde todos os pods ficarem com status `Running`:

```bash
kubectl get pods -n argocd
```

**2. Registrar as aplicações**

```bash
kubectl apply -f argocd/
```

**3. Abrir o painel do ArgoCD**

Execute e mantenha o terminal aberto:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Acesse `http://localhost:8080`. Usuário: `admin`. Para obter a senha:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}"
```

Decodifique o resultado em um segundo terminal:

```bash
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("COLE_AQUI"))
```

**4. Simular uma atualização**

Edite `helm/auth-service/values.yaml` e altere a tag da imagem:

```yaml
# antes
image:
  tag: auth-service-v1

# depois
image:
  tag: auth-service-v2
```

Envie a mudança:

```bash
git add .
git commit -m "update auth-service para auth-service-v2"
git push origin main
```

Observe o cartão `auth-service` no painel do ArgoCD mudar para **OutOfSync** e depois **Synced** automaticamente.

**5. Verificar os serviços no cluster**

```bash
kubectl get pods -n fiap-tc-f4
kubectl get services -n fiap-tc-f4
```

---

## 📌 Melhorias Identificadas (histórico, já resolvidas na versão Helm)

A versão anterior deste repositório usava manifests YAML crus por serviço. Os pontos abaixo foram identificados nessa versão e resolvidos com a migração para Helm chart por microsserviço:

| Problema identificado | Situação atual |
|---|---|
| `apiVersion` ausente em alguns manifests (`analytics/deployment.yaml`, `analytics/service.yaml`) | Resolvido — todos os templates Helm declaram `apiVersion` |
| `service.yaml` ausente em `evaluation-service`/`flag-service` | Resolvido — todo chart tem `templates/service.yaml` |
| `deployment.yaml` ausente em `targeting-service` | Resolvido — todo chart tem `templates/deployment.yaml` |
| Imagens com tags locais (`auth-service:v1.0.0`, sem registry) | Resolvido — `image.repository` em cada `values.yaml` referencia o ECR real |

---

## ✅ Checklist de Conformidade GitOps

| Critério | Status |
|---|---|
| Git como única fonte da verdade | ✅ |
| Deploy automático via ArgoCD | ✅ |
| Auto-correção de desvios no cluster (`selfHeal`) | ✅ |
| Remoção automática de recursos deletados (`prune`) | ✅ |
| Namespace criado automaticamente | ✅ |
| Todos os manifests com `apiVersion` | ✅ |
| Todos os serviços com `Deployment` + `Service` | ✅ |
| Imagens referenciando registry externo (ECR) | ✅ |

---

## 👨‍💻 Autor

**Renan**
Pós-Tech FIAP — Arquitetura Cloud e DevOps
Tech Challenge Fase 3

---

## 📄 Licença

Este projeto é desenvolvido exclusivamente para fins educacionais como parte do programa de pós-graduação em Arquitetura Cloud e DevOps da **FIAP**.