# GitOps - FIAP Tech Challenge Fase 3
# GitOps - FIAP Tech Challenge Fase 3

---

## PARTE 1 - INSTALAR OS PROGRAMAS NO SEU COMPUTADOR

Voce precisa instalar 2 programas antes de comecar.

---

### Programa 1 - Docker Desktop

O Docker Desktop vai criar um Kubernetes no seu proprio computador.
Sempre que o trabalho falar "cluster", e isso: o Kubernetes rodando no seu PC.

**Como instalar:**

1. Abra o Chrome e entre no site: https://www.docker.com/products/docker-desktop/
2. Clique no botao de download para Windows
3. Quando terminar o download, abra a pasta Downloads e de dois cliques no arquivo `Docker Desktop Installer.exe`
4. Clique em OK em tudo que aparecer ate terminar de instalar
5. Se pedir para reiniciar o computador, reinicie
6. Apos reiniciar, clique no botao Iniciar, pesquise por `Docker Desktop` e abra
7. Aguarde a tela principal aparecer (pode demorar 2 minutos)
8. Clique no icone de engrenagem no canto superior direito da tela (e o icone de Settings)
9. No menu do lado esquerdo, clique em `Kubernetes`
10. Marque a caixinha `Enable Kubernetes`
11. Clique no botao azul `Apply & Restart`
12. Aguarde de 3 a 5 minutos
13. Quando aparecer um circulo VERDE no canto inferior esquerdo do Docker Desktop, o Kubernetes esta pronto

---

### Programa 2 - Git

O Git serve para enviar arquivos daqui para o GitHub.

**Como instalar:**

1. Abra o Chrome e entre no site: https://git-scm.com/download/win
2. O download comeca automaticamente
3. Quando terminar, abra a pasta Downloads e de dois cliques no arquivo `Git-x.xx.x-64-bit.exe`
4. Clique em Next em todas as telas sem mudar nada
5. Clique em Install e depois em Finish

---

## PARTE 2 - CONFIRMAR QUE TUDO ESTA FUNCIONANDO

**Como abrir o PowerShell** (a janela preta onde voce digita os comandos):

1. Clique no botao Iniciar (canto inferior esquerdo da tela)
2. Pesquise por `PowerShell`
3. Clique em `Windows PowerShell`
4. Uma janela preta vai abrir

Digite cada comando abaixo e aperte Enter. So continue se aparecer o resultado esperado.

**Testar o Git:**
```
git --version
```
Resultado esperado: `git version 2.xx.x`

**Testar o kubectl:**
```
kubectl version --client
```
Resultado esperado: `Client Version: v1.xx.x`

**Testar o Kubernetes:**
```
kubectl get nodes
```
Resultado esperado:
```
NAME             STATUS   ROLES
docker-desktop   Ready    control-plane
```
Se a coluna STATUS mostrar `Ready`, o Kubernetes esta funcionando. Pode continuar.

---

## PARTE 3 - TUTORIAL DO VIDEO (execute tudo em ordem)

---

### PASSO 1 - Baixar este repositorio para o seu computador

No PowerShell, execute os comandos abaixo um por vez:

```
cd C:\
```

```
git clone https://github.com/nascied/fiap-tech-challenge-fase-3-gitops.git
```

```
cd fiap-tech-challenge-fase-3-gitops
```

```
git checkout GR_20260524
```

Pronto. Os arquivos do projeto agora estao na pasta `C:\fiap-tech-challenge-fase-3-gitops`

---

### PASSO 2 - Instalar o ArgoCD no Kubernetes

O ArgoCD e um programa que fica observando este repositorio no GitHub.
Quando voce muda um arquivo aqui, ele aplica essa mudanca no Kubernetes do seu PC sozinho.
Ele tambem tem uma tela visual no navegador para mostrar no video.

Execute os comandos abaixo no PowerShell, um por vez:

**Comando 1** - cria uma area separada dentro do Kubernetes para o ArgoCD:
```
kubectl create namespace argocd
```
Resultado esperado: `namespace/argocd created`

**Comando 2** - instala o ArgoCD:
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Vai aparecer uma lista longa de linhas com a palavra `created`. Isso e normal.

