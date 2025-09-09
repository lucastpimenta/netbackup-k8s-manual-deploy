# netbackup-k8s-manual-deploy

Este guia fornece um passo a passo completo para a instalação e configuração manual do NetBackup Operator para Kubernetes. Todas as informações e comandos são baseados exclusivamente no documento oficial `NetBackup™ for Kubernetes Administrator's Guide Release 10.5`.

## 📦 O que este Guia Faz

Este documento orienta você através do processo completo para implantar e preparar o ambiente NetBackup para Kubernetes, incluindo:

  * Instalação dos pré-requisitos, como o cliente Helm.
  * Publicação das imagens de contêiner do Operator e do Data Mover em um registro privado.
  * Implantação do NetBackup Kubernetes Operator no seu cluster usando Helm.
  * Configuração do Data Mover para operações de backup e restauração a partir de cópias.
  * Adição e validação do cluster Kubernetes na Web UI do NetBackup.
  * Configuração de certificados de segurança (NBCA) para comunicação segura entre os componentes.
  * Aplicação dos rótulos (labels) necessários em `StorageClasses` e `VolumeSnapshotClasses` para habilitar operações de snapshot.

## 🛠️ Tecnologias Envolvidas

  * **Veritas NetBackup 10.5**
  * **Kubernetes**
  * **Helm 3**
  * **Docker** (ou um runtime de contêiner compatível)
  * **YAML**

## 🚀 Como Usar

O processo é dividido em 7 fases principais, projetadas para serem seguidas em sequência. Cada fase prepara o ambiente para a próxima, garantindo uma configuração bem-sucedida.

* 1️⃣ **Fase 1:** Preparação do Ambiente e Pré-Requisitos
* 2️⃣ **Fase 2:** Preparação das Imagens de Contêiner
* 3️⃣ **Fase 3:** Instalação do NetBackup Kubernetes Operator via Helm
* 4️⃣ **Fase 4:** Configuração do Data Mover
* 5️⃣ **Fase 5:** Adição do Cluster Kubernetes na Web UI do NetBackup
* 6️⃣ **Fase 6:** Configuração de Certificados de Segurança (NBCA)
* 7️⃣ **Fase 7:** Configuração de StorageClasses e VolumeSnapshotClasses

-----

### 1️⃣ Fase 1: Preparação do Ambiente e Pré-Requisitos

Antes de iniciar a instalação, é crucial preparar o ambiente e garantir que todos os pré-requisitos sejam atendidos.

1.  **Configuração do Firewall**
   
    Garanta que as seguintes regras de firewall estejam em vigor para permitir a comunicação entre os componentes:

    | Origem | Destino | Porta | Protocolo |
    | :--- | :--- | :--- | :--- |
    | Servidor Primário | Cluster Kubernetes | 6443 | TCP |
    | Servidores de Mídia | Cluster Kubernetes | 6443 | TCP |
    | Cluster Kubernetes | Servidor Primário e de Mídia | 1556 | TCP |
    | Cluster Kubernetes | Servidor Primário e de Mídia | 13724 | TCP |
    |  Servidor Primário e de Mídia | Cluster Kubernetes | 13724 | TCP |

2.  **Download e Extração do Pacote de Instalação**

    Faça o download do pacote do NetBackup Kubernetes Operator a partir do site de Suporte da Veritas. Em seguida, extraia o pacote para o seu diretório `home`. Após a extração, você terá a estrutura de diretórios do Helm chart disponível.

4.  **Instalação do Cliente Helm**

    O Helm é necessário para gerenciar a implantação do operador. Se ainda não o tiver instalado, execute os seguintes comandos:

    ```bash
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```

