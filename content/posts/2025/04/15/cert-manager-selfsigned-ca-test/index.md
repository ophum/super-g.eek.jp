---
title: "cert-managerでプライベート認証局を作ってみる"
date: "2025-04-15T01:36:10+09:00"
draft: false
---

cert-manager の selfsigned issuer を利用してプライベート認証局を作ってみます。

公式ドキュメントをやるだけ
https://cert-manager.io/docs/configuration/selfsigned/

## プライベート認証局を作成する

namespace を作成

```
$ kubectl create namespace sandbox
```

### 自己署名用の Issuer を作成する

selfSigned な Issuer は与えられた鍵を利用して署名します。(つまり自己署名)

自己署名するので ClusterIssuer としても良かったかも・・・

```bash
$ kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: sandbox
spec:
  selfSigned: {}
EOF
```

### selfsigned-issuer で自己署名した CA 証明書を作成する

```bash
$ kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: sandbox
spec:
  isCA: true
  commonName: my-selfsigned-ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: Issuer
    group: cert-manager.io
EOF
```

### 自己署名した CA 証明書を利用した CA の Issuer を作成する

```bash
$ kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: my-ca-issuer
  namespace: sandbox
spec:
  ca:
    secretName: root-secret
EOF
```

これで、sandbox ネームスペース内では my-ca-issuer を使って証明書を発行することができます。

## 自己署名認証局で証明書を発行する

`spec.usages`に`server auth`と`client auth`を指定してサーバー証明書としてもクライアント証明書としても使える証明書を発行してみます。

`spec.issuerRef`にプライベート認証局の Issuer を指定することでプライベート認証局で証明書を発行できます。

```bash
$ kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: sandbox
spec:
  secretName: example-com-tls
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  duration: 2160h
  renewBefore: 360h
  isCA: false
  usages:
    - server auth
    - client auth
  subject:
    organizations:
      - cert-manager
  commonName: example.com
  dnsNames:
    - example.com
    - www.example.com
  ipAddresses:
    - 127.0.0.1
  issuerRef:
    name: my-ca-issuer
    kind: Issuer
    group: cert-manager.io
EOF
```

example-com が作成され READY=True になっていることがわかります。

```bash
$ kubectl -n sandbox get certificate
NAME               READY   SECRET            AGE
example-com        True    example-com-tls   7m59s
my-selfsigned-ca   True    root-secret       14m
```

example-com-tls シークレットには ca.crt, tls.crt, tls.key の 3 つのデータが入っています。

```bash
$ kubectl -n sandbox get secret example-com-tls
NAME              TYPE                DATA   AGE
example-com-tls   kubernetes.io/tls   3      9m29s
```

### example-com-tls の ca.crt を見てみる

Issuer の CN と Subject の CN が `my-selfsigned-ca`となっており自己署名なことがわかります。
また、Basic Constraints には CA:TRUE があり CA であることがわかります。

```bash
$ kubectl -n sandbox get secrets example-com-tls -o json | jq -r '.data.["ca.crt"]' | base64 -d | openssl x509 -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            7e:56:a6:4c:45:03:92:5e:fe:df:c0:f5:4d:28:a8:5c
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: CN = my-selfsigned-ca
        Validity
            Not Before: Apr 14 16:27:40 2025 GMT
            Not After : Jul 13 16:27:40 2025 GMT
        Subject: CN = my-selfsigned-ca
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:3f:1a:e1:3e:3e:c4:5e:04:95:34:e1:a8:23:67:
                    bd:92:2b:99:22:aa:ad:bd:84:41:1c:54:e7:fd:d6:
                    83:d5:2e:57:de:cc:82:59:bc:9b:a2:45:1d:ae:20:
                    c1:e9:58:38:f1:26:0f:82:9a:ba:f8:77:57:cc:61:
                    eb:af:fd:a2:58
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment, Certificate Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier:
                D5:DF:81:32:68:51:D4:CD:C0:2F:F4:61:6E:93:37:CA:F9:12:B3:A1
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:45:02:20:05:dc:a3:71:19:33:f8:98:d6:80:b7:01:a1:cf:
        67:90:26:ae:f8:09:64:e7:2f:1e:68:d9:f7:86:5a:9d:15:1a:
        02:21:00:f0:5d:29:f0:01:25:74:37:9b:65:92:76:11:0a:16:
        4c:6d:4a:38:4b:d5:17:46:78:cd:64:9d:ee:13:48:cf:cb
```

