# Demo

## Prerequisites

1. Ensure that you have a Kubernetes cluster running version 1.25 or later.
    1.1. You can use Kind/Minikube for this purpose.
2. Have the kubectl CLI tool installed.

# Installation

## Tekton

1. Install Tekton Pipeline 
    ```bash
    kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.54.0/release.yaml
    ```

2. Install Tekton Chains
    ```bash
    kubectl apply -f https://storage.googleapis.com/tekton-releases/chains/previous/v0.19.0/release.yaml
    ```

3. Install Tekton Dashboard
    ```bash
    kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml
    ```

    1. access Dashboard
       ```bash
       kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097
       ```
       http://localhost:9097/

4. Setup Pipelines as Code
    1. Install
    ```bash
    kubectl apply -f https://github.com/openshift-pipelines/pipelines-as-code/releases/download/v0.22.4/release.k8s.yaml
    ```
    2. Port forward the pipelines-as-code controller
       
            a. kubectl port-forward <pipelines-as-code-controller-pod-name> 8080:8080 -n pipelines-as-code
       
            b. Use the gosmee client with the following command

               ```bash
               gosmee client https://hook.pipelinesascode.com/PCoifdgYPYpS http://localhost:8080
               ```
       **OR**
       
       Follow : https://github.com/openshift-pipelines/pipelines-as-code/blob/main/hack/dev/kind/install.sh to create Kind, install Tekton Pipeline and setup gosmee

## ArgoCD
```
kubectl create ns argocd 
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml -n argocd
```

## port forward in order to access
```
kubectl port-forward -n argocd svc/argocd-server 8443:443 > /dev/null 2>&1 &
ADMIN_PASSWD=$(kubectl get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' -n argocd | base64 -d)
argocd login --username admin --password ${ADMIN_PASSWD} localhost:8443 --insecure
IMAGE_UPDATER_TOKEN=$(argocd account generate-token --account image-updater --id image-updater)
kubectl create secret generic argocd-image-updater-secret \
  --from-literal argocd.token=${IMAGE_UPDATER_TOKEN} --dry-run=client -o yaml | kubectl -n argocd apply -f - 
```

## Kyverno Policy Controller

```
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml
```

## Wait for the kyverno controller to be available
```
kubectl -n kyverno wait --for=condition=Available deployment/kyverno-admission-controller && \
kubectl -n kyverno wait --for=condition=Available deployment/kyverno-background-controller && \
kubectl -n kyverno wait --for=condition=Available deployment/kyverno-cleanup-controller && \
kubectl -n kyverno wait --for=condition=Available deployment/kyverno-reports-controller
```

# Instruction to setup and verify CI/CD flow

## CI (Tekton)

1. Apply policies which are required for CI flow

    *. `kubectl create -f policies/require-namespace.yaml`

2. Kyverno policies will kick in and reject request if not satisfied

3. Follow https://pipelinesascode.com/

       1. Create and configure the GithubApp https://pipelinesascode.com/docs/install/github_apps/

       2. Create a repository https://pipelinesascode.com/docs/guide/repositorycrd/

4. Follow Tekton Chains Tutorial https://github.com/tektoncd/chains/blob/main/docs/tutorials/signed-provenance-tutorial.md to set up Chains to sign OCI images built in Tekton

    **Note:** Use the same namespace where Pipelines as Code repo created

5. Send a pull request to https://github.com/savitaashture/kubeday-singapore and observe the triggering of the PipelineRun for the pull request

6. Create the argo application

    ```
    kubectl create -f argo/application.yaml -n argocd
    ```

7. Again Kyverno will verify the pushed image is signed and attested before deploying with Argo as part of CD process.


# Documentation References

Tekton Pipeline doc: https://tekton.dev/docs/

Tekton Chains doc: https://tekton.dev/docs/chains/

Pipelines as Code: https://pipelinesascode.com/

Kyverno: https://kyverno.io/docs

Demo Repository: https://github.com/savitaashture/kubeday-singapore