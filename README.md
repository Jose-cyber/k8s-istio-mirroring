# Istio Traffic Mirroring

Exemplo de **espelhamento de tráfego HTTP com Istio** em um cluster Kubernetes local. O tráfego recebido pelo `httpbin` principal é respondido por ele e, em paralelo, replicado para uma segunda instância para testes e observabilidade.

> **Atenção:** o serviço de destino recebe uma cópia de cada requisição. Use dados de teste; o espelhamento não deve ser tratado como uma forma de processar tráfego de produção sem avaliar privacidade e efeitos colaterais.

## Visão geral

```text
Cliente
  │
  ▼
Gateway de entrada (Istio)
  ├── 100% do tráfego ──► httpbin1 (resposta ao cliente)
  └── cópia de 100% ────► httpbin2 (somente espelhamento)
```

O espelhamento é assíncrono do ponto de vista da resposta: o `httpbin2` recebe a cópia, mas sua resposta **não** é devolvida ao cliente. O projeto usa:

- **kind** para criar um cluster Kubernetes local com um nó de controle e dois workers;
- **Argo CD** e ApplicationSets para instalar e sincronizar os recursos;
- **Istio 1.30.7** para o gateway e o roteamento com espelhamento;
- duas instâncias do **httpbin**, nos namespaces `httpbin1` e `httpbin2`.

## Pré-requisitos

- [Docker](https://www.docker.com/) em execução;
- [kind](https://kind.sigs.k8s.io/);
- [kubectl](https://kubernetes.io/docs/tasks/tools/);
- acesso à internet para baixar imagens e charts.

Os comandos abaixo devem ser executados na raiz deste repositório. Eles usam sintaxe compatível com Bash; no Windows, podem ser executados pelo Git Bash ou WSL.

## Como executar

### 1. Criar o cluster

```bash
kind create cluster --config=k8s/cluster.yaml
```

O arquivo de configuração mapeia as portas HTTP e HTTPS do gateway para as portas `80` e `443` da máquina local. Confira se o cluster está pronto:

```bash
kubectl cluster-info --context kind-istio-mirroring
kubectl get nodes
```

### 2. Instalar o Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment --all \
  -n argocd --timeout=5m
```

### 3. Registrar as aplicações do projeto

```bash
kubectl apply -n argocd -f k8s/argocd/argocd-resources.yaml
```

Esse recurso inicia os ApplicationSets do repositório. O Argo CD instala o Istio, configura o gateway e sincroniza as aplicações `httpbin1` e `httpbin2`. A primeira sincronização pode levar alguns minutos.

Verifique o estado das aplicações e dos pods:

```bash
kubectl get applications -n argocd
kubectl get pods -A
```

### 4. Testar o espelhamento

Faça uma requisição ao gateway local:

```bash
curl http://localhost/get
```

A resposta deve vir do `httpbin1`. Para verificar a cópia recebida pela segunda instância, consulte os logs:

```bash
kubectl logs -n httpbin2 deployment/httpbin-v2
```

Cada chamada ao endpoint deve aparecer nos logs da instância espelhada. Também é possível acessar os endpoints do httpbin, por exemplo `http://localhost/headers` ou `http://localhost/status/200`.

## Acessar a interface do Argo CD (opcional)

Em um terminal, inicie o encaminhamento da porta:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Abra <https://localhost:8080> e entre com o usuário `admin`. Para obter a senha inicial no Bash:

```bash
initial_password=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)  && echo -e "\nUSER:admin\nPASS: $initial_password\n"
```



