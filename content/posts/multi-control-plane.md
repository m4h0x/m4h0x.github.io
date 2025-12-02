---
title: "Kubernetesクラスタ構成変更時の備忘録"
date: 2025-10-05T21:54:20
draft: false
summary: |
  単一のcontrol-plane構成で立ち上げたクラスタを，HAProxyとKeepalivedを用いて複数のcontrol-planeを使用する構成へと移行した際の備忘録である．
categories:
  - Kubernetes
tags:
  - HAProxy
  - keepalived
---

## 1. 背景

1台がcontrol-plane，2台がworkerの計3台のノードで構成されるKubernetesクラスタに対し，ノードの総入れ替えを行い，HA構成に変更した．1台のcontrol-planeを動かす想定で作成されたクラスタを複数台のcontrol-planeを使用する構成に変更することは公式では推奨しておらず，また，情報があまり見つからなかったことからいろいろと苦戦したため，備忘録を残しておく．

## 2. 本題

複数台のnodeをcontrol-planeとして動かす場合，どのcontrol-planeに対しkube-apiserverへアクセスを行えばいいのかといった問題が生じる．1台のcontrol-planeだけに送るということも可能だが，単一障害点となりうるため，HA構成にしたメリットがなくなる．そこで，複数のcontrol-planeを束ねて，一つの仮想的なエンドポイントを提供する．クライアントはこのエンドポイント宛にリクエストを送信することで，仮に一台のcontrol-planeが壊れたとしても同じエンドポイント宛ての通信で他のcontrol-planeを使用することができる．

今回はロードバランサーとしてHAProxyを使用した．また，HAProxy の手前にKeepalivedで仮想IPを立て、その仮想IPに対してHAProxyが待ち受けるといった構成にした．構成図は次の通りになる．
```mermaid
flowchart LR
    subgraph ClientSide[ ]
        C[Client]
    end

    subgraph VIP[Keepalived<br/>Virtual IP]
        V[VIP<br/>(192.168.1.100)]
    end

    subgraph LB[HAProxy Layer]
        H1[HAProxy Node 1]
        H2[HAProxy Node 2]
        H3[HAProxy Node 3]
    end

    subgraph CP[control-plane Nodes]
        CP1[control-plane 1]
        CP2[control-plane 2]
        CP3[control-plane 3]
    end

    C --> V
    V --> H1
    V --> H2
    V --> H3

    H1 --> CP1
    H1 --> CP2
    H1 --> CP3

    H2 --> CP1
    H2 --> CP2
    H2 --> CP3

    H3 --> CP1
    H3 --> CP2
    H3 --> CP3
```

以下に導入方法を示す．

### 2-1. HAProxyの導入

まずはHAProxyをインストールする．
```plaintext
$ sudo apt update
$ sudo apt install -y haproxy
```

設定ファイルは以下の通り．`backend kubernetes-control-planes`には，当初は旧control-planeのみ記載しておく．後に新たに導入する予定のcontrol-planeが無事クラスタに導入できたら旧control-planeの部分は削除し，新たなcontrol-planeに差し替える．
```plaintext
# /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    maxconn 2048
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 10s
    timeout client 1m
    timeout server 1m

frontend kubernetes-api
    bind *:8443
    default_backend kubernetes-control-planes

backend kubernetes-control-planes
    balance roundrobin
    option tcp-check
    default-server inter 3s fall 3 rise 2

    server control-plane0 192.168.1.xx:6443 check # 旧control-plane．後で削除
    server control-plane1 192.168.1.xx:6443 check # 新control-plane1．後で追加
    server control-plane2 192.168.1.xx:6443 check # 新control-plane2．後で追加
    server control-plane3 192.168.1.xx:6443 check # 新control-plane3．後で追加
```

HAProxyを有効化して起動したのち，疎通確認を行う．
```plaintext
$ sudo systemctl restart haproxy
$ sudo systemctl enable haproxy
$ curl -k https://192.168.1.xx:8443/version
```

### 2-2. KeepalivedによるHA構成

HAProxyの手前に仮想IPを提供し，HAProxyの稼働ノードが落ちても仮想IPが引き継がれるようにする．今回は3台のノードを用意し，1台をMASTER，他２台をBACKUPとして動作させる．仮想IPは192.168.1.100とした．

まずはKeepalivedをインストールする．
```plaintext
$ sudo apt update
$ sudo apt install -y keepalived
```

