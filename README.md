# 概要

DigitalOcean に k8s を構築して、色々なアプリケーションをデプロイしてみる。

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

## nginx-ingress-controller と cert-manager の導入

DigitalOcean の MarketPlace に存在しているため、そちらを利用した。

### Ingress の設定

ArgoCD の UI にアクセスするために、Ingress を構築する。

```bash
kubectl apply -f argocd-ingress.yaml
```

ここでは、自分の所有しているドメインを指定した。
TODO: kustomize でドメインを指定できるようにする方法を調べる

これでドメインからアクセスできるようになる。