**Aguarde de 3 a 5 minutos** e depois execute esse comando para verificar:
```
kubectl get pods -n argocd
```

Fique repetindo esse comando ate que TODOS os itens da lista mostrem `Running` na coluna STATUS.
Enquanto estiver iniciando vai aparecer `Pending` ou `ContainerCreating`. So aguarde e repita.

Quando todos estiverem `Running`, continue para o proximo passo.

---

### PASSO 3 - Abrir a tela visual do ArgoCD no navegador

No PowerShell atual execute:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Vai aparecer: `Forwarding from 127.0.0.1:8080`

**IMPORTANTE: nao feche esse PowerShell. Deixe ele aberto.**

Agora abra um **segundo PowerShell** para os proximos comandos:
- Clique em Iniciar, pesquise PowerShell, abra uma nova janela

No Chrome ou Edge, acesse:
```
http://localhost:8080
```

Se aparecer aviso de seguranca no navegador:
- Clique em `Avancado`
- Clique em `Prosseguir para localhost`

A tela de login do ArgoCD vai aparecer.

- No campo **Username** digite: `admin`
- Para pegar a **senha**, execute no segundo PowerShell:

```
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}"
```

Vai aparecer uma sequencia de letras e numeros embaralhados. **Copie esse texto**.

Agora execute o comando abaixo trocando `COLE_AQUI` pelo texto que voce copiou:
```
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("COLE_AQUI"))
```

O resultado que aparecer e a senha. Copie e cole no campo **Password** no navegador e clique em **Sign In**.

---

### PASSO 4 - Conectar o ArgoCD a este repositorio

No segundo PowerShell, execute:
```
cd C:\fiap-tech-challenge-fase-3-gitops
```

```
kubectl apply -f argocd/application.yaml
```

Resultado esperado: `application.argoproj.io/fiap-app created`

Volte ao navegador no ArgoCD. Um cartao chamado **fiap-app** vai aparecer na tela.
Clique nele e voce vera todos os servicos (auth, flag, targeting, evaluation, analytics) exibidos graficamente.

---

### PASSO 5 - Confirmar os servicos rodando

No segundo PowerShell execute:
```
kubectl get pods -n fiap
```
```
kubectl get services -n fiap
```

---

### PASSO 6 - MOSTRAR O GITOPS FUNCIONANDO (ponto principal do video)

Agora voce vai simular uma atualizacao, como se o pipeline tivesse gerado uma nova versao da imagem.

**Abra o arquivo no Bloco de Notas:**
1. Abra o Explorador de Arquivos (a pasta amarela na barra de tarefas)
2. Navegue ate: `C:\fiap-tech-challenge-fase-3-gitops\gitops\auth`
3. Clique com o botao direito no arquivo `deployment.yaml`
4. Clique em `Abrir com` e escolha `Bloco de Notas`

Dentro do arquivo, procure a linha:
```
image: auth-service:v1.0.0
```

Mude para:
```
image: auth-service:v1.0.1
```

Salve com **Ctrl + S** e feche o Bloco de Notas.

Agora no segundo PowerShell execute um por vez:
```
cd C:\fiap-tech-challenge-fase-3-gitops
```
```
git add .
```
```
git commit -m "update auth-service para v1.0.1"
```
```
git push origin GR_20260524
```

Agora olhe o navegador com o ArgoCD aberto.
Em ate 3 minutos o status do cartao `fiap-app` vai mudar sozinho:

- **Synced** - estava tudo igual ao GitHub
- **OutOfSync** - ArgoCD detectou a mudanca no GitHub
- **Synced** - ArgoCD aplicou a mudanca no Kubernetes automaticamente, sem nenhum comando

**Isso e o GitOps:** voce so editou um arquivo e fez o push. O deploy aconteceu sozinho.

---

## FLUXO DO TRABALHO

```
Edson (Terraform)
Cria a infraestrutura na AWS: cluster EKS, repositorio ECR e banco RDS
        |
        v
Sandro (CI/CD)
Faz o build da imagem, envia para o ECR e atualiza o deployment.yaml neste repositorio
        |
        v
Renan (ArgoCD)
O ArgoCD detecta a mudanca neste repositorio e faz o deploy automatico no cluster
```
