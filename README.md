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
│   ├── application-auth-service.yaml        ← Application do ArgoCD (auth-service)
│   ├── application-flag-service.yaml        ← Application do ArgoCD (flag-service)
│   ├── application-targeting-service.yaml   ← Application do ArgoCD (targeting-service)
│   ├── application-evaluation-service.yaml  ← Application do ArgoCD (evaluation-service)
│   └── application-analytics-service.yaml   ← Application do ArgoCD (analytics-service)
│
├── helm/                          ← Um Helm chart independente por microsserviço
│   ├── auth-service/
│   ├── flag-service/
│   ├── targeting-service/
│   ├── evaluation-service/
│   └── analytics-service/         ← inclui KEDA (ScaledObject + TriggerAuthentication)
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

Para registrar todas as aplicações de uma vez:

```bash
kubectl apply -f argocd/
```

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