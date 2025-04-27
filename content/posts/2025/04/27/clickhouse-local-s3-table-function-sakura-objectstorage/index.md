---
title: "clickhouse-localでさくらのオブジェクトストレージに保存したログを解析する"
date: "2025-04-27T13:50:29+09:00"
draft: false
---

# clickhouse local + さくらのオブジェクトストレージ 検証

検証内容

以下のようなログファイルをさくらのオブジェクトストレージに保存します。

- JSON 形式のログファイル (さくらのレンタルサーバーのログの項目をランダム生成する)
- 1 分単位のファイル
  - 1 ファイル毎に 20MiB ぐらいのサイズになるようにする
  - 1 時間分 00:00 ~ 00:59 の計 60 ファイル = 1.2GiB
  - 24 時間分 00:00 ~ 23:59 の計 1440 ファイル = 28.125GiB
- 1 時間単位のファイル
  - 1 ファイル毎に 1200MiB ぐらいのサイズになるようにする
    - 1 分単位\*60
  - 24 時間分 00:00~23:00 の計 24 ファイル (データのアップロードに時間かかるのでやらなかった)

これを clickhouse-local の s3 テーブル関数で読み込んで集計してみます。

その際に以下の点に注目してみたいです。

- 集計速度
  - 全体のリクエスト数
    - `SELECT COUNT(*) FROM s3('s3://log-test/test.log.*.json')`
  - Host ごとのリクエスト数
    - `SELECT Host, COUNT(*) FROM s3('s3://log-test/test.log.*.json') GROUP BY Host`
  - ある Host の GET リクエスト数
    - `SELECT Host, Method, COUNT(*) FROM s3('s3://log-test/test.log.*.json') WHERE Host = 'ホスト名を入れる' AND Method = 'GET' GROUP BY Host, Method`

## 検証環境

さくらのクラウドに以下のスペックのサーバーを作成しました

- CPU: 8 コア
- Memory: 16GB
- Disk: 100GB
- OS: Ubuntu 24.04.1

## clickhouse-local をセットアップする

公式の Getting Started で紹介されているスクリプトでバイナリをダウンロードします。

```
ubuntu@clickhouse-local-test:~$ curl https://clickhouse.com/ | sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2911    0  2911    0     0   9543      0 --:--:-- --:--:-- --:--:--  9575

Will download https://builds.clickhouse.com/master/amd64/clickhouse into clickhouse

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  130M  100  130M    0     0  39.1M      0  0:00:03  0:00:03 --:--:-- 39.1M

Successfully downloaded the ClickHouse binary, you can run it as:
    ./clickhouse

You can also install it:
sudo ./clickhouse install
```

最後に `sudo ./clickhouse install`と出力されていますが、サーバーとして動かさないので実効不要です。

`clickhouse`というバイナリが保存されているので `./clickhouse local`とサブコマンドで `local`を指定すると clickhouse-local を起動できます。

```
ubuntu@clickhouse-local-test:~$ ./clickhouse local
Decompressing the binary.....
ClickHouse local version 25.5.1.1620 (official build).

:)
```

`exit`で抜けます。

```
:) exit
Bye.
```

## 設定ファイル

clickhouse 用の設定ファイルを用意します。
今回はさくらのオブジェクトストレージの認証情報が必要なため設定ファイルで指定しておきます。
クエリ上で直接指定できます。

access_key_id と secret_access_key は利用するオブジェクトストレージのバケットにアクセスできるモノを記入します。

`url_scheme_mappers`は `s3://{bucket名}/...`というように s3 という scheme で指定したときに url に変換してくれる設定です。
さくらのオブジェクトストレージの URL になるようにします。

以下のような xml ファイルを作成します。

```xml
<clickhouse>
    <s3>
        <sakura-s3>
            <endpoint>https://s3.isk01.sakurastorage.jp</endpoint>
            <access_key_id>access-key-id-xxxxxxxxxxx</access_key_id>
            <secret_access_key>secret-access-key-xxxxxxxxxxxxxxxxxxxxxxxxxxxx</secret_access_key>
            <region>jp-north-1</region>
        </sakura-s3>
    </s3>
    <url_scheme_mappers>
        <s3>
            <to>https://s3.isk01.sakurastorage.jp/{bucket}</to>
        </s3>
    </url_scheme_mappers>
</clickhouse>
```

## 設定ファイルを用いて clickhouse-local を起動する

`--config`で指定して起動します。

```
ubuntu@clickhouse-local-test:~$ ./clickhouse local --config config.xml
ClickHouse local version 25.5.1.1620 (official build).

:)
```

## ログファイルの生成

