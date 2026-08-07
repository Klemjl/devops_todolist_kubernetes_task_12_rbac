# INSTRUCTION.md

## Overview
This document describes how to validate the RBAC setup implemented in this repository. It covers spinning up a local `kind` cluster, deploying the application, and verifying that the `todoapp` pod can list Kubernetes secrets using its dedicated `ServiceAccount`.

## Prerequisites
- [Docker](https://docs.docker.com/get-docker/) installed and running
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) installed

## 1. Clone the repository
```bash
git clone <your-fork-url>
cd devops_todolist_kubernetes_task_12_rbac
```

## 2. Create the cluster and deploy all resources
Run the bootstrap script, which creates a `kind` cluster from `cluster.yml` and applies all manifests (MySQL, application, RBAC, ingress):
```bash
bash bootstrap.sh
```

If a cluster named `todoapp` already exists from a previous run, delete it first:
```bash
kind delete cluster --name todoapp
```

## 3. Verify the cluster and namespaces
```bash
kubectl get nodes
kubectl get pods -n todoapp
kubectl get pods -n mysql
```
Wait until all pods are in `Running` state (this may take a minute, especially on first run while images are pulled and MySQL initializes).

## 4. Verify the RBAC resources
```bash
kubectl get serviceaccount -n todoapp
kubectl get role -n todoapp
kubectl get rolebinding -n todoapp
```
Confirm that:
- A `ServiceAccount` named `secrets-reader` exists
- A `Role` exists with permission to `list` `secrets`
- A `RoleBinding` binds the `Role` to `secrets-reader`

## 5. Verify the Deployment uses the ServiceAccount
```bash
kubectl get deployment todoapp -n todoapp -o jsonpath='{.spec.template.spec.serviceAccountName}'
```
Expected output: `secrets-reader`

## 6. Validate access to secrets from inside the pod
Get the pod name:
```bash
kubectl get pods -n todoapp
```

Exec into the pod:
```bash
kubectl exec -it <pod-name> -n todoapp -- sh
```

Inside the pod, run:
```bash
curl -sS -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  https://kubernetes.default.svc/api/v1/namespaces/todoapp/secrets
```

Expected result: a JSON response of `"kind": "SecretList"` containing the secrets in the `todoapp` namespace. This confirms that the RBAC configuration correctly grants the pod's `ServiceAccount` permission to list secrets.

A screenshot of this command and its output is attached to the pull request as evidence.

## 7. Clean up (optional)
```bash
kind delete cluster --name todoapp
```