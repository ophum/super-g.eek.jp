---
title: "Kubespray Operations Memo"
date: "2024-08-18T02:23:34+09:00"
draft: true
---

# kubespray を使ったクラスター構築、ノードのスケール手順のメモ

https://github.com/ophum/kakisute/tree/main/kubespray-operations/

##### 2024/08/17 (土)

kubespray を使った運用手順のメモ

## 環境

Vagrant + VirtualBox で構築します。

cplane 5 台, worker 2 台を起動します。

```bash
vagrant up
```

```bash
vagrant status
```

## シナリオ

1. クラスターをセットアップ
1. cplane のノード数を 3 から 5 に変更
1. cplane のノード数を 5 から 3 に変更
1. worker のノード数を 1 から 2 に変更
1. worker のノード数を 2 から 1 に変更

## クラスターセットアップ

```bash
cd kakisute/kubespray-operations/

pipenv install
pipenv shell

ansible-galaxy install -r requirements.yml

pipenv install -r $HOME/.ansible/collections/ansible_collections/kubernetes_
sigs/kubespray/requirements.txt

ansible-playbook -i 00_initial_hosts --become --user vagrant cluster.yml
```

cplane1 に ssh して kubernetes クラスタが構築できているか確認します。

```bash
$ vagrant ssh cplane1
vagrant@cplane1:~$ sudo su -
root@cplane1:~# kubectl get node
NAME      STATUS   ROLES           AGE     VERSION
cplane1   Ready    control-plane   4m48s   v1.30.3
cplane2   Ready    control-plane   4m14s   v1.30.3
cplane3   Ready    control-plane   3m54s   v1.30.3
worker1   Ready    <none>          3m21s   v1.30.3
```

## cplane のノード数を 3 から 5 に変更

cplane を 5 台にスケールアウトします。

hosts の差分は以下の通り

```diff
--- 00_initial_hosts       2024-08-17 22:31:35.327517285 +0900
+++ 10_cplane_scale_out_hosts      2024-08-17 22:33:00.957506755 +0900
@@ -2,8 +2,8 @@
 cplane1 ansible_host=192.168.56.11 ip=192.168.56.11
 cplane2 ansible_host=192.168.56.12 ip=192.168.56.12
 cplane3 ansible_host=192.168.56.13 ip=192.168.56.13
-#cplane4 ansible_host=192.168.56.14 ip=192.168.56.14
-#cplane5 ansible_host=192.168.56.15 ip=192.168.56.15
+cplane4 ansible_host=192.168.56.14 ip=192.168.56.14
+cplane5 ansible_host=192.168.56.15 ip=192.168.56.15
 worker1 ansible_host=192.168.56.21 ip=192.168.56.21
 #worker2 ansible_host=192.168.56.22 ip=192.168.56.22

@@ -11,15 +11,15 @@
 cplane1
 cplane2
 cplane3
-#cplane4
-#cplane5
+cplane4
+cplane5

 [etcd]
 cplane1
 cplane2
 cplane3
-#cplane4
-#cplane5
+cplane4
+cplane5

 [kube_node]
 worker1
```

cluster.yml を実行します。
cplane4 と cplane5 をセットアップし既存のクラスターに join します。

```bash
ansible-playbook -i 10_cplane_scale_out_hosts --become --user vagrant cluster.yml
```

playbook が終了した時点で kubectl get node すると cplane4 と cplane5 が追加されていることがわかります。

```bash
root@cplane1:~# kubectl get node
NAME      STATUS   ROLES           AGE    VERSION
cplane1   Ready    control-plane   36m    v1.30.3
cplane2   Ready    control-plane   36m    v1.30.3
cplane3   Ready    control-plane   35m    v1.30.3
cplane4   Ready    control-plane   5m3s   v1.30.3
cplane5   Ready    control-plane   4m5s   v1.30.3
worker1   Ready    <none>          35m    v1.30.3
```

また etcdctl でメンバー一覧にも追加されていることがわかります。

```bash
root@cplane1:~# etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-cplane1.pem --key=/etc/ssl/etcd/ssl/node-cplane1-key.pem member list
347799df1339cc0, started, etcd1, https://192.168.56.11:2380, https://192.168.56.11:2379, false
42cc40fb31f053f4, started, etcd2, https://192.168.56.12:2380, https://192.168.56.12:2379, false
6a894f2273e9da1e, started, etcd3, https://192.168.56.13:2380, https://192.168.56.13:2379, false
80e23c6e0ebcba45, started, etcd5, https://192.168.56.15:2380, https://192.168.56.15:2379, false
93f6845439d2c9b4, started, etcd4, https://192.168.56.14:2380, https://192.168.56.14:2379, false
```

この時点では、ワーカーノード上にある kube-system/nginx-proxy が古い設定のまま動作しているため kube-system/nginx-proxy を再起動する必要があります。

