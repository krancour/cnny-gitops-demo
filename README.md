# Cloud Native New York -- GitOps Demo

## Prerequisites

* [Docker](https://www.docker.com/)
* [kind](https://kind.sigs.k8s.io/) or [k3d](https://k3d.io/)
* [Helm](https://helm.sh/docs/)

## Create a Local Cluster (kind)

Paste the following into your terminal:

```shell
kind create cluster \
  --wait 120s \
  --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cnny-demo
nodes:
- extraPortMappings:
  - containerPort: 30080
    hostPort: 8080
  - containerPort: 30081
    hostPort: 8081
  - containerPort: 30082
    hostPort: 8082
  - containerPort: 30083
    hostPort: 8083
EOF
```

## Create a Local Cluster (k3d)

Paste the following into your terminal:

```shell
k3d cluster create cnny-demo \
  --no-lb \
  --k3s-arg '--disable=traefik@server:0' \
  -p '8080-8083:30080-30083@servers:0:direct' \
  --wait
```

## Install Argo CD

Paste the following into your terminal:

```shell
helm install argocd argo-cd \
  --repo https://argoproj.github.io/argo-helm \
  --version 5.21.0 \
  --namespace argocd \
  --create-namespace \
  --set 'configs.secret.argocdServerAdminPassword=$2a$10$5vm8wXaSdbuff0m9l21JdevzXBzJFPCi8sy6OOnpZMAG.fOXL7jvO' \
  --set dex.enabled=false \
  --set notifications.enabled=false \
  --set server.extensions.enabled=true \
  --set server.service.type=NodePort \
  --set server.service.nodePortHttp=30080 \
  --wait
```

## Try Argo CD

1. Paste the following into your terminal:

   ```shell
   cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Namespace
   metadata:
     name: argocd-demo
   ---
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: argocd-demo
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/krancour/cnny-gitops-demo.git
       targetRevision: main
       path: demos/argocd
     destination:
       server: https://kubernetes.default.svc
       namespace: argocd-demo
   EOF
   ```

1. Visit [localhost:8080](http://localhost:8080) and log in.

   Username and password are both `admin`.

1. Explore the UI and use it to _sync_ the `argocd-demo` `Application`.

1. Visit [localhost:8081](http://localhost:8081) to see the running application.

1. Paste the following into your terminal:

   ```shell
   cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Namespace
   metadata:
     name: helm-demo
   ---
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: helm-demo
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://charts.bitnami.com/bitnami
       chart: nginx
       targetRevision: 14.1.0
       helm:
         parameters:
         - name: service.type
           value: NodePort
         - name: service.nodePorts.http
           value: "30082"
     destination:
       server: https://kubernetes.default.svc
       namespace: helm-demo
   EOF
   ```

1. Visit [localhost:8080](http://localhost:8080) and use the UI to _sync_ the
   `helm-demo` `Application`.

1. Visit [localhost:8082](http://localhost:8082) to see the running application.

1. Paste each of the following commands into your terminal to clean up:

   ```shell
   kubectl delete application argocd-demo --namespace argocd

   kubectl delete application helm-demo --namespace argocd

   kubectl delete namespace argocd-demo

   kubectl delete namespace helm-demo
   ```

## Install cert-manager

The basic Kargo installation uses cert-manager to generate self-signed
certificates. This is more straightforward than providing your own certificates,
so we will install cert-manager as a prerequisite.

Paste the following into your terminal:

```shell
helm install cert-manager cert-manager \
  --repo https://charts.jetstack.io \
  --version 1.11.0 \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --wait
```

## Install Kargo

```shell
helm install kargo \
  oci://ghcr.io/akuity/kargo-charts/kargo \
  --version 0.1.0-rc.8 \
  --namespace kargo \
  --create-namespace
```

## Try Kargo

1. Fork this repo, then create a personal access token with permission to read
   and write to your fork.

1. Edit the following and then paste it into your terminal:

   ```shell
   export GITOPS_REPO_URL=<your repo URL, starting with https://>
   export GITHUB_USERNAME=<your github handle>
   export GITHUB_PAT=<your personal access token>
   ```

1. Create three Argo CD `Application`s representing three different
   environments:

   ```shell
   cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Namespace
   metadata:
     name: kargo-demo-test
   ---
   apiVersion: v1
   kind: Namespace
   metadata:
     name: kargo-demo-stage
   ---
   apiVersion: v1
   kind: Namespace
   metadata:
     name: kargo-demo-prod
   ---
   apiVersion: v1
   kind: Secret
   type: Opaque
   metadata:
     name: kargo-demo-repo
     namespace: argocd
     labels:
       argocd.argoproj.io/secret-type: repository
   stringData:
     type: git
     project: default
     url: ${GITOPS_REPO_URL}
     username: ${GITHUB_USERNAME}
     password: ${GITHUB_PAT}
   ---
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: kargo-demo-test
     namespace: argocd
     annotations:
       kargo.akuity.io/authorized-env: kargo-demo:test
   spec:
     project: default
     source:
       repoURL: ${GITOPS_REPO_URL}
       targetRevision: main
       path: demos/kargo/env/test
     destination:
       server: https://kubernetes.default.svc
       namespace: kargo-demo-test
   ---
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: kargo-demo-stage
     namespace: argocd
     annotations:
       kargo.akuity.io/authorized-env: kargo-demo:stage
   spec:
     project: default
     source:
       repoURL: ${GITOPS_REPO_URL}
       targetRevision: main
       path: demos/kargo/env/stage
     destination:
       server: https://kubernetes.default.svc
       namespace: kargo-demo-stage
   ---
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: kargo-demo-prod
     namespace: argocd
     annotations:
       kargo.akuity.io/authorized-env: kargo-demo:prod
   spec:
     project: default
     source:
       repoURL: ${GITOPS_REPO_URL}
       targetRevision: main
       path: demos/kargo/env/prod
     destination:
       server: https://kubernetes.default.svc
       namespace: kargo-demo-prod
   EOF
   ```

1. Create Kargo `PromotionPolicy` and `Environment` resources:

   ```shell
   cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Namespace
   metadata:
     name: kargo-demo
   ---
   apiVersion: kargo.akuity.io/v1alpha1
   kind: PromotionPolicy
   metadata:
     name: test
     namespace: kargo-demo
   environment: test
   authorizedPromoters:
   - subjectType: Group
     name: system:masters
   enableAutoPromotion: true
   ---
   apiVersion: kargo.akuity.io/v1alpha1
   kind: Environment
   metadata:
     name: test
     namespace: kargo-demo
   spec:
     subscriptions:
       repos:
         images:
         - repoURL: nginx
           semverConstraint: ^1.24.0
     promotionMechanisms:
       gitRepoUpdates:
       - repoURL: ${GITOPS_REPO_URL}
         writeBranch: main
         kustomize:
           images:
           - image: nginx
             path: demos/kargo/env/test
       argoCDAppUpdates:
       - appName: kargo-demo-test
         appNamespace: argocd
     healthChecks:
       argoCDAppChecks:
       - appName: kargo-demo-test
         appNamespace: argocd
   ---
   apiVersion: kargo.akuity.io/v1alpha1
   kind: PromotionPolicy
   metadata:
     name: stage
     namespace: kargo-demo
   environment: stage
   authorizedPromoters:
   - subjectType: Group
     name: system:masters
   enableAutoPromotion: true
   ---
   apiVersion: kargo.akuity.io/v1alpha1
   kind: Environment
   metadata:
     name: stage
     namespace: kargo-demo
   spec:
     subscriptions:
       upstreamEnvs:
       - name: test
     promotionMechanisms:
       gitRepoUpdates:
       - repoURL: ${GITOPS_REPO_URL}
         writeBranch: main
         kustomize:
           images:
           - image: nginx
             path: demos/kargo/env/stage
       argoCDAppUpdates:
       - appName: kargo-demo-stage
         appNamespace: argocd
     healthChecks:
       argoCDAppChecks:
       - appName: kargo-demo-stage
         appNamespace: argocd
   ---
   apiVersion: kargo.akuity.io/v1alpha1
   kind: PromotionPolicy
   metadata:
     name: prod
     namespace: kargo-demo
   environment: prod
   authorizedPromoters:
   - subjectType: Group
     name: system:masters
   enableAutoPromotion: true
   ---
   apiVersion: kargo.akuity.io/v1alpha1
   kind: Environment
   metadata:
     name: prod
     namespace: kargo-demo
   spec:
     subscriptions:
       upstreamEnvs:
       - name: stage
     promotionMechanisms:
       gitRepoUpdates:
       - repoURL: ${GITOPS_REPO_URL}
         writeBranch: main
         kustomize:
           images:
           - image: nginx
             path: demos/kargo/env/prod
       argoCDAppUpdates:
       - appName: kargo-demo-prod
         appNamespace: argocd
     healthChecks:
       argoCDAppChecks:
       - appName: kargo-demo-prod
         appNamespace: argocd
   EOF
   ```

1. Visit [localhost:8080](http://localhost:8080) and use the UI to observe
   the latest manifests from your fork and the latest Nginx image progressing
   through our three environments.

> 🔍&nbsp;&nbsp;Tip: Refreshing any `Application` in the UI will trigger
> reconciliation of any related `Environment` resources. This is a convenient
> way of moving the changes through the environments more rapidly instead of
> always waiting for things to happen at their usual intervals.

1. It is also informative to look at the status of `Promotion` and `Environment`
   resources:

   ```shell
   kubectl get promotions --namespace kargo-demo
   
   kubectl get environments --namespace kargo-demo
   ```

1. The application running in each of the three environments should be visible
   at:

    * test: [localhost:8081](http://localhost:8081)
    * stage: [localhost:8082](http://localhost:8082)
    * prod: [localhost:8083](http://localhost:8083)

1. Skip this cleanup and move on to the next section if you're ready to tear
   down your cluster, otherwise:

   ```shell
   kubectl delete namespace kargo-demo

   kubectl delete application kargo-demo-test --namespace argocd
   kubectl delete application kargo-demo-stage --namespace argocd
   kubectl delete application kargo-demo-prod --namespace argocd
   ```

## Clean Up the Cluster (kind)

Paste the following into your terminal:

```
kind delete cluster --name cnny-demo
```

## Clean Up the Cluster (kind)

Paste the following into your terminal:

```
k3d cluster delete cnny-demo
```
