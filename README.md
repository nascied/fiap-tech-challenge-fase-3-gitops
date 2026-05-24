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
│   └── application.yaml          ← Recurso Application do ArgoCD
│
├── gitops/                        ← Manifests Kubernetes observados pelo ArgoCD
│   ├── analytics/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── auth/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── evaluation/
│   │   └── deployment.yaml
│   ├── flag/
│   │   └── deployment.yaml
│   └── targeting/
│       └── service.yaml
│
├── RELATORIO.md
└── README.md
```

---

## 🔧 Configuração do ArgoCD

O arquivo `argocd/application.yaml` define como o ArgoCD monitora e sincroniza este repositório com o cluster.

| Parâmetro | Valor |
|---|---|
| Nome da aplicação | `fiap-app` |
| Namespace do ArgoCD | `argocd` |
| Repositório monitorado | `https://github.com/nascied/fiap-tech-challenge-fase-3-gitops` |
| Branch monitorada | `main` |
| Pasta dos manifests | `gitops/` |
| Cluster de destino | `https://kubernetes.default.svc` |
| Namespace de deploy | `fiap` |
| Sincronização automática | ativada |
| Auto-correção de desvios (`selfHeal`) | ativada |
| Remoção de recursos deletados (`prune`) | ativada |
| Criação automática do namespace | ativada |

---

## 🧩 Microsserviços

Cada subpasta dentro de `gitops/` representa um microsserviço. Todos rodam na porta `8080` internamente e são expostos dentro do cluster via `ClusterIP`.

| Serviço | Deployment | Service |
|---|---|---|
| `analytics-service` | ✅ | ✅ |
| `auth-service` | ✅ | ✅ |
| `evaluation-service` | ✅ | ⚠️ ausente |
| `flag-service` | ✅ | ⚠️ ausente |
| `targeting-service` | ⚠️ ausente | ✅ |

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

**2. Registrar a aplicação**

```bash
kubectl apply -f argocd/application.yaml
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

Edite `gitops/auth/deployment.yaml` e altere a versão da imagem:

```yaml
# antes
image: auth-service:v1.0.0

# depois
image: auth-service:v1.0.1
```

Envie a mudança:

```bash
git add .
git commit -m "update auth-service para v1.0.1"
git push origin main
```

Observe o cartão `fiap-app` no painel do ArgoCD mudar para **OutOfSync** e depois **Synced** automaticamente.

**5. Verificar os serviços no cluster**

```bash
kubectl get pods -n fiap
kubectl get services -n fiap
```

---

## 📌 Melhorias Identificadas

### 1. `apiVersion` ausente em alguns manifests

Os arquivos `analytics/deployment.yaml` e `analytics/service.yaml` estão sem o campo `apiVersion`, que é obrigatório para o Kubernetes processar o recurso corretamente.

| Tipo de recurso | `apiVersion` correto |
|---|---|
| Deployment | `apps/v1` |
| Service | `v1` |
| Application (ArgoCD) | `argoproj.io/v1alpha1` |

### 2. Manifests faltantes

| Serviço | O que está ausente |
|---|---|
| `evaluation-service` | `service.yaml` |
| `flag-service` | `service.yaml` |
| `targeting-service` | `deployment.yaml` |

### 3. Imagens sem registry externo

As imagens estão definidas com tags locais, por exemplo `auth-service:v1.0.0`. Em produção devem referenciar o registry real no ECR da AWS:

```yaml
image: <account>.dkr.ecr.<region>.amazonaws.com/auth-service:v1.0.0
```

---

## ✅ Checklist de Conformidade GitOps

| Critério | Status |
|---|---|
| Git como única fonte da verdade | ✅ |
| Deploy automático via ArgoCD | ✅ |
| Auto-correção de desvios no cluster (`selfHeal`) | ✅ |
| Remoção automática de recursos deletados (`prune`) | ✅ |
| Namespace criado automaticamente | ✅ |
| Todos os manifests com `apiVersion` | ⚠️ Parcial |
| Todos os serviços com `Deployment` + `Service` | ⚠️ Parcial |
| Imagens referenciando registry externo (ECR) | ⚠️ Pendente |

---

## 👨‍💻 Autor

**Renan**
Pós-Tech FIAP — Arquitetura Cloud e DevOps
Tech Challenge Fase 3

---

## 📄 Licença

Este projeto é desenvolvido exclusivamente para fins educacionais como parte do programa de pós-graduação em Arquitetura Cloud e DevOps da **FIAP**.