---
title: "Helm ChartをOCIレジストリで公開する"
date: "2025-03-18T23:00:09+09:00"
draft: false
tags:
  - kubernetes
  - Helm
---

Helm Chart を OCI イメージにしてレジストリで管理できるらしいので試してみる。
いわゆるコンテナイメージをコンテナレジストリで公開する感じ。

v3.8.0 からデフォルトで対応している。

> Beginning in Helm 3, you can use container registries with OCI support to store and share chart packages. Beginning in Helm v3.8.0, OCI support is enabled by default.

https://helm.sh/docs/topics/registries/

## テスト用の Chart を作成

```
hum@ryzen5pc:~$ helm create helm-oci-test
Creating helm-oci-test
hum@ryzen5pc:~$ cd helm-oci-test/
hum@ryzen5pc:~/helm-oci-test$ tree
.
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

4 directories, 10 files
```

## package 化

tgz を作ります。

```
hum@ryzen5pc:~/helm-oci-test$ helm package .
Successfully packaged chart and saved it to: /home/hum/helm-oci-test/helm-oci-test-0.1.0.tgz
```

## OCI レジストリに push する

OCI レジストリにはさくらのコンテナレジストリを利用しました。

### レジストリにログイン

```
hum@ryzen5pc:~/helm-oci-test$ helm registry login -u helm-test *******.sakuracr.jp
Password:
Login Succeeded
```

※ ホスト名は伏字にしています。

`~/.config/helm/registry/config.json`に docker config と同じ形式で認証情報が保存されていることが確認できます。

```
hum@ryzen5pc:~/helm-oci-test$ cat ~/.config/helm/registry/config.json
{
        "auths": {
                "********.sakuracr.jp": {
                        "auth": "*************************"
                }
        }
}
```

※ホスト名とパスワードは伏字にしています。

### レジストリにパッケージを push する

```
hum@ryzen5pc:~/helm-oci-test$ helm push helm-oci-test-0.1.0.tgz oci://********/.sakuracr.jp/
Pushed: *******.sakuracr.jp/helm-oci-test:0.1.0
Digest: sha256:48716f11a298c99ceb32b4856745ad895876b93ae16bdb668fedc246e76f8737
```

※ホスト名は伏字にしています。

チャート名がイメージ名に、チャートバージョンがタグになっていることがわかります。

### oras を使ってチャート(イメージ)一覧を見る

現在(2025 年 3 月 18 日)イメージ一覧をコントロールパネルで見ることができないので、oras というツールを使って見てみます。

`docker login`してから`oras repo list <レジストリ名>`として確認してみます。

```
hum@ryzen5pc:~/helm-oci-test$ docker login *******sakuracr.jp
Username: helm-test

i Info → A Personal Access Token (PAT) can be used instead.
         To create a PAT, visit https://app.docker.com/settings


Password:
Login Succeeded
hum@ryzen5pc:~/helm-oci-test$ oras repo list *******.sakuracr.jp
helm-oci-test
```

helm-oci-test というレポジトリがあることがわかります。

`oras repo tags <レジストリ名>/<レポジトリ名>`とすることでタグ一覧を確認します。

```
hum@ryzen5pc:~/helm-oci-test$ oras repo tags *******.sakuracr.jp/helm-oci-test
0.1.0
```

タグが 0.1.0 があることがわかります。

## バージョンを上げてみる

Chart.yaml の version を上げて push して見ます。

### Chart.yaml のバージョンを編集

version を 0.2.0 にします

```
hum@ryzen5pc:~/helm-oci-test$ cat Chart.yaml | yq '.version = "0.2.0"' -y | tee Chart.yaml
apiVersion: v2
name: helm-oci-test
description: A Helm chart for Kubernetes
type: application
version: 0.2.0
appVersion: 1.16.0
```

### package

helm-oci-test-0.2.0.tgz ができます。

```
hum@ryzen5pc:~/helm-oci-test$ helm package .
Successfully packaged chart and saved it to: /home/hum/helm-oci-test/helm-oci-test-0.2.0.tgz
```

### push

```
hum@ryzen5pc:~/helm-oci-test$ helm push helm-oci-test-0.2.0.tgz oci://*******.sakuracr.jp/
Pushed: *******.sakuracr.jp/helm-oci-test:0.2.0
Digest: sha256:c1cafe18066a2691f6743a70bfdbcdb1d3da5c82aecb29760732ddaa47414ea4
```

### oras repo tags

helm-oci-test の tags に 0.2.0 が追加されました。

```
hum@ryzen5pc:~/helm-oci-test$ oras repo tags *******.sakuracr.jp/helm-oci-test
0.1.0
0.2.0
```

