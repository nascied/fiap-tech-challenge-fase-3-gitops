# GitOps - FIAP Tech Challenge Fase 3

Repositório GitOps monitorado pelo ArgoCD para deploy automático no cluster Kubernetes.

## Estrutura

```
gitops/
??? auth/
?   ??? deployment.yaml
?   ??? service.yaml
??? flag/
?   ??? deployment.yaml
?   ??? service.yaml
??? targeting/
?   ??? deployment.yaml
?   ??? service.yaml
??? evaluation/
?   ??? deployment.yaml
?   ??? service.yaml
??? analytics/
    ??? deployment.yaml
    ??? service.yaml
argocd/
??? application.yaml
```

---

## Pré-requisitos

- Cluster Kubernetes rodando (EKS, Minikube ou Kind)
- `kubectl` configurado apontando para o cluster

---

## TUTORIAL PARA O VÍDEO

### PASSO 1 — Instalar o ArgoCD no cluster

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Aguardar os pods ficarem prontos:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=120s
```

---

### PASSO 2 — Acessar a interface web do ArgoCD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Abrir no navegador: **http://localhost:8080**

Usuário: `admin`

Pegar a senha:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
```

---

### PASSO 3 — Criar a Application no ArgoCD

```bash
kubectl apply -f argocd/application.yaml
```

Isso faz o ArgoCD monitorar este repositório e fazer o deploy automático de todos os serviços na pasta `gitops/`.

---

### PASSO 4 — Verificar o deploy

```bash
kubectl get pods -n fiap
kubectl get services -n fiap
```

---

### PASSO 5 — Demonstrar o GitOps em ação (para o vídeo)

Edite `gitops/auth/deployment.yaml` e mude a versão da imagem:
```yaml
image: auth-service:v1.0.1
```

Depois faça o commit e push:

```bash
git add .
git commit -m "update auth-service to v1.0.1"
git push
```

O ArgoCD detecta a mudança automaticamente e aplica no cluster em até 3 minutos.
No painel web você verá o status mudar de **Synced** ? **OutOfSync** ? **Synced**.

---

## Fluxo Completo

```
Edson (Terraform)      Sandro (CI/CD)           Renan (GitOps/ArgoCD)
Cria EKS + ECR  ?  Build + Push imagem  ?  ArgoCD detecta mudança
                   Atualiza este repo   ?  Deploy automático no EKS
```