6.  **Verificação e Limpeza de Instalações Anteriores (Opcional)**

    Antes de instalar uma nova versão, desinstale qualquer plug-in mais antigo para evitar conflitos.

      * Liste os charts Helm existentes no namespace `netbackup`:
        ```bash
        helm list -n netbackup
        ```
      * Se um plug-in antigo for encontrado, desinstale-o:
        ```bash
        helm uninstall <nome_do_plugin_antigo> -n netbackup
        ```
        >**Nota:** Substitua `<nome_do_plugin_antigo>` pelo nome real do release listado.

### 2️⃣ Fase 2: Preparação das Imagens de Contêiner

As imagens do NetBackup Kubernetes Operator e do Data Mover devem ser enviadas para um registro de contêiner privado acessível pelo seu cluster Kubernetes.

1.  **Login no Registro de Contêiner Privado**

    Execute o login no seu registro de contêiner. Este passo cria ou atualiza o arquivo `config.json` com o token de autorização.

    ```bash
    docker login -u <seu_usuario> <seu_repositorio>
    ```

    >**Nota:** Substitua `<seu_usuario>` e `<seu_repositorio>` (ex: `registry.empresa.com`) pelos seus dados.

3.  **Criação do Segredo de Acesso ao Registro (Image Pull Secret)**

    Crie um segredo no namespace `netbackup` para que o Kubernetes possa autenticar e puxar as imagens do seu registro privado.

    ```bash
    kubectl create secret generic netbackupkops-docker-cred \
    --from-file=.dockerconfigjson=${HOME}/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson -n netbackup
    ```

      * Verifique se o segredo foi criado:
        ```bash
        kubectl get secrets -n netbackup
        ```

5.  **Carregamento e Envio da Imagem do Operator**

      * Carregue a imagem do Operator (contida no pacote baixado) para o cache local do Docker:
        ```bash
        docker load -i <caminho_para_o_arquivo>/netbackupkops.tar
        ```
      * Identifique o nome e a tag da imagem carregada com `docker images`.
      * Marque (tag) a imagem para o seu repositório privado:
        ```bash
        docker tag <nome_imagem_carregada>:<tag> <seu_repositorio>/<imagem_operador>:<tag>
        ```
      * Envie a imagem para o repositório:
        ```bash
        docker push <seu_repositorio>/<imagem_operador>:<tag>
        ```

6.  **Carregamento e Envio da Imagem do Data Mover**

    Repita o processo para a imagem do Data Mover, que é fornecida em um arquivo `tar` separado.

      * Carregue a imagem do Data Mover:
        ```bash
        docker load -i <caminho_para_o_arquivo>/veritas-netbackup-datamover-<versao>.tar
        ```
      * Marque (tag) a imagem para o seu repositório privado:
        ```bash
        docker tag <nome_imagem_datamover>:<tag> <seu_repositorio>/<imagem_datamover>:<tag>
        ```
      * Envie a imagem para o repositório:
        ```bash
        docker push <seu_repositorio>/<imagem_datamover>:<tag>
        ```

### 3️⃣ Fase 3: Instalação do NetBackup Kubernetes Operator via Helm

Com as imagens no registro, você pode prosseguir com a instalação do operador.

1.  **Modificação do Arquivo `values.yaml`**

    Edite o arquivo `netbackupkops-helm-chart/values.yaml` e atualize os seguintes parâmetros:

      * **Imagem do Operator:** Substitua o valor de `image` na seção `manager` pelo caminho completo da imagem que você enviou ao seu repositório.
        ```yaml
        # Exemplo de modificação em values.yaml
        manager:
          image: <seu_repositorio>/<imagem_operador>:<tag>
        ```
      * **Réplicas:** Altere o valor de `replicas` para `0`.
      * **Image Pull Secret:** Se você criou um segredo no passo anterior, adicione-o.

    >**Nota sobre o Volume Persistente:** O tamanho padrão do volume persistente de metadados para o operador é de 10Gi. Se necessário, você pode ajustar este valor no arquivo `deployment.yaml` antes da instalação.

