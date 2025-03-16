---
title: "kubernetesにtlsを有効にしたHashiCorp Vaultを構築してみる"
date: "2025-03-16T10:19:33+09:00"
draft: false
tags:
  - kubernetes
  - HashiCorp Vault
---

こちらのチュートリアルの内容をやってみる

https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-minikube-tls

## 環境

チュートリアルでは minikube を使っていますが今回は docker desktop の k8s を利用します。

```
$ kubectl get node
NAME             STATUS   ROLES           AGE    VERSION
docker-desktop   Ready    control-plane   266d   v1.29.1
```

## HashiCorp の Helm レポジトリを追加する

```
$ helm repo add hashicorp https://helm.releases.hashicorp.com
"hashicorp" has been added to your repositories
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "hashicorp" chart repository
Update Complete. ⎈Happy Helming!⎈
$ helm search repo hashicorp/vault
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
hashicorp/vault                         0.29.1          1.18.1          Official HashiCorp Vault Chart
hashicorp/vault-secrets-gateway         0.0.2           0.1.0           A Helm chart for Kubernetes
hashicorp/vault-secrets-operator        0.10.0          0.10.0          Official Vault Secrets Operator Chart
```

## TLS で使用する証明書を作成する

### 準備

```
$ mkdir /tmp/vault
$ export VAULT_K8S_NAMESPACE="vault"
$ export VAULT_HELM_RELEASE_NAME="vault"
$ export VAULT_SERVICE_NAME="vault-internal"
$ export K8S_CLUSTER_NAME="cluster.local"
$ export WORKDIR=/tmp/vault
```

## 秘密鍵を生成

```
$ openssl genrsa -out ${WORKDIR}/vault.key 2048
```

## CSR を作成

```
$ cat > ${WORKDIR}/vault-csr.conf <<EOF
[req]
default_bits = 2048
prompt = no
encrypt_key = yes
default_md = sha256
distinguished_name = kubelet_serving
req_extensions = v3_req
[ kubelet_serving ]
O = system:nodes
CN = system:node:*.${VAULT_K8S_NAMESPACE}.svc.${K8S_CLUSTER_NAME}
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.${VAULT_SERVICE_NAME}
DNS.2 = *.${VAULT_SERVICE_NAME}.${VAULT_K8S_NAMESPACE}.svc.${K8S_CLUSTER_NAME}
DNS.3 = *.${VAULT_K8S_NAMESPACE}
IP.1 = 127.0.0.1
EOF
```

```
$ openssl req -new -key ${WORKDIR}/vault.key -out ${WORKDIR}/vault.csr -config ${WORKDIR}/vault-csr.conf
```

## Kubernetes で証明書を発行する

先ほど生成した CSR を yaml にする

```
$ cat > ${WORKDIR}/csr.yaml <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
   name: vault.svc
spec:
   signerName: kubernetes.io/kubelet-serving
   expirationSeconds: 8640000
   request: $(cat ${WORKDIR}/vault.csr|base64|tr -d '\n')
   usages:
   - digital signature
   - key encipherment
   - server auth
EOF
```

CSR のリソースを Kubernetes に作成

```
$ kubectl create -f ${WORKDIR}/csr.yaml
certificatesigningrequest.certificates.k8s.io/vault.svc created
```

Approve する

```
$ kubectl certificate approve vault.svc
certificatesigningrequest.certificates.k8s.io/vault.svc approved
```

Approve されたことが確認できる

```
$ kubectl get csr vault.svc
NAME        AGE   SIGNERNAME                      REQUESTOR            REQUESTEDDURATION   CONDITION
vault.svc   38s   kubernetes.io/kubelet-serving   docker-for-desktop   100d                Approved,Issued
```