適当にログを生成しさくらのオブジェクトストレージにアップロードするツールを作りました

https://github.com/ophum/kakisute/blob/d65683af1b945ef1288a943a7cf01c8e8f16d44a/sakura-rs-access-log-gen-and-upload-s3/main.go

```
$ go run main.go -h
Usage of /tmp/go-build3636531536/b001/exe/main:
  -file-size int
        file size (MiB) (default 20)
  -partition-duration string
        partition (default "1m")
  -prefix string
        prefix (default "test.log.")
```

## s3 テーブル関数を用いてデータを集計してみる

### 1 分単位を 1 時間分

```
$ go run main.go -file-size 20 -partition-duration 1m -prefix test.log.
```

#### 全体のリクエスト数

約 524 万行あることがわかりました。
約 3.3 秒で集計できています。
またメモリ使用量は 321MiB です。

```
:) SELECT COUNT(*) FROM s3('s3://log-test/test.log.*.json')

SELECT COUNT(*)
FROM s3('s3://log-test/test.log.*.json')

Query id: 4f306102-f522-446b-ad58-bbbe2bed7d2e

   ┌─COUNT()─┐
1. │ 5238536 │ -- 5.24 million
   └─────────┘

1 row in set. Elapsed: 3.312 sec. Processed 5.11 million rows, 1.22 GB (1.54 million rows/s., 368.54 MB/s.)
Peak memory usage: 320.37 MiB.
```

#### Host ごとのリクエスト数

```
:) SELECT Host, COUNT(*) FROM s3('s3://log-test/test.log.*.json') GROUP BY Host

SELECT
    Host,
    COUNT(*)
FROM s3('s3://log-test/test.log.*.json')
GROUP BY Host

Query id: c8f19e46-40a3-4336-93ad-bf704559162b

    ┌─Host─────────────────────────────────┬─COUNT()─┐
 1. │ 16f93c0e-39aa-470f-97de-8e79f59ec594 │  524282 │
 2. │ 62e704d7-4abb-4aaa-9339-9ef373cdc37d │  523668 │
 3. │ 4b7576ac-541a-4b5a-83f0-6d26c9fe8899 │  524129 │
 4. │ 3f8607af-932d-4210-b4c6-f17a68b38ca6 │  524515 │
 5. │ 4d7f8f57-9a96-4fd1-9207-f5bd799cc6e4 │  524448 │
 6. │ 130db589-db2a-4574-8685-9c2079be8cef │  523224 │
 7. │ 464277ce-9869-4410-a2e4-d7bce121d560 │  524247 │
 8. │ 33552391-3ae2-47a8-9958-ee41c755b9a5 │  524228 │
 9. │ c753e7bc-4317-4536-bca0-376f0ef14987 │  523009 │
10. │ c04d42e1-3f5b-458c-8d02-55efc5bad352 │  522786 │
    └──────────────────────────────────────┴─────────┘

10 rows in set. Elapsed: 3.421 sec. Processed 5.15 million rows, 1.24 GB (1.51 million rows/s., 361.69 MB/s.)
Peak memory usage: 385.07 MiB.
```

#### ある Host の GET リクエスト数

先ほどのクエリで出てきたホスト `16f93c0e-39aa-470f-97de-8e79f59ec594`を指定してみます。

```
:) SELECT Host, Method, COUNT(*) FROM s3('s3://log-test/test.log.*.json') WHERE Host = '16f93c0e-39aa-470f-97de-8e79f59ec594' AND Method = 'GET' GROUP BY Host, Method

SELECT
    Host,
    Method,
    COUNT(*)
FROM s3('s3://log-test/test.log.*.json')
WHERE (Host = '16f93c0e-39aa-470f-97de-8e79f59ec594') AND (Method = 'GET')
GROUP BY
    Host,
    Method

Query id: 58fc2359-74b6-4b02-8ce2-077774babe1f

   ┌─Host─────────────────────────────────┬─Method─┬─COUNT()─┐
1. │ 16f93c0e-39aa-470f-97de-8e79f59ec594 │ GET    │  104543 │
   └──────────────────────────────────────┴────────┴─────────┘

1 row in set. Elapsed: 3.331 sec. Processed 5.15 million rows, 1.24 GB (1.55 million rows/s., 371.47 MB/s.)
Peak memory usage: 351.06 MiB.
```

### 1 分単位を 24 時間分

#### 全体のリクエスト数

約 1 億 2 千万行あることがわかりました。
約 1 分で集計できています。
またメモリ使用量は 350MiB です。