3.  **Instalação com o Helm**

    Execute o comando a seguir para instalar o NetBackup Kubernetes Operator no namespace `netbackup`.

    ```bash
    helm install veritas-netbackupkops ./netbackupkops-helm-chart -n netbackup
    ```

5.  **Verificação da Instalação**

      * Verifique o status da implantação:
        ```bash
        helm list -n netbackup
        ```
      * Verifique o histórico do release:
        ```bash
        helm history veritas-netbackupkops -n netbackup
        ```

### 4️⃣ Fase 4: Configuração do Data Mover

O Data Mover é configurado através de um ConfigMap para cada servidor primário que realizará operações de "Backup from Snapshot" e "Restore from Backup".

1.  **Criação do Arquivo `configmap-datamover.yaml`**

    Crie um arquivo YAML com o conteúdo abaixo, substituindo os placeholders pelos valores corretos para o seu ambiente.

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: <nome_do_servidor_primario>
      namespace: netbackup
    data:
      datamover.properties: |
        image=<seu_repositorio>/<imagem_datamover>:<tag>
      datamover.hostaliases: |
        <ip_servidor_primario> <nome_fqdn_servidor_primario> <nome_curto_servidor_primario>
        <ip_servidor_de_midia> <nome_fqdn_servidor_de_midia> <nome_curto_servidor_de_midia>
      version: "1"
    ```

    **Explicação:**

      * `metadata.name`: Deve ser o nome exato do seu servidor primário NetBackup.
      * `datamover.properties.image`: O caminho completo da imagem do Data Mover no seu repositório.
      * `datamover.hostaliases`: Mapeia os IPs para os nomes de host FQDN e curtos dos seus servidores primário e de mídia para garantir a resolução de nomes correta a partir do pod do Data Mover.

3.  **Aplicação do ConfigMap**

    Aplique o arquivo de configuração no cluster Kubernetes.

    ```bash
    kubectl create -f configmap.yaml
    ```

### 5️⃣ Fase 5: Adição do Cluster Kubernetes na Web UI do NetBackup

Após a instalação do operador, adicione o cluster à interface web do NetBackup.

1.  **Navegue até a Seção Kubernetes**

    Na Web UI do NetBackup, vá para `Workloads` \> `Kubernetes`.

3.  **Adicione um Novo Cluster**

    Clique na aba `Kubernetes clusters` e depois em `Add`.

5.  **Preencha os Detalhes do Cluster**

      * **Cluster name:** O nome DNS ou IP do servidor da API do Kubernetes.
      * **Port:** A porta do servidor da API do Kubernetes.
      * **Controller namespace:** O namespace onde o operador foi instalado, que neste guia é `netbackup`.

6.  **Forneça as Credenciais**

    Na página `Manage credentials`, selecione `Add credential`. Você precisará do **Token** e do **CA certificate** da conta de serviço de backup criada pelo operador.

      * Para obter as credenciais, execute o seguinte comando no seu cluster Kubernetes:
        ```bash
        kubectl get secret netbackup-backup-server-secret -n netbackup -o yaml
        ```
      * Copie os valores de `token` e `ca.crt`. **Importante:** O token está codificado em Base64 e precisa ser decodificado antes de ser inserido na Web UI.
      * Insira o token decodificado e o conteúdo do certificado CA nos campos correspondentes.

8.  **Validação e Conclusão**

    Clique em `Next`. Após a validação bem-sucedida, o cluster será adicionado e a descoberta automática de ativos será iniciada.

### 6️⃣ Fase 6: Configuração de Certificados de Segurança (NBCA)

Esta fase é mandatória para habilitar a comunicação segura para operações de "Backup from Snapshot" e "Restore from Backup".

1.  **Obtenha o Fingerprint da Autoridade Certificadora (CA) do NetBackup**

    No servidor primário do NetBackup, execute o comando:

    ```bash
    /usr/openv/netbackup/bin/nbcertcmd -listCACertDetails
    ```

    Copie o valor do `SHA-256 Fingerprint`.

3.  **Crie um Token de Autorização**

    Na Web UI do NetBackup, crie um token de autorização.

5.  **Crie um Segredo com o Token e o Fingerprint**

    Crie um arquivo chamado `secret-nbca.yaml` com o seguinte conteúdo, substituindo os placeholders:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: nbca-token-fingerprint-secret
      namespace: netbackup
    type: Opaque
    stringData:
      token: "<seu_token_de_autorizacao>"
      fingerprint: "<seu_fingerprint_sha256>"
    ```

    Aplique o segredo no cluster:

    ```bash
    kubectl create -f secret-nbca.yaml
    ```