## helm install で使ってみる

docker desktop の k8s にインストールしてみます。

```
hum@ryzen5pc:~/helm-oci-test$ kubectl get node
NAME             STATUS   ROLES           AGE    VERSION
docker-desktop   Ready    control-plane   2d7h   v1.32.2
```

### 0.1.0 をインストールする

```
hum@ryzen5pc:~/helm-oci-test$ helm install myrelease oci://*******.sakuracr.jp/helm-oci-test --version 0.1.0
Pulled: *******.sakuracr.jp/helm-oci-test:0.1.0
Digest: sha256:48716f11a298c99ceb32b4856745ad895876b93ae16bdb668fedc246e76f8737
NAME: myrelease
LAST DEPLOYED: Tue Mar 18 23:26:49 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=helm-oci-test,app.kubernetes.io/instance=myrelease" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT
```

pod が作成されていることがわかります。

```
hum@ryzen5pc:~/helm-oci-test$ kubectl get pods
NAME                                       READY   STATUS    RESTARTS   AGE
myrelease-helm-oci-test-6d9c77c7c9-8bph4   1/1     Running   0          17s
```

### 0.2.0 にアップグレードする

```
hum@ryzen5pc:~/helm-oci-test$ helm upgrade myrelease oci://*******.sakuracr.jp/helm-oci-test --vers
ion 0.2.0
Pulled: *******.sakuracr.jp/helm-oci-test:0.2.0
Digest: sha256:c1cafe18066a2691f6743a70bfdbcdb1d3da5c82aecb29760732ddaa47414ea4
Release "myrelease" has been upgraded. Happy Helming!
NAME: myrelease
LAST DEPLOYED: Tue Mar 18 23:28:25 2025
NAMESPACE: default
STATUS: deployed
REVISION: 2
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=helm-oci-test,app.kubernetes.io/instance=myrelease" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT
```

```
hum@ryzen5pc:~/helm-oci-test$ kubectl get pods -L helm.sh/chart
NAME                                       READY   STATUS    RESTARTS   AGE   CHART
myrelease-helm-oci-test-5957b5c694-vt6cq   1/1     Running   0          58s   helm-oci-test-0.2.0
```

## レポジトリ名のパスを切ってみる

helm push で oci の URL のパスを `/`を指定するとチャート名がレポジトリ名となりました。

例えばパスを `/charts`とすると、そのパス配下にチャートのレポジトリが作成されます。

※ `charts`は任意の文字列

```
hum@ryzen5pc:~/helm-oci-test$ helm push helm-oci-test-0.2.0.tgz oci://*******.sakuracr.jp/charts
Pushed: *******.sakuracr.jp/charts/helm-oci-test:0.2.0
Digest: sha256:c1cafe18066a2691f6743a70bfdbcdb1d3da5c82aecb29760732ddaa47414ea4
```

さらにパスを切ってみる

```
hum@ryzen5pc:~/helm-oci-test$ helm push helm-oci-test-0.2.0.tgz oci://*******.sakuracr.jp/charts/test
Pushed: *******.sakuracr.jp/charts/test/helm-oci-test:0.2.0
Digest: sha256:c1cafe18066a2691f6743a70bfdbcdb1d3da5c82aecb29760732ddaa47414ea4
```

`oras repo list` で見るとちゃんとパスを切っていてもレポジトリとして判定されています。

```
hum@ryzen5pc:~/helm-oci-test$ oras repo list *******.sakuracr.jp
charts/helm-oci-test
charts/test/helm-oci-test
helm-oci-test
```

パスを切ることで OCI イメージをグルーピングできます。
通常のコンテナイメージを管理しつつ、helm chart を管理する場合はパスを切ったほうが管理しやすそうです。

## OCI イメージを削除してみる

現在(2025 年 3 月 18 日時点)は、コントロールパネルでイメージの一覧を確認できません。
確認できないので削除もできません。

そのため、イメージを削除するには API 経由で行う必要があります。
この API を oras から実行できます。

```
hum@ryzen5pc:~/helm-oci-test$ oras manifest delete *******.sakuracr.jp/charts/helm-oci-test:0.2.0
Are you sure you want to delete the manifest "sha256:c1cafe18066a2691f6743a70bfdbcdb1d3da5c82aecb29760732ddaa47414ea4" and all tags associated with it? [y/N] y
Deleted [registry] *******.sakuracr.jp/charts/helm-oci-test:0.2.0
```

削除されました。

```
hum@ryzen5pc:~/helm-oci-test$ oras repo tags *******.sakuracr.jp/helm-oci-test
0.1.0
```
