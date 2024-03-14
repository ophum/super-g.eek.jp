---
title: "AnsibleでHashiCorp Vaultに登録した機密情報を使用する"
date: "2022-07-17"
categories:
  - "infra"
---

[HashiCorp Vault](https://www.vaultproject.io/)(以下 vault)を利用してパスワードなどの機密情報を保存し、それを Ansible で利用してみました。

## 検証環境

- Ubuntu on WSL2

```
$ uname -a
Linux ryzen5pc 5.10.16.3-microsoft-standard-WSL2 #1 SMP Fri Apr 2 22:23:49 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04.4 LTS"
```

## セットアップ

### ワークスペース

`~/hashicorp-vault-ansible`というディレクトリで作業することにします。

```bash
$ mkdir ~/hashicorp-vault-ansible
```

### vault のインストール

vault はバイナリをダウンロードして利用するか、各 OS 向けのパッケージマネージャでインストールすることができます。  
今回はバイナリをダウンロードしました。

- ドキュメント [Install Vault](https://learn.hashicorp.com/tutorials/vault/getting-started-install?in=vault/getting-started)

```bash
$ cd ~/hashicorp-vault-ansible
$ curl -O https://releases.hashicorp.com/vault/1.11.0/vault_1.11.0_linux_amd64.zip
$ unzip vault_1.11.0_linux_amd64.zip
$ ./vault version
Vault v1.11.0 (ea296ccf58507b25051bc0597379c467046eb2f1), built 2022-06-17T15:48:44Z
```

### ansible のインストール

pipenv で python をインストールしつつ、pip で ansible をインストールします。

```bash
$ sudo apt update
$ sudo apt install pipenv
$ pipenv --python 3
$ pipenv shell

$ pip install ansible
```

### ansible で vault のデータを扱うためのセットアップ

[community.hashi_vault](https://docs.ansible.com/ansible/latest/collections/community/hashi_vault/index.html)という collection を利用します。

```bash
$ pip install hvac
$ ansible-galaxy collection install community.hashi_vault
```

## vault を起動する

以下のコマンドで vault dev server を起動できます

```bash
$ vault server -dev
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.17.11
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.11.0, built 2022-06-17T15:48:44Z
             Version Sha: ea296ccf58507b25051bc0597379c467046eb2f1

==> Vault server started! Log data will stream in below:
```

Api Address (http://127.0.0.1:8200/)にブラウザでアクセスするとUIを利用できます。

![](images/vault-ui-1024x651.png)

ログの最後に以下のようなメッセージが出力されるので、この Root Token(`hvs.0KXrA6oFiZnLBJpvoMnPc4vB`) を利用してログインします。

```bash
WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: fxwAHiHVyi3vQdHGiFfTs4KuQUBkGtHZr0aELRfNTwg=
Root Token: hvs.0KXrA6oFiZnLBJpvoMnPc4vB

Development mode should NOT be used in production installations!
```

## kv の secret engine を追加する

ログインすると以下のように作成済みの secret が見れます。dev 環境のため`secret`という key/value storage が作成されています。  
今回は、新規で作成してみます。

![](images/vault-secret-list-1024x652.png)

1. テーブルの右上にある`Enable new secret`をクリックし作成ページに遷移します。

2. `KV`を選択し`Next`をクリックします。

![](images/vault-enable-kv-secret-1024x650.png)

3. Path に`kv-test`を指定し`Enable Engine`をクリックします。  
   (デフォルトでは`kv`が指定されてます)

![](images/vault-enable-kv-secret-2-1024x725.png)

4. 作成された secret engine のページに遷移されます。

![](images/vault-enable-kv-secret-3-1024x727.png)

## `kv-test`に機密情報を登録する。

1. 先ほどの`kv-test`のページのテーブルの右上にある`Create secret`をクリックし作成ページに遷移します。

2. `Path for this secret`にシークレットの名前を入力し、`Secret data`に機密情報を Key/Value で入力します。  
   今回はシークレットの名前を`mysql-prd`、機密情報を`username=prd-user`、 `password=prd-password`としてみました。

![](images/vault-create-secret-1-1024x729.png)

3. 作成されたシークレットのページに遷移されます。

![](images/vault-create-secret-2-1024x464.png)

## Ansible で取得する。

### token の設定

Ansible で Vault を利用するために使用する Token を設定します。  
playbook に直接記入できますが、メリットはないので ansible.cfg などに記述し git 管理しないようにします。

```bash
$ cat <<EOF > ansible.cfg
[hashi_vault_collection]
token = "hvs.jkOyqGirKFHOLmpgIvneu52B"
url = "http://127.0.0.1:8200"
EOF
$ echo "ansible.cfg" >> .gitignore
```

### playbook の記述

kv(version 2) の secret engine からデータを取得する場合[`community.hashi_vault.vault_kv2_get`](https://docs.ansible.com/ansible/latest/collections/community/hashi_vault/vault_kv2_get_lookup.html#ansible-collections-community-hashi-vault-vault-kv2-get-lookup)を利用します。

lookup の第 2 引数にシークレットの名前を指定します。また、engine の名前(`kv-test`)を`engine_mount_point`で指定します。

```bash
cat <<EOF > playbook.yml
- hosts: localhost
  gather_facts: false
  tasks:
  - name: kv-testのデータを取得
    ansible.builtin.set_fact:
      response: "{{ lookup('community.hashi_vault.vault_kv2_get', 'mysql-prd', engine_mount_point='kv-test') }}"
  - name: 結果を表示
    ansible.builtin.debug:
      msg:
        - "Secret: {{ response.secret }}"
        - "Data: {{ response.data }}"
        - "Metadata: {{ response.metadata}}"
        - "Full reponse: {{ response.raw }}"
        - "Value of key 'username' in the secret: {{ response.secret.username }}"
        - "Value of key 'password' in the secret: {{ response.secret.password }}"
EOF
```

### 実行する。

先ほど UI で登録したデータを取得できていることが分かります。

```bash
$ ansible-playbook playbook.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] **********************************************************************************************************************

TASK [kv-testのデータを取得] **********************************************************************************************************
[DEPRECATION WARNING]: The default value for 'token_validate' will change from True to False. This feature will be removed from
community.hashi_vault in version 4.0.0. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.
ok: [localhost]

TASK [結果を表示] *********************************************************************************************************************
ok: [localhost] => {
    "msg": [
        "Secret: {'password': 'prd-password', 'username': 'prd-user'}",
        "Data: {'data': {'password': 'prd-password', 'username': 'prd-user'}, 'metadata': {'created_time': '2022-07-17T11:08:23.918173727Z', 'custom_metadata': None, 'deletion_time': '', 'destroyed': False, 'version': 1}}",
        "Metadata: {'created_time': '2022-07-17T11:08:23.918173727Z', 'custom_metadata': None, 'deletion_time': '', 'destroyed': False, 'version': 1}",
        "Full reponse: {'request_id': '8f43c6e5-9d3a-7dae-3121-2986e441cca9', 'lease_id': '', 'renewable': False, 'lease_duration': 0, 'data': {'data': {'password': 'prd-password', 'username': 'prd-user'}, 'metadata': {'created_time': '2022-07-17T11:08:23.918173727Z', 'custom_metadata': None, 'deletion_time': '', 'destroyed': False, 'version': 1}}, 'wrap_info': None, 'warnings': None, 'auth': None}",
        "Value of key 'username' in the secret: prd-user",
        "Value of key 'password' in the secret: prd-password"
    ]
}

PLAY RECAP ****************************************************************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## さいごに

今回の検証で、Playbook と利用する機密情報を別々に管理することでより安全に機密情報を扱えるようになりました。  
また、今回は Token に Root Token を利用しましたが、Ansible 実行用の Token や Policy を作成し利用することでさらに実用的になると思うので今後調べていきたいです。

## 参考文献

- [HashiCorp Learn Vault Getting Started](https://learn.hashicorp.com/collections/vault/getting-started)

- [community.hashi_vault.vault_kv2_get lookuphashi_vault – retrieve secrets from HashiCorp’s vault](https://docs.ansible.com/ansible/latest/collections/community/hashi_vault/vault_kv2_get_lookup.html)