7.  **Crie a Solicitação de Certificado (Custom Resource)**

    Crie um arquivo chamado `backupservercert-request.yaml` com o seguinte conteúdo, substituindo os placeholders:

    ```yaml
    apiVersion: netbackup.veritas.com/v1
    kind: BackupServerCert
    metadata:
      name: <nome_do_servidor_primario>-nbca-create
      namespace: netbackup
    spec:
      clusterName: <nome_do_cluster_k8s>:<porta>
      backupServer: <nome_do_servidor_primario>
      certificateOperation: Create
      certificateType: NBCA
      nbcaAttributes:
        nbcaCreateOptions:
          secretName: nbca-token-fingerprint-secret
    ```

    Aplique o recurso personalizado para iniciar a solicitação do certificado:

    ```bash
    kubectl create -f nbca-create-backupservercert.yaml
    ```

9.  **Verifique o Status da Solicitação**

    A solicitação de certificado é um processo assíncrono. Verifique seu status. A operação só é concluída quando o status for `Success`.

    ```bash
    kubectl get backupservercert <nome_do_servidor_primario>-nbca-create -n netbackup -o yaml
    ```

### 7️⃣ Fase 7: Configuração de StorageClasses e VolumeSnapshotClasses

Para que as operações de snapshot funcionem corretamente, as `StorageClasses` e `VolumeSnapshotClasses` do seu provedor CSI devem ser rotuladas (labeled) adequadamente.

1.  **Rotule (Label) as StorageClasses**

    Identifique as StorageClasses que serão usadas para provisionar volumes e aplique os rótulos correspondentes.

      * Para `StorageClass` que provisiona volumes em modo `Block`:
        ```bash
        kubectl label storageclass <nome_da_storageclass_block> netbackup.veritas.com/default-csi-storage-class=true
        ```
      * Para `StorageClass` que provisiona volumes em modo `Filesystem`:
        ```bash
        kubectl label storageclass <nome_da_storageclass_filesystem> netbackup.veritas.com/default-csi-filesystem-storage-class=true
        ```

    >**Nota:** Se uma StorageClass suportar ambos os modos, você pode aplicar ambos os rótulos.

3.  **Rotule (Label) as VolumeSnapshotClasses**

    Aplique o seguinte rótulo a todas as `VolumeSnapshotClasses` que serão usadas para as operações de snapshot:

    ```bash
    kubectl label volumesnapshotclass <nome_da_volumesnapshotclass> netbackup.veritas.com/default-csi-volume-snapshot-class=true
    ```

>**Aviso:** A falha em rotular corretamente esses recursos resultará em falhas nas operações de snapshot de namespaces que contêm volumes persistentes.

-----

## 🆘 Suporte

Para suporte, comece verificando se você seguiu todas as instruções corretamente. Se o problema persistir, consulte o capítulo "Troubleshooting Kubernetes issues" no `NetBackup™ for Kubernetes Administrator's Guide` e a documentação oficial da Veritas.

## 🌟 Contribuições

Contribuições são sempre bem-vindas\! Se você tem uma sugestão para melhorar este script, sinta-se à vontade para criar um pull request.

## ✒️ Autor

[Lucas Pimenta](https://github.com/lucastpimenta) - Trabalho Inicial
