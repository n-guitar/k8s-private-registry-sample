# k8s-private-registry-sample

1.  docker registry 設定用の id,password を htpasswd で作成
2.  作成した htpasswd を secret として作成
3.  自己証明書の作成
4.  作成した 自己証明書 を secret として作成
5.  private registry の作成
6.  service の作成
7.  別 Pod から接続確認

## create password

## 作成

```bash
# docker-registryに設定するhtpasswordを作成
$ mkdir auth
$ docker run --entrypoint htpasswd registry:2.7.0 -Bbn testuser testpassword > auth/htpasswd
```

## secret 登録

### Referral link

-   https://kubernetes.io/ja/docs/concepts/configuration/secret/

```bash
# 特定のnamespaceにファイルからシークレットを登録する
$ kubectl create secret generic private-registry-htpasswd --from-file=htpasswd=auth/htpasswd -n docker-registry

# 確認
$ kubectl get secret private-registry-htpasswd -n docker-registry
NAME                        TYPE     DATA   AGE
private-registry-htpasswd   Opaque   1      36s
```

## self-signed-certificate

### Referral link

-   https://github.com/n-guitar/self-signed-certificate

### 作成

```bash
# サーバー鍵を作成 パスフレーズなし
$ openssl genrsa -out server.key.encrypted 4096

# 暗号化されない鍵ファイル作成
$ openssl rsa -in server.key.encrypted -out server.key

# 証明書署名要求(CSR)を作成
$ openssl req -new -key server.key -out server.csr -subj "/C=JP/ST=Tokyo/L=Okawari/O=Haku-mai/CN=k8s-private-registry-svc.docker-registry.svc.cluster.local"

# SANエントリの作成
$ echo "subjectAltName = DNS:k8s-private-registry-svc.docker-registry.svc.cluster.local" > san.txt

# 自己署名証明書作成
$ openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt -extfile san.txt
```

### 確認

```bash
# 証明書の確認
$ openssl x509 -text -noout -in server.crt

# 秘密鍵の確認
$ openssl rsa -text -noout -in server.key

# 証明書署名要求(CSR)の確認
$ openssl req -text -noout -in server.csr
```

## 証明書を secret 登録

```bash
$ kubectl create secret generic docker-registry-tls-cert --from-file=server.crt=server.crt --from-file=server.key=server.key -n docker-registry
```

## docker registry 用 data volume 作成 (option)

-   local volume のサンプル
-   [private-registry-data-pv.yaml](private-registry-data-pv.yaml)

```bash
$ kubectl apply -f private-registry-data-pv.yaml
```

## docker registry 用 pvc 作成 (option)

-   サンプル
-   [private-registry-data-pvc.yaml](private-registry-data-pvc.yaml)

## docker registry 作成

```bash
$ kubectl apply -f k8s-private-registry.yaml
```

## docker registry cluster ip service 作成

```bash
$ kubectl apply -f k8s-private-registry-svc.yaml
```

### template yaml (option)

```bash
$ kubectl create deployment k8s-private-registry --image=registry:2.7.0 -n docker-registry --dry-run=client -o yaml > k8s-private-registry.yaml
```

# create secret docker-registry (option)

```bash
$ kubectl create secret docker-registry k8s-private-registry --server=k8s-private-registry-svc.docker-registry.svc.cluster.local --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD
```

# 同じ namespace の pod から確認

-   alpine
    -   注意!! privileged: true
        -   現在非特権の Container から build はできない
        -   https://github.com/genuinetools/img/issues/228

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: test
    namespace: docker-registry
spec:
    replicas: 1
    selector:
        matchLabels:
            app: deployment-docker-registry-test
    template:
        metadata:
            labels:
                app: deployment-docker-registry-test
        spec:
            containers:
                - image: alpine:3.14
                  name: test
                  securityContext:
                      privileged: true
                  volumeMounts:
                      - mountPath: /usr/local/share/ca-certificates/
                        name: crt
            volumes:
                - name: crt
                  secret:
                      defaultMode: 256
                      optional: false
                      secretName: docker-registry-tls-cert
```

## Container 内

```bash
# 証明書update
update-ca-certificates

# nslookup
nslookup k8s-private-registry-svc.docker-registry.svc.cluster.local

# module 追加
apk add git img curl jq

# エンドポイント確認 > HTTP/2 200
curl -I https://k8s-private-registry-svc.docker-registry.svc.cluster.local


# regirtry login
img login k8s-private-registry-svc.docker-registry.svc.cluster.local

# regirtry 登録
# git先はsample
cd /tmp
git clone https://github.com/n-guitar/fiber-file-uploader.git; cd fiber-file-uploader

# image build
img build -t k8s-private-registry-svc.docker-registry.svc.cluster.local/haku-mai/fiber-f
ile-uploader:1.00 .

# 確認
img ls
NAME                                                                                            SIZE            CREATED AT      UPDATED AT      DIGEST
k8s-private-registry-svc.docker-registry.svc.cluster.local/haku-mai/fiber-file-uploader:1.00    15.68MiB        2 minutes ago   2 minutes ago   sha256:5456f40bef6a6f0fd1e2e22197cfb56c9e9507c8794d930a11ae94f1e60d4f30

# image
img push k8s-private-registry-svc.docker-registry.svc.cluster.local/haku-mai/fiber-file-uploader:1.00
Pushing k8s-private-registry-svc.docker-registry.svc.cluster.local/haku-mai/fiber-file-uploader:1.00...
Successfully pushed k8s-private-registry-svc.docker-registry.svc.cluster.local/haku-mai/fiber-file-uploader:1.00

# 確認 image一覧
curl --basic -u $username:$password https://k8s-private-registry-svc.docker-registry.svc.cluster.local/v2/_catalog
{"repositories":["haku-mai/fiber-file-uploader"]}

# 確認 tag一覧
curl --basic -u $username:$password https://k8s-private-registry-svc.docker-registry.svc.cluster.local/v2/haku-mai/fiber-file-
uploader/tags/list
{"name":"haku-mai/fiber-file-uploader","tags":["1.00"]}
```

## エラー

-   san エントリの作成が必要

```bash
Error: creating registry client failed: Get "https://k8s-private-registry-svc.docker-registry.svc.cluster.local/v2/": x509: certificate relies on legacy Common Name field, use SANs or temporarily enable Common Name matching with GODEBUG=x509ignoreCN=0
```
