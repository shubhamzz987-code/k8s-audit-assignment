# Kubernetes Audit Assignment

## 1. My App

- **Application name:** Sample Voting Application
- **GitHub repository:** https://github.com/shubhamzz987-code/k8s-audit-assignment
- **Application architecture:** Vote frontend, result frontend, worker, PostgreSQL database, and Redis.
- **Kubernetes namespace:** `vote`
- **Kubernetes manifests:** `examples/sample-app-k8s/`
- **Kind cluster name:** `audit`
- **Cluster configuration:** 1 control-plane node and 2 worker nodes.
- **Ingress hostnames:**
  - `vote.localtest.me`
  - `result.localtest.me`

### Application overview

The application allows users to vote and view the voting results. The vote and result frontends communicate with the backend components, while PostgreSQL stores the voting data and Redis supports the application workflow.

Kubernetes resources are used to manage application deployments, networking, storage, configuration, access permissions, scaling, and scheduled database backups.

## 2. How to Run the Application

### Prerequisites

Install and configure the following tools on the machine where the cluster will run:

- Docker
- kind
- kubectl
- Git

The ingress-nginx controller must also be installed in the cluster.

### Step 1: Clone the repository

```bash
git clone https://github.com/shubhamzz987-code/k8s-audit-assignment.git
cd k8s-audit-assignment
```

### Step 2: Create the kind cluster

```bash
kind create cluster --config kind/kind-config.yaml
```

This creates the configured Kubernetes cluster named `audit`.

### Step 3: Build the application images

```bash
for app in vote result worker; do
  docker build -t "$app:1.0" "sample-app/$app"
done
```

### Step 4: Load images into kind

```bash
for app in vote result worker; do
  kind load docker-image "$app:1.0" --name audit
done
```

This makes the locally built application images available to the cluster nodes.

### Step 5: Install ingress-nginx

Install the ingress-nginx controller using the official kind installation instructions and wait until its controller pod is ready.

### Step 6: Deploy the application

```bash
kubectl apply -f examples/sample-app-k8s/
```

Wait for the application deployments:

```bash
for d in db redis vote result worker; do
  kubectl -n vote rollout status "deploy/$d" --timeout=180s
done
```

### Step 7: Verify the application

Check the resources:

```bash
kubectl -n vote get deployments
kubectl -n vote get pods
kubectl -n vote get services
kubectl -n vote get ingress
```

Test the vote endpoint:

```bash
curl -s -o /dev/null -w 'Vote HTTP status: %{http_code}\n' \
  -H 'Host: vote.localtest.me' http://127.0.0.1/
```

Test the result endpoint:

```bash
curl -s -o /dev/null -w 'Result HTTP status: %{http_code}\n' \
  -H 'Host: result.localtest.me' http://127.0.0.1/
```

Both endpoints returned HTTP 200 during my testing.

**Note:** The commands above assume the cluster and ingress controller have been configured successfully. On a fresh machine, complete the ingress installation before testing the hostnames.

## 3. Kubernetes Audit Results

**Audit command:**

```bash
./audit.sh
```

**Result: 16/16 checks detected**

- Must-have checks: 12/12
- Stretch checks: 4/4
- Total: 16/16

| # | Audit check | Status | Evidence / verification command | Why it is used |
|---:|---|---|---|---|
| 1 | Deployment and ReplicaSet | PASS | `kubectl -n vote get deployments,replicasets` | Manages application replicas and creates replacement pods when required. |
| 2 | Services | PASS | `kubectl -n vote get services` | Provides stable network endpoints for application components. |
| 3 | Namespace | PASS | `kubectl get namespaces` | Isolates application resources in the `vote` namespace. |
| 4 | Labels and selectors | PASS | `kubectl -n vote get services,endpoints` | Connects Services to matching pods through labels and selectors. |
| 5 | Rolling update and rollback | PASS | `kubectl -n vote rollout history deployment/vote` | Demonstrates Deployment revisions and rollback capability. |
| 6 | ConfigMap | PASS | `kubectl -n vote get configmap vote-config -o yaml` | Stores non-sensitive application configuration separately from container images. |
| 7 | Secret | PASS | `kubectl -n vote get secret db-credentials` | Stores database credentials separately from ordinary configuration. |
| 8 | Resource requests and limits | PASS | `kubectl -n vote get deployments -o yaml` | Specifies container CPU and memory requirements and limits. |
| 9 | Liveness and readiness probes | PASS | `kubectl -n vote get deployments -o yaml` | Helps Kubernetes detect unhealthy containers and determine when pods are ready for traffic. |
| 10 | PersistentVolumeClaims | PASS | `kubectl -n vote get pvc` | Requests persistent storage for database data and backup files. |
| 11 | Ingress | PASS | `kubectl -n vote get ingress` | Routes HTTP traffic to the vote and result Services using hostnames. |
| 12 | Multi-node kind cluster | PASS | `kubectl get nodes -o wide` | Demonstrates a cluster with one control-plane node and two worker nodes. |
| 13 | HPA — stretch | PASS | `kubectl -n vote get hpa` | Configures horizontal scaling of the vote Deployment based on CPU utilization. |
| 14 | RBAC and ServiceAccounts — stretch | PASS | `kubectl -n vote get roles,rolebindings,serviceaccounts` | Demonstrates scoped permissions and Kubernetes identities. |
| 15 | CronJob — stretch | PASS | `kubectl -n vote get cronjob,jobs,pvc` | Schedules database backups and stores the resulting SQL dump on persistent storage. |
| 16 | GitHub Actions to kind — stretch | PASS — workflow detected | `.github/workflows/kind-deploy.yml` | Defines a workflow to create a kind cluster, build and load images, deploy the application, and run a smoke test. |