設定ファイル例は以下の通り．
```plaintext
! Configuration File for keepalived

global_defs {
    router_id LVS_HAPROXY_<HOST_NAME>
}

vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight -10
}

vrrp_instance VI_1 {
    state MASTER # BACKUPとして使用する場合は，MASTERからBACKUPに変更
    interface <YOUR_INTERFACE>
    virtual_router_id 51
    priority 100 # priorityは高いものから使用される．BACKUPとして使用する場合90, 80と低めの値に変更
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass <SECRET>
    }

    virtual_ipaddress {
        192.168.1.100/24 dev <YOUR_INTERFACE> label <YOUR_INTERFACE>:vip
    }

    track_script {
        chk_haproxy
    }
}
```

Keepalivedを有効化し起動する．
```plaintext
$ sudo systemctl enable keepalived
$ sudo systemctl restart keepalived
```

念のため，haproxyも再起動する．
```plaintext
$ sudo systemctl restart haproxy
```

この状態で仮想IPにcurlできるか確認する．
```plaintext
$ curl -k https://192.168.1.100:8443/version
```


### 2-3. 証明書の更新とendpointの変更

現証明書には，仮想IPである192.168.1.100からの通信を受け入れるよう書かれていない．そこで証明書を更新することで，旧control-planeのIPアドレスの通信ではなく，仮想IPの通信を受け入れるようSANを変更する．
```plaintext
$ sudo kubeadm init phase certs apiserver --apiserver-cert-extra-sans 192.168.1.100
$ sudo systemctl restart kubelet
```

SANに仮想IPが含まれているかを確認する．
```plaintext
$ openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A 1 "Subject Alternative Name"
            X509v3 Subject Alternative Name:
                DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, IP Address:10.96.0.1, IP Address:192.168.1.100
```

今回は`tar`コマンドを使用して必要な証明書をまとめ，新control-planeに配布した．
```plaintext
$ cd /etc/kubernetes/pki
$ sudo tar czf /tmp/pki-minimal.tar.gz ca.crt ca.key front-proxy-ca.crt front-proxy-ca.key sa.key sa.pub
```

新contro-planeノード上でファイルを受け取り展開する．
```plaintext
$ scp <USER>@<OLD_control-plane_IP>:/tmp/pki-minimal.tar.gz /tmp/
$ sudo mkdir -p /etc/kubernetes/pki
$ sudo tar xzf /tmp/pki-minimal.tar.gz -C /etc/kubernetes/pki
```

kubeadm-configにエンドポイントが192.168.1.100ということを明記する．
```plaintext
$ kubectl -n kube-system get cm kubeadm-config -o yaml
（略）
data:
  ClusterConfiguration: |
	（略）
    # apiServer:
    controlPlaneEndpoint: "192.168.1.100:8443"
```

### 2-4. control-planeをクラスタに追加

既存クラスタに追加するためのトークンを取得する．
```plaintext
$ kubeadm token create --print-join-command
```

ここで出力されたコマンドに，control-planeとして追加するオプションを追加して実行する．
```plaintext
$ sudo kubeadm join 192.168.1.100:8443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH> --control-plane
```

これでcontrol-planeとしてnodeを追加できた．

HAProxyの設定ファイルでは，旧control-planeのIPアドレスしか入れていなかったため，旧control-planeのIPを削除し，新control-planeのIPを追記して，haproxyとkeepalivedに再起動をかける．その後旧control-planeを退役させた．

## 3. 結論

今回は単一のcontrol-plane構成で立ち上げたクラスタを，HAProxyとKeepalivedを用いて複数のcontrol-planeを使用する構成へと移行し，新たに３台のcontrol-planeを追加した．その後，旧構成で使用していたnodeを全て退役させた．

単一のcontrol-plane構成で立ち上げたクラスタを，複数のcontrol-planeを使用する構成に変更することは公式でサポートされていない．新しくcontrol-planeとして追加するノードがjoinする際に仮想IPを参照するkubelet.confを生成することでkubeletの参照先が仮想IPとなり，複数台のcontrol-planeを使用した構成に移行できたが，今後何かしら問題が起きるかもしれない．
また，今回証明書の配布は手動で行った．kubeadmの自動生成に任せていないため，今後`kubeadm cert renew`を行う際に問題が生じる可能性もある．