```
:) SELECT COUNT(*) FROM s3('s3://log-test/test.log.*.json')

SELECT COUNT(*)
FROM s3('s3://log-test/test.log.*.json')

Query id: 866f94a1-0f10-42b0-b147-435f934010d1

   ┌───COUNT()─┐
1. │ 125725065 │ -- 125.73 million
   └───────────┘

1 row in set. Elapsed: 54.106 sec. Processed 125.55 million rows, 30.16 GB (2.32 million rows/s., 557.37 MB/s.)
Peak memory usage: 350.74 MiB.
```

#### Host ごとのリクエスト数

```
:) SELECT Host, COUNT(*) FROM s3('s3://log-test/test.log.*.json') GROUP BY Host

SELECT
    Host,
    COUNT(*)
FROM s3('s3://log-test/test.log.*.json')
GROUP BY Host

Query id: b5fe6585-8e26-4a4d-8b38-23214e996adc

    ┌─Host─────────────────────────────────┬──COUNT()─┐
 1. │ 5373b13f-83a5-41e8-a608-0ba9d9df735b │ 12577026 │
 2. │ 54a00df2-6b1e-4e97-9997-2472729dda45 │ 12570650 │
 3. │ 1327cdee-bf78-4323-8e99-8e734264ab28 │ 12575398 │
 4. │ f945db8c-b51b-4569-aee6-2b8e648c5280 │ 12573749 │
 5. │ fbf2f05f-0557-4762-b6de-1d275b36fa03 │ 12574135 │
 6. │ be0adb19-2252-44ff-91d0-fa065f2b411c │ 12577024 │
 7. │ 3b60107b-b490-4ccb-8805-9cc2dbd720b8 │ 12568226 │
 8. │ e2af42db-fcc9-470d-9cf3-e0986c4f3798 │ 12572376 │
 9. │ 6e5a1e56-fe76-412f-91da-1ee48e07b66f │ 12569245 │
10. │ 32cac4ca-f3ff-4739-adb5-a0b1ebc1aff2 │ 12567236 │
    └──────────────────────────────────────┴──────────┘

10 rows in set. Elapsed: 53.752 sec. Processed 125.62 million rows, 30.17 GB (2.34 million rows/s., 561.24 MB/s.)
Peak memory usage: 405.77 MiB.
```

#### ある Host の GET リクエスト数

先ほどのクエリで出てきたホスト `5373b13f-83a5-41e8-a608-0ba9d9df735b`を指定してみます。

```
:) SELECT Host, Method, COUNT(*) FROM s3('s3://log-test/test.log.*.json') WHERE Host = '5373b13f-83a5-41e8-a608-0ba9d9df735b' AND Method = 'GET' GROUP BY Host, Method

SELECT
    Host,
    Method,
    COUNT(*)
FROM s3('s3://log-test/test.log.*.json')
WHERE (Host = '5373b13f-83a5-41e8-a608-0ba9d9df735b') AND (Method = 'GET')
GROUP BY
    Host,
    Method

Query id: 2d8dbb28-eab7-4de9-ab24-6b8675011165

   ┌─Host─────────────────────────────────┬─Method─┬─COUNT()─┐
1. │ 5373b13f-83a5-41e8-a608-0ba9d9df735b │ GET    │ 2514744 │
   └──────────────────────────────────────┴────────┴─────────┘

1 row in set. Elapsed: 52.656 sec. Processed 125.70 million rows, 30.19 GB (2.39 million rows/s., 573.32 MB/s.)
Peak memory usage: 373.30 MiB.
```

### 1 時間単位を 1 時間分

#### 全体のリクエスト数

約 500 万行あることがわかりました。
約 6 秒で集計できています。
またメモリ使用量は 40MiB です。

```
:) SELECT COUNT(*) FROM s3('s3://log-test/test-per-hour.log.20250401T0000.json')

SELECT COUNT(*)
FROM s3('s3://log-test/test-per-hour.log.20250401T0000.json')

Query id: 2de780f7-32f4-4bcf-8dc5-d831852f8cbb

   ┌─COUNT()─┐
1. │ 5238512 │ -- 5.24 million
   └─────────┘

1 row in set. Elapsed: 6.678 sec. Processed 5.17 million rows, 1.24 GB (773.77 thousand rows/s., 185.75 MB/s.)
Peak memory usage: 40.34 MiB.
```

#### Host ごとのリクエスト数