## 4. Rolling Update and Rollback Evidence

The audit detected multiple image revisions for the vote Deployment.

The rollback demonstration used a deliberately invalid image tag to create a failed rollout, then restored the previous Deployment revision.

Commands used to inspect the result:

```bash
kubectl -n vote rollout history deployment/vote
kubectl -n vote get replicasets
kubectl -n vote get deployment vote \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="vote")].image}{"\n"}'
kubectl -n vote get pods -l app=vote
```

The intended final state is that the vote Deployment uses `vote:1.0` and its two replicas are ready.

The rollback demonstrates how Kubernetes can restore a previous Deployment revision after an unsuccessful rollout.

## 5. Database Backup Evidence

A CronJob named `db-backup` was configured to run a PostgreSQL backup using `pg_dump`.

The backup was tested by creating a manual Job from the CronJob:

```bash
kubectl -n vote create job db-backup-manual --from=cronjob/db-backup
```

Inspect the Job and its logs:

```bash
kubectl -n vote get jobs
kubectl -n vote logs job/db-backup-manual
```

The manual backup Job completed successfully and generated a timestamped SQL dump in the backup directory on the `db-backups` PVC.

The SQL dump header was inspected to verify that a PostgreSQL database dump had been created.

## 6. GitHub Actions Evidence

**Workflow file:** `.github/workflows/kind-deploy.yml`

The workflow is configured to run on pull requests and through manual execution using `workflow_dispatch`.

It contains steps to:

1. Check out the repository.
2. Create a three-node kind cluster using `helm/kind-action`.
3. Build the vote, result, and worker Docker images.
4. Load the images into kind.
5. Apply the Kubernetes manifests.
6. Wait for application deployments.
7. Run a smoke test against the vote application.

The local audit script detected this workflow and awarded the GitHub Actions stretch check.

**Important:** The local audit confirms that the workflow file was detected. It does not establish that a GitHub-hosted workflow run succeeded. A successful run must be verified in the repository's GitHub Actions tab.

## 7. Audit Score Summary

| Category | Score |
|---|---:|
| Must-have checks | 12/12 |
| Stretch checks | 4/4 |
| **Total checks** | **16/16** |

The audit script reported that the assignment pass mark is 10 or more.

## 8. What Was Hard and What I Would Improve

### Challenges

- Setting up a multi-node kind cluster and resolving Docker permission issues.
- Configuring ingress routing and verifying application access through HTTP requests.
- Understanding Deployment revisions and demonstrating rollback.
- Configuring persistent storage for PostgreSQL and database backups.
- Setting up Kubernetes resource requests, limits, probes, HPA, and RBAC.
- Verifying that the backup Job generated a valid PostgreSQL SQL dump.

### Improvements

- Run the GitHub Actions workflow and verify its successful execution.
- Add a documented database restore procedure and test recovery from a backup.
- Improve secret management for production deployments.
- Add monitoring and alerts for failed backups and unhealthy workloads.
- Configure backup retention and test disaster recovery.
- Use a production-grade Kubernetes environment for workloads that require high availability and stronger security.

## 9. Conclusion

This assignment demonstrates practical Kubernetes skills through a sample voting application deployed to a three-node kind cluster.

The implementation covers Deployments, ReplicaSets, Services, Namespaces, labels and selectors, rolling updates and rollback, ConfigMaps, Secrets, resource management, health probes, persistent storage, Ingress, HPA, RBAC, ServiceAccounts, CronJobs, and a GitHub Actions workflow.

The local audit reported **16/16 checks detected**, including all must-have and stretch checks.