[nodes.md#2-restart-kube-systemnginx-proxy](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/nodes.md#2-restart-kube-systemnginx-proxy)

kube-system/nginx-proxy は kubespray で HA 構成を構築する際にインストールされるローカル LB です。
ワーカーノードの kubelet はローカルで動いている nginx に問い合わせることで、nginx が各 cplane にロードバランシングします。

[ha-mode.md#kube-apiserver](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/ha-mode.md#kube-apiserver)

また、kube-apiserver の起動コマンドの引数に指定する etcd のサーバーの配列が古いままになっているため、新しくセットアップされた cplane4 と cplane5 の etcd も見るように設定する必要があります。

変わっていないことが確認できます。

```bash
root@cplane1:~# cat /etc/kubernetes/manifests/kube-apiserver.yaml  | grep "etcd-servers"
    - --etcd-servers=https://192.168.56.11:2379,https://192.168.56.12:2379,https://192.168.56.13:2379
```

[nodes.md#2-add-the-new-node-to-apiserver-config](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/nodes.md#2-add-the-new-node-to-apiserver-config)

kube-system/nginx-proxy の再起動と kube-apiserver の設定変更を行う playbook を実行します。

```bash
ansible-playbook -i 10_cplane_scale_out_hosts --become --user vagrant post-cplane-scale.yml
```

## cplane のノード数を 5 から 3 に変更

cplane を 3 台にスケールインします。

今回は cplane1 と cplane2 を消してみます。

まずは kubespray の remove-node プレイブックを利用してノードを削除します。
削除するノードを extra-vars で指定します。
また、hosts の kube_control_plane グループの先頭に指定したノードは削除できない仕様なので順番を入れ替えておきます。

```diff
--- 10_cplane_scale_out_hosts      2024-08-17 22:33:00.957506755 +0900
+++ 20_cplane_scale_in_hosts     2024-08-18 01:17:31.696328448 +0900
@@ -8,11 +8,11 @@
 #worker2 ansible_host=192.168.56.22 ip=192.168.56.22

 [kube_control_plane]
-cplane1
-cplane2
 cplane3
 cplane4
 cplane5
+cplane1
+cplane2

 [etcd]
 cplane1
```

```bash
ansible-playbook -i 20_cplane_scale_in_hosts --become --user vagrant --extra-vars "node=cplane1" remove-node.yml
```

remove-node では drain や cordon といった安全にノードを切り離す処理を行ってくれます。

以下の通り ROLES に SchedulingDisabled が追加されることが確認できます。

```bash
$ vagrant ssh cplane3
vagrant@cplane3:~$ sudo su -
root@cplane3:~# kubectl get node
NAME      STATUS                     ROLES           AGE    VERSION
cplane1   Ready,SchedulingDisabled   control-plane   166m   v1.30.3
cplane2   Ready                      control-plane   165m   v1.30.3
cplane3   Ready                      control-plane   165m   v1.30.3
cplane4   Ready                      control-plane   134m   v1.30.3
cplane5   Ready                      control-plane   133m   v1.30.3
worker1   Ready                      <none>          164m   v1.30.3
```

playbook の実行が終わると cplane1 がなくなっていることが確認できます。

```bash
root@cplane3:~# kubectl get node
NAME      STATUS   ROLES           AGE    VERSION
cplane2   Ready    control-plane   166m   v1.30.3
cplane3   Ready    control-plane   166m   v1.30.3
cplane4   Ready    control-plane   135m   v1.30.3
cplane5   Ready    control-plane   134m   v1.30.3
worker1   Ready    <none>          165m   v1.30.3
```

cplane2 も削除します。

```bash
ansible-playbook -i 21_cplane_scale_in_hosts --become --user vagrant --extra-vars "node=cplane2" remove-node.yml
```

```bash
root@cplane3:~# kubectl get node
NAME      STATUS     ROLES           AGE    VERSION
cplane3   Ready      control-plane   173m   v1.30.3
cplane4   Ready      control-plane   142m   v1.30.3
cplane5   Ready      control-plane   141m   v1.30.3
worker1   NotReady   <none>          172m   v1.30.3
```

cluster-info configmap の server フィールドが cplane1 の ip になっているので編集します。

```
root@cplane3:~# kubectl get cm -n kube-public cluster-info -o yaml | grep server:
        server: https://192.168.56.11:6443
root@cplane3:~# kubectl edit cm -n kube-public cluster-info
root@cplane3:~# kubectl get cm -n kube-public cluster-info -o yaml | grep server:
        server: https://192.168.56.13:6443
```

inventory から cplane1 と cplane2 を消します。

そして、cluster.yml で各種設定を更新します。

```bash
ansible-playbook -i 22_cplane_scale_in_hosts --become --user vagrant cluster.yml
```

cluster.yml では kube-apiserver の設定は更新されないので、post-cplane-scale.yml で更新します。

```bash
ansible-playbook -i 22_cplane_scale_in_hosts --become --user vagrant post-cplane-scale.yml
```

## worker のノード数を 1 から 2 に変更

worker を 2 台にスケールアウトします。

```diff
--- 21_cplane_scale_in_hosts_2     2024-08-18 01:35:14.926201969 +0900
+++ 30_worker_scale_out_hosts      2024-08-18 01:41:27.986155587 +0900
@@ -3,7 +3,7 @@
 cplane4 ansible_host=192.168.56.14 ip=192.168.56.14
 cplane5 ansible_host=192.168.56.15 ip=192.168.56.15
 worker1 ansible_host=192.168.56.21 ip=192.168.56.21
-#worker2 ansible_host=192.168.56.22 ip=192.168.56.22
+worker2 ansible_host=192.168.56.22 ip=192.168.56.22

 [kube_control_plane]
 cplane3
@@ -17,7 +17,7 @@

 [kube_node]
 worker1
-#worker2
+worker2

 [calico_rr]

```

scale.yml を実行します。

```bash
ansible-playbook -i 30_worker_scale_out_hosts --become --user vagrant scale.yml --limit=worker2
```

ノードが追加されていることが確認できます。

```bash
root@cplane3:~# kubectl get node
```

## worker のノード数を 2 から 1 に変更

worker を 1 台にスケールインします。

worker1 を削除します。

```
ansible-playbook -i 40_worker_scale_in_hosts --become --user vagrant --extra-vars "node=worker1" remove-node.yml
```

ノードが削除されていることが確認できます。

```bash
root@cplane3:~# kubectl get node
```

最後に inventory から worker1 を削除し終了です。