### example-com-tls の tls.crt を見てみる

Issuer の CN が my-selfsigned-ca となっており自己署名認証局によって発行されていることがわかります。
Subject も `O = cert-manager, CN = example.com`となっています。
また、`Extended Key Usage`には`TLS Web Server Authentication, TLS Web Client Authentication`があり、証明書の用途がサーバー証明書・クライアント証明書ということがわかります。

```bash
$ kubectl -n sandbox get secrets example-com-tls -o json | jq -r '.data.
["tls.crt"]' | base64 -d | openssl x509 -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            20:9d:46:87:da:75:26:e9:65:7d:d2:55:52:40:ed:83
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: CN = my-selfsigned-ca
        Validity
            Not Before: Apr 14 16:34:03 2025 GMT
            Not After : Jul 13 16:34:03 2025 GMT
        Subject: O = cert-manager, CN = example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c7:2f:e9:d8:ee:4b:3a:df:46:46:5d:0f:39:2b:
                    68:4f:13:9c:4e:f6:b0:7c:52:dd:dd:9d:f9:a3:b5:
                    cb:51:96:57:79:7a:f2:37:48:49:60:ec:3d:b6:57:
                    c3:aa:af:76:5f:75:9b:61:01:46:df:51:c1:57:ec:
                    b1:c1:d2:76:a1:26:2d:17:5c:96:c5:4c:bd:50:0a:
                    1b:15:e2:3c:e2:48:ad:02:4a:36:7b:ff:b7:e9:d1:
                    eb:38:be:69:2d:e7:5e:79:9e:73:33:65:74:49:86:
                    97:4f:2a:f6:99:fe:43:03:04:f5:2e:4f:58:75:48:
                    b5:50:a1:eb:0e:3f:bb:9d:f2:92:4d:87:18:c2:6f:
                    d3:e9:95:95:c0:86:13:e2:69:c6:68:39:57:93:5c:
                    9c:fe:89:6d:84:8a:d1:5d:8e:26:dc:9b:f0:61:a5:
                    76:15:b9:5b:3b:f5:91:d9:1f:bc:88:6a:5a:b9:fe:
                    43:da:f8:17:eb:ec:4c:20:43:57:97:70:03:a9:34:
                    3f:56:af:ce:32:d7:e6:7f:c9:30:06:1d:79:ed:9a:
                    e9:92:c0:08:02:4d:e1:b2:d0:66:18:65:67:8c:1a:
                    76:0f:64:54:30:07:cb:29:e4:44:f9:1b:1f:8d:a7:
                    8f:b5:13:19:5b:96:69:03:b8:49:a5:37:9f:47:c5:
                    b9:09
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier:
                D5:DF:81:32:68:51:D4:CD:C0:2F:F4:61:6E:93:37:CA:F9:12:B3:A1
            X509v3 Subject Alternative Name:
                DNS:example.com, DNS:www.example.com, IP Address:127.0.0.1
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:46:02:21:00:f0:0b:48:c4:ff:67:f1:3a:06:ff:6a:db:b1:
        56:4d:86:de:17:2e:17:04:44:4b:b0:97:89:06:61:45:a0:94:
        80:02:21:00:a8:e0:5c:b0:0b:79:a3:10:3c:94:fb:32:fe:92:
        09:a2:43:2f:f6:3e:f8:80:f3:ec:9e:36:cd:b6:06:4a:3b:56
```
