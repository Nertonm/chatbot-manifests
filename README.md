# Manifestos Kubernetes para o Chatbot Abacate (GitOps)

Este repositório armazena os manifestos Kubernetes (YAML) para a implantação do projeto `chatbot-abacate`. Ele é projetado para ser usado com uma ferramenta de GitOps como o ArgoCD.

A atualização da imagem da aplicação é automatizada: a pipeline de CI/CD do [repositório da aplicação](https://github.com/Nertonm/chatbot-abacate) cria um Pull Request neste repositório sempre que uma nova versão é publicada.

## Estrutura dos Manifestos

O ambiente é composto por uma aplicação FastAPI e um banco de dados MySQL.

### 1. Banco de Dados (MySQL)

* `secret-mysql.yaml`: Cria um `Secret` contendo as credenciais do banco de dados (usuário, senha, root).
* `service-mysql.yaml`: Define um *Headless Service* (`clusterIP: None`), que é necessário para fornecer uma identidade de rede estável para o `StatefulSet`.
* `statefulset-mysql.yaml`: Implanta o MySQL usando um `StatefulSet`, garantindo que ele tenha armazenamento persistente (`PersistentVolumeClaim`) e um nome de host previsível. As variáveis de ambiente do container consomem os dados do `mysql-secret`.

### 2. Aplicação (Chatbot Abacate)

* `deployment-app.yaml`: Define o `Deployment` da aplicação.
    * Especifica a imagem a ser usada (ex: `docker.io/nerton/chatbot-abacate:2.0.5`). **Esta linha é atualizada automaticamente pelo CI/CD.**
    * Mapeia as variáveis de ambiente (como `DB_HOST`, `DB_USER`, `DB_PASSWORD`) para os valores definidos no `mysql-secret`.
    * Define `readinessProbe` e `livenessProbe` para monitorar a saúde da aplicação.
* `service-app.yaml`: Expõe o `Deployment` da aplicação internamente no cluster através de um serviço `ClusterIP` na porta 5000.

## Implantação com ArgoCD (GitOps)

Para implantar este projeto usando ArgoCD:

1.  **Crie a Aplicação no ArgoCD:** Na interface do ArgoCD, selecione "New App".

2.  **Configure as Informações Principais:**
    * **Application Name:** `chatbot-abacate`
    * **Project:** `default` (ou um projeto de sua escolha)
    * **Sync Policy:** `Automated`
        * Marque `Prune Resources` e `Self Heal` para uma sincronização completa.

3.  **Configure a Fonte (Source):**
    * **Repository URL:** (A URL deste repositório Git)
    * **Revision:** `HEAD` (ou `main`)
    * **Path:** `.` (ou o subdiretório onde os arquivos YAML estão)

4.  **Configure o Destino (Destination):**
    * **Cluster URL:** `https://kubernetes.default.svc` (para o cluster local)
    * **Namespace:** `chatbot` (ou um namespace de sua escolha. Crie-o se não existir: `kubectl create namespace chatbot`)

5.  **Sincronize:** Clique em "Create" e depois em "Sync". O ArgoCD irá aplicar os manifestos, e você verá a aplicação e o banco de dados sendo criados.

## Acesso e Teste

1.  **Verifique os Pods:** Após a sincronização do ArgoCD, verifique se os pods estão rodando:
    ```bash
    kubectl get pods -n chatbot
    
    # Saída esperada:
    # NAME                               READY   STATUS    RESTARTS   AGE
    # chatbot-abacate-7d5d...-abcde     1/1     Running   0          5m
    # chatbot-abacate-7d5d...-fghij     1/1     Running   0          5m
    # mysql-0                            1/1     Running   0          10m
    ```

2.  **Acesse a Aplicação (Port-Forward):** Para testar a aplicação localmente, use o `port-forward` do Kubernetes para direcionar o tráfego do serviço:
    ```bash
    kubectl port-forward svc/chatbot-abacate-svc 8080:5000 -n chatbot
    ```

3.  **Teste no Navegador:** Abra seu navegador e acesse [http://localhost:8080](http://localhost:8080). Você verá a interface do Chatbot Abacate.

## Fluxo de Atualização (CI/CD)

1.  Um desenvolvedor envia uma nova tag (ex: `v2.2.1`) para o repositório da aplicação.
2.  O GitHub Actions constrói e publica a imagem `nerton/chatbot-abacate:v2.2.1`.
3.  O Actions cria um PR *neste* repositório, alterando o `deployment-app.yaml` para usar a nova tag.
4.  Você revisa e **mescla** o Pull Request.
5.  O ArgoCD detecta que o estado no Git (`deployment-app.yaml` com a nova tag) está diferente do estado no cluster.
6.  O ArgoCD inicia um *rolling update* automático do `Deployment`, substituindo os pods antigos pelos novos com a imagem `v2.2.1`, tudo sem tempo de inatividade.