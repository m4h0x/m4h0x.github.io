---
title: "HA構成のGrafanaを構築した際の備忘録"
date: 2025-10-19T21:54:20+09:00
draft: false
summary: |
  RedisとPostgresqlを導入することで，Kubernetes上のGrafanaをHA構成で運用した際の備忘録である．
categories:
  - Kubernetes
tags:
  - Grafana
  - Redis
  - Postgresql
---

## 1. 背景

Kubernetesクラスタを監視するためにPrometheusとGrafanaを導入した．
Grafanaは1台のpodで動くことを基本的には推奨されているが，クラスタ内のほぼすべてのアプリケーションを冗長化していることから，Grafanaも冗長化させたいと思い，いろいろ試した備忘録を残しておく．

## 2. 本題

Grafanaを複数台のpodで動かす場合，各podが共通して使用できるセッションストアとデータベースが必要となる．
本環境ではIngress ControllerによりL7ロードバランシングを行っている．
ここでセッションアフィニティは有効化していないため，外からの通信はランダムに各Podに割り当てられることから，Podローカルのセッション情報では整合性が取れない．
また，GrafanaのデフォルトデータベースであるSQLiteはマルチインスタンス非対応であることから，複数Podからの同時アクセスに対応していない．
そこで，リモートキャッシュとしてRedis，データベースとしてPostgresqlを使用することで，共通のセッションストアとデータベースを使用できるようにし，Grafanaを冗長化させた．
以下にGrafanaを冗長化させた際の手順を示す．

### 2-1. PrometheusとGrafanaの導入

まずはhelmを使ってPrometheusとGrafanaを導入する．その際，Grafanaのpod数は1として，動作確認をした．
```
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories

$ helm repo add grafana https://grafana.github.io/helm-charts
"grafana" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "grafana" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Grafanaにログインでき，かつダッシュボードにnodeやpodを表示させることができたことから，1台のPodを使用して問題なく動作することを確認できた．

### 2-2. Redisの導入

Redisは高可用性構成をとるためにvalues.yamlを書き換えて，Redis Sentinelを使用するようにした．
Redis Sentinel を導入することで，Master障害時に自動フェイルオーバーが可能となる．
Sentinelについての詳しい説明はここを参照してほしい：[High availability with Redis Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm show values bitnami/redis > redis-values.yaml
$ helm install grafana-redis bitnami/redis -n monitoring -f redis-values.yaml
```

### 2-3. PostgreSQLの導入

次にpostgresqlを導入する．はじめはbitnami/postgresql を使おうとしたが， 2024年でHelm chartの配布が終了しており，イメージ取得エラーが発生した．
そこで，postgresqlのオペレーターとしてzalandoを使用し，postgresqlを動かすこととした．

```
$ helm repo add postgres-operator-charts https://opensource.zalando.com/postgres-operator/charts/postgres-operator
```

```
$ cat postgresql.yaml

apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: grafana-postgresql
  namespace: monitoring
spec:
  teamId: "grafana"
  volume:
    size: 10Gi
    storageClass: <STORAGECLASS>
  numberOfInstances: 3
  users:
    grafana: []
  databases:
    grafana: grafana
  postgresql:
    version: "15"
```

### 2-4. Grafanaの設定を変更

ここまででクラスタ内にRedisとPostgresqlを導入できた．
values.yaml内の`grafana.ini`や環境変数にてRedisとPostgresqlを使うよう指定することで，Grafanaがリモートキャッシュやデータベースを使えるようになる．
実際，replicaCountを3にした状態での動作確認は問題なかった．


## 3. 結論

GrafanaをHA構成にするためにはGrafanaの各 Pod が共有のセッションストアとデータベースを使用する必要があった．  
RedisとPostgreSQLを導入し，Grafanaのvalues.yaml内で正しく指定することで，Grafanaを複数台のPodで運用することができた．

現在，Grafanaのvalue.yamlでは，単一のRedisのPodを直接参照する設定になっており，せっかくのRedisの高可用性を活かすことができていない．
今後，RedisもPostgreSQLのようにサービス経由でフェイルオーバーを意識せず使える構造にしたい．



















































































































