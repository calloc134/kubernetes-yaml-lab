# 概要

DigitalOcean に k8s を構築して、色々なアプリケーションをデプロイしてみる。

## nginx-ingress-controller と cert-manager の導入

DigitalOcean の MarketPlace に存在しているため、そちらを利用した。

### TLS 有効で Ingress を構築する方法

cert-manager と nginx-ingress-controller を導入すると、TLS 有効な Ingress を構築できる。

1. ClusterIssuer を作成

ClusterIssuer とは、証明書発行のリクエストを行うためのリソース。
cert-manager は、Issuer と ClusterIssuer の 2 つのリソースをサポートしている。
前者は一つの名前空間でのみ利用でき、後者はクラスタ全体で利用できる。

詳細は以下を参考にする。
[https://cert-manager.io/docs/concepts/issuer/](https://cert-manager.io/docs/concepts/issuer/)

letsencrypt を利用して証明書を発行するための ClusterIssuer を作成した。
ClusterIssuer はクラスタに一つ作成すればいいため、ここではデフォルトの名前空間に作成した。

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  # namespace: argocd
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: <your-email>
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

2. Certificate リソースを作成

Certificate リソースは、証明書リクエスト自体を表現するリソース。
詳細は以下を参考にする。
[https://cert-manager.io/docs/usage/certificate/](https://cert-manager.io/docs/usage/certificate/)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-server-tls
  namespace: argocd
spec:
  secretName: argocd-server-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - argocd.calloc.tech
```

3. Ingress リソースを作成

以下のように Ingress リソースを作成する。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    cert-manager.io/clusterissuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - "argocd.calloc.tech"
      secretName: argocd-server-tls
  rules:
    - host: argocd.calloc.tech
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

このようにして、TLS 有効な Ingress を構築できる。

## ArgoCD

DigitalOcean の MarketPlace に存在する ArgoCD はバージョンが古いため、最新の ArgoCD をインストールする。

Helm でインストールした。

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd -f argocd-values.yaml
```

ここで値を指定したのは、ArgoCD の UI に HTTP でアクセスできるようにするため。
後ほど Ingress を構築するときに、SSL/TLS を Ingress で終端させるため。

ArgoCD のパスワードを取得する。

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

とりあえずポートフォワードして、ブラウザでアクセスしてみる。

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

http://localhost:8080 にアクセスできることを確認。

あとは上記の通りの ingress を設定して、SSL/TLS 有効な Ingress を構築する。