### 発行された証明書の中身を確認してみる

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            4e:99:79:ee:40:56:99:2e:90:85:a7:d2:14:58:f6:d0
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: Mar 16 01:25:02 2025 GMT
            Not After : Jun 24 01:25:02 2025 GMT
        Subject: O = system:nodes, CN = system:node:*.vault.svc.cluster.local
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:93:9d:76:d1:4e:80:05:29:ef:d0:bc:6b:47:75:
                    e1:46:c5:8d:51:ef:6d:93:e6:a2:56:53:cf:99:fa:
                    2c:11:0e:d0:09:e7:86:1b:73:df:1c:d8:22:8f:57:
                    ed:f7:fd:93:5e:e6:56:6d:44:ca:3f:69:96:4c:51:
                    11:8b:b8:e9:6a:f2:53:fa:04:3f:be:ea:f8:91:de:
                    6c:b0:ec:06:f4:84:d8:dd:c6:b6:34:fa:ab:c0:97:
                    1a:a4:f7:78:73:c5:47:39:b3:19:ae:d4:cf:a7:85:
                    f2:be:66:fc:93:1d:c3:c6:d8:da:6a:af:84:05:b7:
                    9a:dd:c4:06:31:f4:88:fc:f8:8f:a6:3d:d8:cf:52:
                    4d:db:de:7d:17:2d:91:06:5f:4d:bc:3b:44:62:7b:
                    1b:0e:a1:7d:7d:43:26:58:0d:88:bf:32:21:1c:d4:
                    4a:42:be:6e:32:16:88:ce:f8:6a:a1:11:64:8e:10:
                    19:58:d7:3b:47:40:e5:e8:28:d0:af:17:88:57:02:
                    e3:b7:78:a8:3b:ba:aa:41:9c:ca:34:0f:ec:95:23:
                    a1:d3:44:c3:c5:e7:d1:bb:99:36:0d:45:9b:93:ce:
                    37:19:fe:17:ef:ee:1d:e7:b3:22:3e:cb:40:d2:c1:
                    eb:70:b9:89:ee:7a:66:86:80:cf:95:78:a7:3d:b2:
                    d6:61
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier:
                AF:29:F2:53:06:57:E8:50:29:1F:CA:74:A6:8F:CA:3D:C5:37:64:AF
            X509v3 Subject Alternative Name:
                DNS:*.vault-internal, DNS:*.vault-internal.vault.svc.cluster.local, DNS:*.vault, IP Address:127.0.0.1
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        b3:d0:93:d9:0e:aa:9d:a7:2d:c6:3b:83:fa:0f:58:a0:97:5d:
        81:c7:72:73:dd:12:60:2b:c9:07:4d:57:4e:48:89:a9:58:00:
        ce:23:aa:cc:b5:ab:b2:53:54:c9:c2:75:fa:d0:38:8f:af:73:
        bb:fe:87:73:29:02:65:46:c2:a7:2e:4e:74:22:ab:ae:54:10:
        21:6d:1e:57:66:e5:df:3a:3c:05:c2:5d:ee:62:4f:22:57:4e:
        7c:35:12:ce:c5:6e:c9:34:e3:a5:1f:ee:fc:8f:b5:ec:d7:58:
        ea:48:43:c8:6a:f4:f3:d7:2f:ef:98:d1:66:43:8b:ae:db:4a:
        c2:2e:23:01:30:9c:97:40:f3:14:23:0a:db:1b:e2:a2:27:18:
        5a:ee:7e:f4:8e:24:c3:90:11:7a:2a:2a:f9:76:44:80:4e:95:
        7a:13:45:d1:79:27:42:65:e9:0a:e6:25:d2:27:14:ee:54:63:
        90:d4:33:51:f3:65:21:f3:2a:75:d1:29:8e:63:80:b7:20:46:
        a1:f1:28:60:ee:04:f8:7c:19:a6:89:28:2a:0d:a6:26:58:a8:
        f8:65:d8:26:5a:b1:14:a2:a6:f6:c7:ca:68:b0:a4:6b:30:7a:
        eb:8b:9d:6f:a8:79:e1:22:b9:31:d2:41:d1:9c:82:99:ef:2d:
        b0:15:f4:0c
