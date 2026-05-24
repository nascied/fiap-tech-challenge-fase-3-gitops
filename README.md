# GitOps + ArgoCD - FIAP Tech Challenge Fase 3

---

















## O QUE VOCE PRECISA INSTALAR

**1. Docker Desktop**
- Baixe em: https://www.docker.com/products/docker-desktop/
- Instale, reinicie o PC se pedir
- Abra o Docker Desktop, va em Settings > Kubernetes, marque **Enable Kubernetes**, clique em **Apply & Restart**
- Aguarde o circulo verde no canto inferior esquerdo — quando ficar verde, esta pronto

**2. Git**
- Baixe em: https://git-scm.com/download/win
- Instale clicando em Next em tudo

---

## COMO EXECUTAR

Abra o **PowerShell** (tecla Iniciar > pesquise PowerShell > abra)

**1. Baixar o repositorio**
```
git clone https://github.com/nascied/fiap-tech-challenge-fase-3-gitops.git
cd fiap-tech-challenge-fase-3-gitops
```

**2. Instalar o ArgoCD**
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Aguarde 5 minutos e verifique se todos estao com STATUS `Running`:
```
kubectl get pods -n argocd
```

**3. Abrir o painel do ArgoCD no navegador**

Execute e deixe o PowerShell aberto:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Abra um **segundo PowerShell** para os proximos comandos.

Acesse no Chrome: **http://localhost:8080**
> Se aparecer aviso de seguranca clique em Avancado > Prosseguir para localhost

Login: usuario `admin`, para pegar a senha execute no segundo PowerShell:
```
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}"
```
Copie o resultado, depois execute trocando `COLE_AQUI` pelo texto copiado:
```
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("COLE_AQUI"))
```
O resultado e a senha. Cole no campo Password e clique em Sign In.

**4. Conectar o ArgoCD a este repositorio**
```
kubectl apply -f argocd/application.yaml
```
No navegador aparece o cartao **fiap-app** com todos os servicos.

**5. Confirmar os servicos rodando**
```
kubectl get pods -n fiap
kubectl get services -n fiap
```

---

## DEMONSTRAR O GITOPS

Abra o arquivo `gitops\auth\deployment.yaml` no Bloco de Notas e mude:
```
image: auth-service:v1.0.0
```
para:
```
image: auth-service:v1.0.1
```
Salve com Ctrl+S. Depois no segundo PowerShell:
```
git add .
git commit -m "update auth-service para v1.0.1"
git push origin main
```
Olhe o navegador. Em ate 3 minutos o ArgoCD detecta a mudanca e atualiza o cluster sozinho.
O status vai mudar: **OutOfSync** e depois **Synced** automaticamente.

---

## FLUXO DO TRABALHO

```
Edson (Terraform)  -->  Sandro (CI/CD)          -->  Renan (ArgoCD)
Cria EKS + ECR         Build, push da imagem         ArgoCD detecta mudanca
na AWS                 Atualiza deployment.yaml       Deploy automatico no cluster
```
