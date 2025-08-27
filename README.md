# netbackup-k8s-manual-deploy

Este guia fornece um passo a passo completo para a instalação e configuração **manual** do NetBackup Operator para Kubernetes. Todas as informações e comandos são baseados exclusivamente no documento oficial **NetBackup105\_AdminGuide\_Kubernetes.pdf**.

### **Fase 1: Preparação e Pré-requisitos do Ambiente**

Esta fase é dedicada exclusivamente à preparação do seu ambiente para garantir que a instalação ocorra sem problemas.

#### **1.1. Pré-requisitos Essenciais**

  * **Acesso Administrativo:** Você precisa de privilégios de administrador no cluster Kubernetes.
  * **Instalação do Helm:** O Helm v3 ou superior deve estar instalado em sua estação de trabalho.
  * **Obtenção dos Pacotes:** Baixe e extraia o pacote do **NetBackup Kubernetes Operator**.
  * **Preparo das Imagens de Contêiner:** Faça o upload das imagens do **NetBackup Operator** e do **Data Mover** para um repositório de contêineres que seja acessível pelo seu cluster Kubernetes.

#### **1.2. Configuração do Firewall**

Garanta que as seguintes regras de firewall estejam em vigor para permitir a comunicação entre os componentes:

| Origem | Destino | Porta | Protocolo | Finalidade |
| :--- | :--- | :--- | :--- | :--- |
| Servidor Primário | Cluster Kubernetes | 443 | TCP | Comunicações HTTPS |
| Servidores de Mídia | Cluster Kubernetes | 443 | TCP | Comunicações HTTPS |
| Cluster Kubernetes | Servidor Primário | 1556 | TCP (Saída) | Comunicação PBX e Certificados |
| Cluster Kubernetes | Servidores de Mídia | 1556 | TCP (Saída) | Certificados |
| Cluster Kubernetes | Servidor Primário e de Mídia | 13724 | TCP (Bidirecional) | VNETD para movimentação de dados |

### **Fase 2: Coleta de Dados para Configuração**

Nesta fase, vamos coletar e anotar todas as informações que serão usadas como variáveis nas fases de configuração e execução.

#### **2.1. Checklist de Coleta de Dados**

**a. Namespace da Instalação:**

Defina um nome padrão para sua organização.

**`<NAMESPACE_DA_INSTALACAO>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**b. Nome do Release:**

Defina um nome único para esta instalação com o Helm.

**`<NOME_DO_RELEASE>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**c. URLs das Imagens de Contêiner:**

URLs completas que você definiu ao fazer `docker push`.

**`<URL_DA_IMAGEM_DO_OPERADOR>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<URL_DA_IMAGEM_DO_DATA_MOVER>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**d. Nomes das Classes de Armazenamento e Snapshot:**

> ```bash
> kubectl get storageclasses
> kubectl get volumesnapshotclasses
> ```

**`<NOME_DA_STORAGECLASS_FILESYSTEM>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<NOME_DA_STORAGECLASS_BLOCK>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<NOME_DA_SNAPSHOTCLASS>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**e. Informações do Servidor Primário:**

**FQDN:** Nome de domínio completo do seu servidor.
**Impressão Digital:** Na UI Web do NetBackup, vá para **Segurança \> Certificados \> Autoridade Certificadora**.

**`<FQDN_DO_SERVIDOR_PRIMARIO>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<IMPRESSAO_DIGITAL_SHA256>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_


**f. Token de Autorização:**

Na UI Web do NetBackup, vá para **Segurança \> Tokens** e crie um novo token.

**`<TOKEN_DE_AUTORIZACAO>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**g. Informações do Cluster Kubernetes:**

> ```bash
> kubectl cluster-info
> ```