```

CSR の通り発行されていることが確認できる

- Issuer CN が kubernetes となっている
- 期限が 100 日

```
$ echo $((($(date --date "Jun 24 01:25:02 2025 GMT" +%s)-$(date --date "Mar 16 01:25:02 2025 GMT" +%s)) / 60 / 60 / 24 ))
100
```

- CN は`system:node:*.vault.svc.cluster.local`
- SAN は`DNS:*.vault-internal, DNS:*.vault-internal.vault.svc.cluster.local, DNS:*.vault, IP Address:127.0.0.1`

## 証明書と秘密鍵を Secret リソースにする

先ほど発行した証明書を取得

```
$ kubectl get csr vault.svc -o jsonpath='{.status.certificate}' | openssl base64 -d -A -out ${WORKDIR}/vault.crt
```

Kubernetes の CA を取得

```
kubectl config view \
--raw \
--minify \
--flatten \
-o jsonpath='{.clusters[].cluster.certificate-authority-data}' \
| base64 -d > ${WORKDIR}/vault.ca
```

Vault のネームスペースを作成

```
$ kubectl create namespace $VAULT_K8S_NAMESPACE
namespace/vault created
```

TLS の Secret リソースを作成

```
$ kubectl create secret generic vault-ha-tls \
   -n $VAULT_K8S_NAMESPACE \
   --from-file=vault.key=${WORKDIR}/vault.key \
   --from-file=vault.crt=${WORKDIR}/vault.crt \
   --from-file=vault.ca=${WORKDIR}/vault.ca
secret/vault-ha-tls created
```

## Helm を使って vault cluster をデプロイする

### overrides.yaml を作成する

values.yaml を上書きするための overrides.yaml を作成します。

```
$ cat > ${WORKDIR}/overrides.yaml <<EOF
global:
   enabled: true
   tlsDisable: false
injector:
   enabled: true
server:
   extraEnvironmentVars:
      VAULT_CACERT: /vault/userconfig/vault-ha-tls/vault.ca
      VAULT_TLSCERT: /vault/userconfig/vault-ha-tls/vault.crt
      VAULT_TLSKEY: /vault/userconfig/vault-ha-tls/vault.key
   volumes:
      - name: userconfig-vault-ha-tls
        secret:
         defaultMode: 420
         secretName: vault-ha-tls
   volumeMounts:
      - mountPath: /vault/userconfig/vault-ha-tls
        name: userconfig-vault-ha-tls
        readOnly: true
   standalone:
      enabled: false
   affinity: ""
   ha:
      enabled: true
      replicas: 3
      raft:
         enabled: true
         setNodeId: true
         config: |
            cluster_name = "vault-integrated-storage"
            ui = true
            listener "tcp" {
               tls_disable = 0
               address = "[::]:8200"
               cluster_address = "[::]:8201"
               tls_cert_file = "/vault/userconfig/vault-ha-tls/vault.crt"
               tls_key_file  = "/vault/userconfig/vault-ha-tls/vault.key"
               tls_client_ca_file = "/vault/userconfig/vault-ha-tls/vault.ca"
            }
            storage "raft" {
               path = "/vault/data"
            }
            disable_mlock = true
            service_registration "kubernetes" {}