```
:) SELECT Host, COUNT(*) FROM s3('s3://log-test/test-per-hour.log.20250401T0000.json') GROUP BY Hos


SELECT
    Host,
    COUNT(*)
FROM s3('s3://log-test/test-per-hour.log.20250401T0000.json')
GROUP BY Host

Query id: f1fce0d9-076a-4c00-a239-68f5cb4ccb22

    ┌─Host─────────────────────────────────┬─COUNT()─┐
 1. │ 23a3a90e-f51b-4447-afca-5032802fa3d9 │  524869 │
 2. │ 54ae05cf-a467-4786-9ef9-ac4852900567 │  523585 │
 3. │ 18ee77d0-e85d-4cbe-9a5e-86501347b90c │  523630 │
 4. │ 00e637d7-cca5-4ac1-86d5-e6087ccdbe4a │  524375 │
 5. │ f03fac7b-6b10-473a-b22f-b237284c72bc │  523721 │
 6. │ 73ea37e5-adc1-45e4-bb64-d271d715204f │  523161 │
 7. │ f80a225b-0309-48ba-80c0-5b9caa7d6256 │  523469 │
 8. │ cf38bfd2-80ec-476d-9673-de9693938523 │  523993 │
 9. │ 0f6aa308-cd65-4285-be46-894f929d31c2 │  524101 │
10. │ e2e74442-d82b-42af-a9dd-16b93c05d396 │  523608 │
    └──────────────────────────────────────┴─────────┘

10 rows in set. Elapsed: 5.474 sec. Processed 5.10 million rows, 1.22 GB (931.99 thousand rows/s., 223.73 MB/s.)
Peak memory usage: 46.94 MiB.
```

#### ある Host の GET リクエスト数

先ほどのクエリで出てきたホスト `23a3a90e-f51b-4447-afca-5032802fa3d9`を指定してみます。

```
:) SELECT Host, Method, COUNT(*) FROM s3('s3://log-test/test-per-hour.log.20250401T0000.json') WHERE Host = '23a3a90e-f51b-4447-afca-5032802fa3d9' AND Method = 'GET' GROUP BY Host, Method

SELECT
    Host,
    Method,
    COUNT(*)
FROM s3('s3://log-test/test-per-hour.log.20250401T0000.json')
WHERE (Host = '23a3a90e-f51b-4447-afca-5032802fa3d9') AND (Method = 'GET')
GROUP BY
    Host,
    Method

Query id: 48aae404-24c0-4126-94f8-6d398c634f9c

   ┌─Host─────────────────────────────────┬─Method─┬─COUNT()─┐
1. │ 23a3a90e-f51b-4447-afca-5032802fa3d9 │ GET    │  104977 │
   └──────────────────────────────────────┴────────┴─────────┘

1 row in set. Elapsed: 5.612 sec. Processed 5.23 million rows, 1.25 GB (932.44 thousand rows/s., 222.35 MB/s.)
Peak memory usage: 47.99 MiB.
```

## 検証結果

| ファイルの分け方 | 範囲      | 検証項目                      | 実行時間  | 最大メモリ使用量 |
| ---------------- | --------- | ----------------------------- | --------- | ---------------- |
| per minute       | 1 時間分  | 全体のレコード数              | 3.312 秒  | 320.37MiB        |
|                  |           | Host ごとのレコード数         | 3.421 秒  | 385.07MiB        |
|                  |           | ある Host の GET リクエスト数 | 3.331 秒  | 351.06MiB        |
|                  | 24 時間分 | 全体のレコード数              | 54.106 秒 | 350.74MiB        |
|                  |           | Host ごとのレコード数         | 53.752 秒 | 405.77MiB        |
|                  |           | ある Host の GET リクエスト数 | 52.656 秒 | 373.30MiB        |
| per hour         | 1 時間分  | 全体のレコード数              | 6.678 秒  | 40.34MiB         |
|                  |           | Host ごとのレコード数         | 5.474 秒  | 46.94MiB         |
|                  |           | ある Host の GET リクエスト数 | 5.612 秒  | 47/99MiB         |

---

追記

Parquet ファイルにエクスポートしたり、clickhouse-local でテーブルを作成し s3 テーブル関数から取得したレコードを INSERT するなどを試していたけど、そのメモが吹っ飛んでた・・・。

JSON で 30GB のログを Parquet ファイルにすると 100MB ほどになりました。
テーブルに INSERT した場合は 2GB ほどになってました。

生成したログが Host 名や Method が違う以外は同じ文字列、数値なので圧縮効率が良かったということだと思います。
しかし、それを踏まえてもかなり容量削減を期待できると思います。

Parquet にしたときに 100MB くらいということもあってクエリの実行速度も JSON に比べて速かったです。
ログを保存するときは S3 Backed Engine で ClickHouse のデータ形式でオブジェクトストレージに置いたり、Parquet で置いたりするとオブジェクトストレージとの通信量が減ったり読み込みを効率化できてよいなと思いました。

(なお、S3 Backed Engine はまだ試せてない・・・)
