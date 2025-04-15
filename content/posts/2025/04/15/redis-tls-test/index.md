---
title: "Redis TLSを試す"
date: "2025-04-15T19:30:06+09:00"
draft: false
tags:
  - redis
  - Kubernetes
---

Redis 6 で対応した TLS を試してみます。
Redis は Kubernetes で動かします。

## 証明書

前回の[cert-manager でプライベート認証局を作ってみる](../cert-manager-selfsigned-ca-test/index.md)で作成した証明書を利用します。

```
$ kubectl -n sandbox get secret example-com-tls
NAME              TYPE                DATA   AGE
example-com-tls   kubernetes.io/tls   3      17h
```

## redis の deployment を作成

以下のような manifest を作成します。

- example-com-tls Secret を/etc/redis/tls にマウントします
- `--tls-port 6379 --port 0`とすることで、非 TLS ポートでの LISTEN を無効にできます
- `--tls-cert-file` `--tls-key-file` `--tls-ca-cert-file`に証明書と鍵を指定します

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: sandbox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          command:
            - redis-server
          args:
            - --tls-port 6379
            - --port 0
            - --tls-cert-file /etc/redis/tls/tls.crt
            - --tls-key-file /etc/redis/tls/tls.key
            - --tls-ca-cert-file /etc/redis/tls/ca.crt
            - --auth-clients no

          ports:
            - containerPort: 6379
          volumeMounts:
            - name: example-com-tls
              mountPath: /etc/redis/tls
              readOnly: true
      volumes:
        - name: example-com-tls
          secret:
            secretName: example-com-tls
```

## テスト

```
$ kubectl get pods -n sandbox
NAME                     READY   STATUS    RESTARTS   AGE
redis-5fc9c6bfc7-tlhrk   1/1     Running   0          7m1s
```

shell に入ります。

```
$ kubectl -n sandbox exec -it redis-5fc9c6bfc7-tlhrk -- ash
/data #
```

redis-cli を起動します。起動できましたが、`keys *`などのコマンドを実行すると `Connection reset by peer`というエラーになります。

```
/data # redis-cli
127.0.0.1:6379> keys *
Error: Connection reset by peer
```

redis-cli に`--tls` オプションを付けて起動してみます。
今度は、`SSL_connect failed: certificate verify failed`となります。証明書の検証ができなかったというエラーです。

```
/data # redis-cli --tls
Could not connect to Redis at 127.0.0.1:6379: SSL_connect failed: certificate verify failed
```

`--cacert` オプションで`/etc/redis/tls/ca.crt` を指定して起動してみます。
うまく接続できました。

```
/data # redis-cli --tls --cacert /etc/redis/tls/ca.crt
127.0.0.1:6379> keys *
(empty array)
```

`--auth-clients no`を `--auth-clients yes`としてクライアント証明書を使ったクライアント認証を有効にしてみます。

```diff
$ diff -u deploy.yaml.old deploy.yaml
--- deploy.yaml.old     2025-04-15 19:45:34.981957295 +0900
+++ deploy.yaml 2025-04-15 19:45:41.187313887 +0900
@@ -24,7 +24,7 @@
           - --tls-cert-file /etc/redis/tls/tls.crt
           - --tls-key-file /etc/redis/tls/tls.key
           - --tls-ca-cert-file /etc/redis/tls/ca.crt
-          - --tls-auth-clients no
+          - --tls-auth-clients yes

         ports:
         - containerPort: 6379
```

再度 shell に入り先ほどのオプションで redis-cli を起動します。
`keys *`を実行するとエラーになります。

```
/data # redis-cli --tls --cacert /etc/redis/tls/ca.crt
127.0.0.1:6379> keys *
Error: Server closed the connection
```

`--cert` `--key`でクライアント証明書と秘密鍵を指定してみます。
エラーにならず実行できました。

```
/data # redis-cli --tls --cacert /etc/redis/tls/ca.crt --cert /etc/redis/tls/tls.crt --key /etc/redis/tls/tls.key
127.0.0.1:6379> keys *
(empty array)
```