EOF
```

volumes で先ほど作成した TLS の Secret リソースを指定し、volumeMountes でマウントしていることがわかります。

また `server.ha.config`の `listener "tcp"`の `tls_{cert,key,client_ca}_file`にマウントした Secret のファイルのパスを指定しています。

### helm install

この overrides.yaml を指定して`helm install`を実行します。

```
$ helm install -n $VAULT_K8S_NAMESPACE $VAULT_HELM_RELEASE_NAME hashicorp/vault -f ${WORKDIR}/overrides.yaml
NAME: vault
LAST DEPLOYED: Sun Mar 16 10:50:20 2025
NAMESPACE: vault
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://developer.hashicorp.com/vault/docs


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault
```

## vault-0 をセットアップする

### 初期化する

しばらくすると Pod が Running になります

```
$ kubectl -n $VAULT_K8S_NAMESPACE get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 0/1     Running   0          32s
vault-1                                 0/1     Running   0          32s
vault-2                                 0/1     Running   0          32s
vault-agent-injector-8666bf4bb8-w9l82   1/1     Running   0          42s
```

初期化を行っていないので READY が `0/1`になっています。Pod`vault-0` で `vault operator init`を実行します。

```
$ kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > ${WORKDIR}/cluster-keys.json
```

### Unseal する

初期化しただけだと Seal 状態なのでまだ READY が `0/1`です。

```
$ kubectl -n $VAULT_K8S_NAMESPACE get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 0/1     Running   0          2m26s
vault-1                                 0/1     Running   0          2m26s
vault-2                                 0/1     Running   0          2m26s
vault-agent-injector-8666bf4bb8-w9l82   1/1     Running   0          2m36s
$ kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       1
Threshold          1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.18.1
Build Date         2024-10-29T14:21:31Z
Storage Type       raft
HA Enabled         true
command terminated with exit code 2
```

保存した unseal key をつかって Unseal します。

```
$ VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" ${WORKDIR}/cluster-keys.json)
```

```
$ kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.18.1
Build Date              2024-10-29T14:21:31Z
Storage Type            raft
Cluster Name            vault-integrated-storage
Cluster ID              cd380c88-b8e1-c44a-7add-709b36ed00bf
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 active
Active Since            2025-03-16T01:54:35.333945382Z
Raft Committed Index    57
Raft Applied Index      57
```

vault-0 がセットアップできたので READY が `1/1`になりました。

```
$ kubectl -n $VAULT_K8S_NAMESPACE get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          5m11s
vault-1                                 0/1     Running   0          5m11s
vault-2                                 0/1     Running   0          5m11s
vault-agent-injector-8666bf4bb8-w9l82   1/1     Running   0          5m21s
```

## vault-1 をセットアップする

### Raft cluster に Join させる

vault-1 を Raft cluster に join させます。

```
$ kubectl exec -n $VAULT_K8S_NAMESPACE -it vault-1 -- /bin/sh
/ $ vault operator raft join -address=https://vault-1.vault-internal:8200 -leader-ca-cert="$(cat /vault/userconfig/vault-ha-tls/vault.ca)" -leader-client-cert="$(cat /vault/userconfig/vault-ha-tls/vault.crt)" -leader-client-ke
y="$(cat /vault/userconfig/vault-ha-tls/vault.key)" https://vault-0.vault-internal:8200
Key       Value
---       -----
Joined    true

/ $ exit
```

### Unseal する

```
$ kubectl exec -n $VAULT_K8S_NAMESPACE -ti vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       1
Threshold          1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.18.1
Build Date         2024-10-29T14:21:31Z
Storage Type       raft
HA Enabled         true
```

vault-1 も READY が `1/1`になりました

```
hum@ryzen5pc:~/github.com/ophum/super-g.eek.jp$ kubectl -n $VAULT_K8S_NAMESPACE get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          7m49s
vault-1                                 1/1     Running   0          7m49s
vault-2                                 0/1     Running   0          7m49s
vault-agent-injector-8666bf4bb8-w9l82   1/1     Running   0          7m59s
```

## vault-2 をセットアップする

### Raft cluster に Join させる

```
$ kubectl exec -n $VAULT_K8S_NAMESPACE -it vault-2 -- /bin/sh
/ $ vault operator raft join -address=https://vault-2.vault-internal:8200 -leader-ca-cert="$(cat /vault/userconfig/vault-ha-tls/vault.ca)" -leader-client-cert="$(cat /vault/userconfig/vault-ha-tls/vault.crt)" -leader-client-ke
y="$(cat /vault/userconfig/vault-ha-tls/vault.key)" https://vault-0.vault-internal:8200
Key       Value
---       -----
Joined    true
/ $ exit
```

### Unseal する

```
$ kubectl exec -n $VAULT_K8S_NAMESPACE -ti vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       1
Threshold          1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.18.1
Build Date         2024-10-29T14:21:31Z
Storage Type       raft
HA Enabled         true
```

## 動作確認

### Vault cluster の状態を確認する

Root Token でログインします。

(※出力の token が github に push するときに怒られるので伏字にしてますが、本来は平文で出力されます。本番で作業ログで出力を保存する際は気を付けたほうがいいかもしれない。)

```
$ export CLUSTER_ROOT_TOKEN=$(cat ${WORKDIR}/cluster-keys.json | jq -r ".root_token")
$ kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault login $CLUSTER_ROOT_TOKEN
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.************************
token_accessor       zMY8fpB4ZXMdCgr4JMa918Ei
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

