# Deploying DB for the K8s based application

Kubernetes shines when it comes to deloying, and scaling stateless workloads. The're fast to scale up and down.
Stateful workloads are the opposite. They require relatively careful orchestration, optimized distributed systems
implementation (Raft, Paxos, Leader Election, simple Primary/Secondary etc), and lots of experimentation for handling
scaling, backups, and restores.

Most databases like MySQL, and Postgres were designed during the time when the concept of cloud
computing either didn't exist or was in nascent phases. Compute, and storage was limited so it made sense to optimize
the utilization of single node system as much as possible. K8s which was inspired by Google internaly used orchestrator
Borg was came into existence 10 years ago, during the peak of cloud native era. Kubernetes solved

Yes it is true that some companies have able to achieve the goal running on Databases on K8s like Zalando with their
Zalando Postgres Operator. Yes there are vendors out there who provide commercial offerings of their

My approach would be simple: Keep compute and storage separate. Since Databases are require storage, I would like to see
it get deployed on separate, non K8s nodes. This strategy is actually very common when see it in the context of managed K8s
services from different cloud vendors like AWS, and Azure. For example, in AWS customers typically use EKS in conjunction with
with RDS/Aurora. In terms of the infrastructure, RDS nodes and K8s workers nodes are entirely different.

Finally for my application, I can think of couple of approaches. **I've listed them in least to most likeable in terms of scaling, and security IMO**:

1. Hard coded list of environment variables (`HOST`, `PORT`, `USERNAME`, `DB_NAME` and `PASSWORD` at minimum) in the chart. E.g
  
  ```yaml
    # values.yaml
    environment:
      HOST: "127.0.0.1"
      PORT: "5432"
      USERNAME: "postgres"
      PASSWORD: "postgres"
  ```

  ```yaml
    # templates/deployment.yaml
    .....
    spec:
      containers:
      - name: my-app-container
        image: my-app-image
        env:
        {{- range $key, $val := .Values.environment }}
        - name: {{ $key | upper }}
          value: {{ $val }}
        {{- end }}
  ```

2. Provide option for K8s `Secret` which contains relevant variables in chart, and mount it as a file. Application
  will then read from there.

  ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: db-creds
      namespace: default # change if needed
    type: Opaque
    data:
      HOST: MTI3LjAuMC4x
      PORT: NTQzMg==
      USERNAME: cG9zdGdyZXM=
      PASSWORD: cG9zdGdyZXM=
  ```

  ```yaml
    # values.yaml
    dbSecretName: tls-secret
  ```

  ```yaml
    # deployment.yaml
    .....
    spec:
      containers:
      - name: my-app-container
        image: my-app-image
        volumeMounts:
        - name: db-creds-volume
          mountPath: "/etc/db-creds" # some random path
          readOnly: true
      volumes:
      - name: db-creds-volume
        secret:
          secretName: db-creds
  ```

3. Use secret manager like KMS (AWS) or Hashicorp Vault to dynamicall inject the secrets via `Annotations`,
`ServiceAccounts`, and side car based agents. Secret managers can also help in handling lifecyles, and even
TLS certificates. Here's 1 such example from Hashicorp's (tutorial)[https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#inject-secrets-into-the-pod]:

  ```yaml
    # values.yaml
    vault:
      enable: true
      type: vault # can be either vault (Hashicorp), kms, akv or sm (Google Secret Manager)
      annotations:
        # annotations will vary based on the secret manager type
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'internal-app'
        vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'

    serviceAccount:
      enable: true
      name: vault-sa
  ```

  ```yaml
    # deployment.yaml
    .....
    spec:
      template:
        spec:
          {{- if .values.serviceAccount.enable }}
          serviceAccountName: {{ .values.serviceAccount.name }}
          {{- end}} 
          # pod level annotations are required, not deployment level for HCP Vault.
          {{- if eq .Values.vault.type "vault" }}
          annotations:
          {{- range $key, $val := .Values.vault.annotations }}
            {{ $key }}: {{ $val}}
          {{- end }}
  ```