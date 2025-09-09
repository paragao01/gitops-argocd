# Implantação de Ambiente de Desenvolvimento com GitOps usando Argo CD

## Visão Geral

Este documento descreve o processo padronizado para configurar um ambiente de desenvolvimento local em um cluster Kubernetes, utilizando uma abordagem GitOps auto-contida. Neste modelo, o próprio repositório contém tanto a configuração do Argo CD quanto os manifestos da aplicação a ser implantada.

O Argo CD é configurado para monitorar seu próprio repositório Git, aplicando os manifestos da aplicação `reviewvideo` que se encontram no diretório `k8s-webpage/`.

O fluxo de implantação é o seguinte:
1.  Criação de um cluster Kubernetes local com `k3d`, incluindo o mapeamento de porta necessário.
2.  Instalação do Argo CD no cluster.
3.  Configuração do Argo CD para se conectar a este repositório.
4.  Implantação da aplicação `reviewvideo` (uma aplicação .NET com um banco de dados PostgreSQL) gerenciada pelo Argo CD.

## 1. Pré-requisitos

Antes de iniciar, certifique-se de que as seguintes ferramentas estejam instaladas e configuradas:

*   **Docker:** Para execução dos contêineres do cluster.
    *   *Instalação:* [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)
*   **kubectl:** A ferramenta de linha de comando para interagir com o cluster Kubernetes.
    *   *Instalação:* [https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
*   **k3d:** Uma ferramenta leve para criar clusters k3s (distribuição Kubernetes) dentro do Docker.
    *   *Instalação:* [https://k3d.io/#installation](https://k3d.io/#installation)

## 2. Configuração do Cluster Kubernetes Local

Usaremos o `k3d` para criar o cluster. É crucial mapear a porta da aplicação `reviewvideo` do cluster para a sua máquina local durante a criação.

**Comando para Criação do Cluster:**

```bash
# O argumento "-p 30000:30000@loadbalancer" expõe o NodePort da nossa aplicação na sua máquina local.
k3d cluster create meucluster -p "30000:30000@loadbalancer"
```

## 3. Instalação e Configuração do Argo CD

O Argo CD será o núcleo da nossa estratégia GitOps.

### 3.1. Instalação do Argo CD

O arquivo `install.yaml` contém todos os manifestos necessários para a instalação.

Primeiro, crie o namespace onde o Argo CD será instalado:

```bash
kubectl create namespace argocd
```

Em seguida, aplique o manifesto de instalação:

```bash
kubectl apply -n argocd -f install.yaml
```

### 3.2. Acesso à Interface do Argo CD

Para acessar a interface de usuário do Argo CD, use `port-forward`. Execute em um terminal separado:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Acesse a UI em seu navegador: **https://localhost:8080**

### 3.3. Login no Argo CD

O nome de usuário padrão é `admin`. Para obter a senha inicial, execute:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 4. Implantação da Aplicação `reviewvideo`

Com o Argo CD em execução, vamos instruí-lo a monitorar este repositório e implantar a aplicação.

### 4.1. Atualize a URL do Repositório

O arquivo `app-argocd.yaml` define qual repositório o Argo CD deve observar.

**AÇÃO NECESSÁRIA:** Antes de continuar, abra o arquivo `app-argocd.yaml` e **altere o valor de `spec.source.repoURL`** para a URL do *seu* repositório Git (o repositório que você está usando para este projeto).

### 4.2. Aplique o Manifesto da Aplicação

Após atualizar a URL, aplique o manifesto:

```bash
kubectl apply -f app-argocd.yaml
```

O Argo CD irá agora sincronizar e implantar a aplicação `reviewvideo` e seu banco de dados PostgreSQL, conforme definido no diretório `k8s-webpage/`.

## 5. A Aplicação: `reviewvideo`

A aplicação implantada consiste em dois componentes principais:
*   **PostgreSQL:** Um banco de dados para a aplicação.
*   **ReviewVideo:** Uma aplicação .NET que se conecta ao banco de dados.

## 6. Verificação e Acesso

### 6.1. Verificação no Cluster

Você pode verificar o status da implantação via UI do Argo CD ou via linha de comando. Os pods devem estar no estado `Running` no namespace `argocd`.

```bash
kubectl get pods -n argocd
```

### 6.2. Acesso à Aplicação

Graças ao mapeamento de porta que fizemos na criação do cluster, a aplicação `reviewvideo` está diretamente acessível em seu navegador:

**URL:** [http://localhost:30000](http://localhost:30000)

## 7. Limpeza do Ambiente (Teardown)

Para remover completamente o ambiente de desenvolvimento, simplesmente delete o cluster `k3d`.

```bash
k3d cluster delete meucluster
```