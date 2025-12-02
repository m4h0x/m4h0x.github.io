---
title: "Docker Registryをデプロイした際の備忘録"
date: 2025-10-12T21:54:20+09:00
draft: false
summary: |
  Kubetnetesクラスタ内にDocker Registryをデプロイした際の備忘録である．
categories:
  - Kubernetes
tags:
  - DockerRegistry
---

## 1. 背景

今まで，ghcr.ioにDockerイメージを置き，KubernetesクラスタでもGHCRからイメージをpullして利用していた．しかし，GHCRのレートリミットやクラスタ内でイメージをpullすることが多々あることから，クラスタ内にホスティングして快適に使用できるようしたいと思うようになった．今回はDocker Registryを構築した際の備忘録を残しておく．

## 2. 本題

### 2-1. Registry認証ユーザー作成

Registryにログインするための`htpasswd`を生成し，Kubernetes Secretとして登録  
```plaintext
$ htpasswd -nbB <USER> <PASSWORD> > htpasswd 
$ kubectl create ns docker-registry 
$ kubectl -n docker-registry create secret generic registry-auth \   --from-file=htpasswd=./htpasswd
```

### 2-2. ConfigMapの作成

Basic認証と永続化ストレージを有効化するための`config.yml`を作成
```plaintext
$ cat config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
auth:
  htpasswd:
    realm: basic-realm
    path: /auth/htpasswd
```

ConfigMapとして登録
```plaintext
$ kubectl -n docker-registry create configmap registry-config --from-file=config.yml
```

### 2-3. PersistentVolumeClaim 作成

```plaintext
$ cat pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-data
  namespace: docker-registry
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: <YOUR_STORAGE_CLASS>
```

### 2-4. Deployment 作成

今回は単一ノードでRegistryをデプロイする構成のため，nodeSelectorでノードを指定している．
```plaintext
$ cat deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: docker-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
        - name: registry
          image: registry:2
          ports:
            - containerPort: 5000
          volumeMounts:
            - name: data
              mountPath: /var/lib/registry
            - name: config
              mountPath: /etc/docker/registry
            - name: auth
              mountPath: /auth
          env:
            - name: REGISTRY_CONFIGURATION_PATH
              value: /etc/docker/registry/config.yml
          readinessProbe:
            httpGet: { path: /, port: 5000 }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /, port: 5000 }
            initialDelaySeconds: 10
            periodSeconds: 20
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: registry-data
        - name: config
          configMap:
            name: registry-config
        - name: auth
          secret:
            secretName: registry-auth
      nodeSelector:
        kubernetes.io/hostname: <YOUR_HOSTNAME>
```

### 2-5. Service 作成

```plaintext
$ cat service.yml
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: docker-registry
spec:
  selector:
    app: registry
  ports:
    - port: 5000
      targetPort: 5000
      name: http
  type: ClusterIP
```

## 3. 使用例

**Registryにログイン**
```plaintext
$ docker login https://registry.example.com
# Username: <USER>
# Password: <PASSWORD>
```

**イメージにFQDNをつけてタグ付け**
```plaintext
$ docker tag <LOCAL_IMAGE>:<TAG> registry.example.com/<IMAGE>:<TAG>
```

**イメージのpush**
```plaintext
$ docker push registry.example.com/<IMAGE>:<TAG>
```

**イメージのpull**
例1. 別マシン・別環境で使用するとき
```plaintext
$ docker pull registry.example.com/<REPO>/<IMAGE>:<TAG>
```

例2. Kubernetesから参照するとき（DeploymentやPodのmanifest）
```plaintext
image: registry.example.com/<IMAGE>:<TAG>
```

**Registryに保存されている全イメージとタグを確認**
```plaintext
$ cat image.sh
curl -s -u <USER>:<PASSWORD> https://registry.example.com/v2/_catalog \
| jq -r '.repositories[]' \
| while read repo; do
    echo "Repository: $repo"
    curl -s -u <USER>:<PASSWORD> \
        https://registry.example.com/v2/$repo/tags/list \
    | jq -r '.tags[]?' | sed 's/^/  Tag: /'
  done
```

## 4. 結論

自宅Kubernetesクラスタ上に Docker Registryをデプロイした．単一のノード上で動作する前提となっていることや，大きいイメージをpushするとタイムアウトになりpushできないといった問題がまだ残っているため，今後修正していきたい．