raft の peer に vault-{0,1,2}が存在しており、vault-0 が leader になっていることがわかります。

```
$ kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator raft list-peers
Node       Address                        State       Voter
----       -------                        -----       -----
vault-0    vault-0.vault-internal:8201    leader      true
vault-1    vault-1.vault-internal:8201    follower    true
vault-2    vault-2.vault-internal:8201    follower    true
```

HA Cluster の Scheme が https になっていることがわかります。

```
$ kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault status
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.18.1
Build Date              2024-10-29T14:21:31Z
Storage Type            raft
Cluster Name            vault-integrated-storage
Cluster ID              cd380c88-b8e1-c44a-7add-709b36ed00bf
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 active
Active Since            2025-03-16T01:54:35.333945382Z
Raft Committed Index    76
Raft Applied Index      76
```

### KV SecretEngine を作ってみる

KV SecretEngine を作成し、https で操作できるか試してみます。

vault-0 のシェルに接続

```
$ kubectl exec -n $VAULT_K8S_NAMESPACE -it vault-0 -- /bin/sh
```

KV SecretEngine を作成

```
/ $ vault secrets enable -path=secret kv-v2
Success! Enabled the kv-v2 secrets engine at: secret/
```

シークレット情報を保存

```
/ $ vault kv put secret/tls/apitest username="apiuser" password="supersecret"
===== Secret Path =====
secret/data/tls/apitest

======= Metadata =======
Key                Value
---                -----
created_time       2025-03-16T02:05:37.208477055Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

シェルを抜ける

```
/ $ exit
```

### API で KV SecretEngine からシークレット情報を取得する

vault の Service

```
$ kubectl -n $VAULT_K8S_NAMESPACE get service vault
NAME    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
vault   ClusterIP   10.109.238.161   <none>        8200/TCP,8201/TCP   17m
```

Type が ClusterIP なので kubectl の port-forward を使って手元からアクセスできるようにします。

別のターミナルで実行します。

```
$ kubectl -n vault port-forward service/vault 8200:8200
Forwarding from 127.0.0.1:8200 -> 8200
Forwarding from [::1]:8200 -> 8200

```

curl で API を実行します。
先ほど作成したシークレット情報を取得できました。

```
$ curl --cacert $WORKDIR/vault.ca \
    --header "X-Vault-Token: $CLUSTER_ROOT_TOKEN" \
    https://127.0.0.1:8200/v1/secret/data/tls/apitest | jq .data.data
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   365  100   365    0     0  35592      0 --:--:-- --:--:-- --:--:-- 36500
{
  "password": "supersecret",
  "username": "apiuser"
}
```

http でアクセスしてみるとエラーになり HTTPS サーバーとして動作していることがわかります。

```
$ curl --header "X-Vault-Token: $CLUSTER_ROOT_TOKEN" \
    http://127.0.0.1:8200/v1/secret/data/tls/apitest
Client sent an HTTP request to an HTTPS server.
```