**`<FQDN_DO_CLUSTER_K8S>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<PORTA_DO_CLUSTER_K8S>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

### **Fase 3: Implantação e Configuração Manual**

Com os dados coletados, execute os passos de implantação e configuração.

#### **3.1. Implantação do Operador (Modo Manual)**

1.  **Edite o `values.yaml`:** Defina a imagem do operador e `replicas: 0`.
    ```yaml
    # Arquivo: netbackupkops-helm-chart/values.yaml
    netbackupkops:
      containers:
        manager:
          image: <URL_DA_IMAGEM_DO_OPERADOR>
    nbsetup:
      replicas: 0
    ```
2.  **Execute a Instalação com Helm:**
    ```bash
    helm install <NOME_DO_RELEASE> ./netbackupkops-helm-chart -n <NAMESPACE_DA_INSTALACAO>
    ```
3.  **Verifique o Pod:** Confirme que o pod `...-controller-manager-...` está rodando.
    ```bash
    kubectl get pods -n <NAMESPACE_DA_INSTALACAO>
    ```

#### **3.2. Configuração do Armazenamento**

Execute estes comandos usando as variáveis coletadas no passo **2.1.d**.

```bash
kubectl label storageclass <NOME_DA_STORAGECLASS_FILESYSTEM> netbackup.veritas.com/default-csi-filesystem-storage-class=true
kubectl label storageclass <NOME_DA_STORAGECLASS_BLOCK> netbackup.veritas.com/default-csi-storage-class=true
kubectl label volumesnapshotclass <NOME_DA_SNAPSHOTCLASS> netbackup.veritas.com/default-csi-volume-snapshot-class=true
```

#### **3.3. Configuração do Data Mover (`ConfigMap`)**

1.  **Crie o arquivo `datamover-configmap.yaml`:**
    ```yaml
    # Arquivo: datamover-configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: <FQDN_DO_SERVIDOR_PRIMARIO>
      namespace: <NAMESPACE_DA_INSTALACAO>
    data:
      datamover.properties: |
        image=<URL_DA_IMAGEM_DO_DATA_MOVER>
    ```
2.  **Aplique o `ConfigMap`:**
    ```bash
    kubectl apply -f datamover-configmap.yaml
    ```

#### **3.4. Implantação de Certificados**

1.  **Crie o `cert-secret.yaml`:**
    ```yaml
    # Arquivo: cert-secret.yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: nbca-secret-name
      namespace: <NAMESPACE_DA_INSTALACAO>
    type: Opaque
    stringData:
      token: <TOKEN_DE_AUTORIZACAO>
      fingerprint: <IMPRESSAO_DIGITAL_SHA256>
    ```
2.  **Crie o `backupservercert.yaml`:**
    ```yaml
    # Arquivo: backupservercert.yaml
    apiVersion: netbackup.veritas.com/v1
    kind: BackupServerCert
    metadata:
      name: <FQDN_DO_SERVIDOR_PRIMARIO>-cert-request
      namespace: <NAMESPACE_DA_INSTALACAO>
    spec:
      clusterName: <FQDN_DO_CLUSTER_K8S>:<PORTA_DO_CLUSTER_K8S>
      backupServer: <FQDN_DO_SERVIDOR_PRIMARIO>
      certificateOperation: Create
      certificateType: NBCA
      nbcaAttributes:
        nbcaCreateOptions:
          secretName: nbca-secret-name
    ```
3.  **Aplique os arquivos:**
    ```bash
    kubectl apply -f cert-secret.yaml
    kubectl apply -f backupservercert.yaml
    ```

#### 3.5. Adição do Cluster na Interface do NetBackup

1.  **Colete as Credenciais do Cluster:** Agora que o operador está configurado, colete o token final e o certificado.
    ```bash
    # Encontre o nome do secret
    SECRET_NAME=$(kubectl get secrets -n <NAMESPACE_DA_INSTALACAO> | grep backup-server-secret | awk '{print $1}')

    # Obtenha e anote o token
    kubectl get secret $SECRET_NAME -n <NAMESPACE_DA_INSTALACAO> -o jsonpath='{.data.token}' | base64 --decode

    # Obtenha e anote o certificado de CA
    kubectl get secret $SECRET_NAME -n <NAMESPACE_DA_INSTALACAO> -o jsonpath='{.data.ca\.crt}' | base64 --decode
    ```
2.  **Adicione o Cluster na UI:**
      * Vá para **Cargas de Trabalho \> Kubernetes \> Clusters Kubernetes** e clique em **Adicionar**.
      * Preencha as informações do cluster usando as variáveis coletadas.
      * Na tela de credenciais, selecione **Adicionar credencial** e cole o **Token** e o **Certificado de CA** que você acabou de obter.
      * Conclua o assistente.

### **Fase 4: Verificação Final**

  * Na UI do NetBackup, na tela de **Clusters Kubernetes**, o cluster recém-adicionado deve aparecer.
  * O status da descoberta deve mudar para **"Sucesso"** após alguns minutos, confirmando que a configuração manual foi bem-sucedida.
