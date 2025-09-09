# Implantação de Ambiente de Desenvolvimento com GitOps usando Argo CD

## Visão Geral

Este documento descreve o processo padronizado para configurar um ambiente de desenvolvimento local em um cluster Kubernetes. A abordagem utiliza a metodologia GitOps, com o Argo CD como ferramenta principal para garantir que o estado do cluster espelhe continuamente o estado definido em um repositório Git.

O fluxo de implantação é o seguinte:
1.  Criação de um cluster Kubernetes local.
2.  Instalação do Argo CD no cluster.
3.  Implantação de uma aplicação de exemplo (`webpage`) gerenciada pelo Argo CD, que por sua vez busca suas configurações de um repositório Git externo.

## 1. Pré-requisitos

Antes de iniciar, certifique-se de que as seguintes ferramentas estejam instaladas e configuradas em sua máquina local:

*   **Docker:** Para execução dos contêineres do cluster.
*   **kubectl:** A ferramenta de linha de comando para interagir com o cluster Kubernetes.
*   **k3d:** Uma ferramenta leve para criar clusters k3s (uma distribuição Kubernetes certificada) dentro do Docker. É ideal para desenvolvimento local.

## 2. Configuração do Cluster Kubernetes Local

Para um ambiente rápido e isolado, usaremos o `k3d` para criar o cluster.

**Comando para Criação do Cluster:**

Execute o seguinte comando para criar um novo cluster chamado `meucluster`:

```bash
k3d cluster create meucluster
```

Este comando provisionará um cluster Kubernetes simples, pronto para receber nossas aplicações.

## 3. Instalação e Configuração do Argo CD

O Argo CD será o núcleo de nossa estratégia GitOps.

### 3.1. Instalação do Argo CD

O arquivo `install.yaml` contém todos os manifestos necessários para a instalação do Argo CD.

Primeiro, crie o namespace onde o Argo CD será instalado:

```bash
kubectl create namespace argocd
```

Em seguida, aplique o manifesto de instalação:

```bash
kubectl apply -n argocd -f install.yaml
```

Este passo implantará todos os componentes do Argo CD, incluindo a UI, o controller e os CRDs (Custom Resource Definitions).

### 3.2. Acesso à Interface do Argo CD

Para acessar a interface de usuário do Argo CD, você pode expor o serviço `argocd-server` usando `port-forward`.

Execute o seguinte comando em um terminal separado:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Agora, acesse a UI em seu navegador: **https://localhost:8080**

### 3.3. Login no Argo CD

O nome de usuário padrão é `admin`. Para obter a senha inicial, execute o comando abaixo:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 4. Implantação da Aplicação (App of Apps)

Com o Argo CD em execução, o próximo passo é instruí-lo a monitorar nosso repositório de aplicação. O arquivo `app-argocd.yaml` define uma `Application` do Argo CD que aponta para um repositório Git externo.

**Aplique o Manifesto da Aplicação:**

```bash
kubectl apply -f app-argocd.yaml
```

Após aplicar este manifesto, o Argo CD irá:
1.  Ler a definição da aplicação `webpage`.
2.  Conectar-se ao repositório `https://github.com/paragao01/devops4devs.git`.
3.  Aplicar os manifestos localizados no diretório `k8s/` do repositório.
4.  Manter a aplicação sincronizada com o estado definido no Git.

## 5. Verificação

Você pode verificar o status da implantação através da UI do Argo CD ou via linha de comando.

**Verificar Pods:**

Liste todos os pods no namespace `argocd` para ver os componentes do Argo CD e da nova aplicação `webpage` (que pode ter sido implantada neste ou em outro namespace, dependendo da definição no repositório Git).

```bash
kubectl get pods -n argocd
```

Na UI do Argo CD, você verá um card para a aplicação `webpage` e poderá inspecionar seu status de sincronização, saúde e todos os recursos Kubernetes associados.

## 6. Limpeza do Ambiente (Teardown)

Para remover completamente o ambiente de desenvolvimento, siga os passos abaixo.

**1. Deletar o Cluster Kubernetes:**

O comando a seguir irá destruir o cluster `meucluster` e todos os recursos dentro dele.

```bash
k3d cluster delete meucluster
```

Este é o método mais limpo e rápido para resetar o ambiente.
