# Set 01: Configure Jenkins to Connect to a Kubernetes Cluster and Verify Connection

This guide walks you through connecting a Jenkins controller (running either inside or outside a Kubernetes cluster) to a Kubernetes cluster using the **Jenkins Kubernetes Plugin**, and verifying the integration.

---

## Architecture Overview

```
+-------------------------------------------------------------+
| Jenkins Controller                                          |
| (Manages jobs, pipelines, cloud configuration)              |
+------------------------------+------------------------------+
                               |
                               | 1. API Calls (HTTPS:6443)
                               |    via ServiceAccount Token
                               v
+-------------------------------------------------------------+
| Kubernetes Cluster                                          |
|                                                             |
|   +-----------------------------------------------------+   |
|   | Pod: jenkins-agent-xxxx                             |   |
|   |   +-------------------+     +---------------------+ |   |
|   |   | jnlp container    | <-> | user container(s)   | |   |
|   |   | (talks to Jenkins |     | (runs build steps)  | |   |
|   |   |  tunnel port)     |     |                     | |   |
|   |   +-------------------+     +---------------------+ |   |
|   +-----------------------------------------------------+   |
+-------------------------------------------------------------+
```

---

## Prerequisites

1. **Kubernetes Cluster**: Accessible via `kubectl` with administrative privileges.
2. **Jenkins Controller**: Running (e.g., LTS version) with the **Kubernetes Plugin** installed:
   - Navigate to **Manage Jenkins** > **Plugins** > **Available plugins**.
   - Search for **Kubernetes** (`kubernetes`) and install it without restart if possible (or restart Jenkins).

---

## Step 1: Create Kubernetes RBAC & ServiceAccount Token

Jenkins needs permissions to create, monitor, and delete agent pods and related resources in the cluster.

Apply the provided [`jenkins-rbac.yaml`](./jenkins-rbac.yaml):

```bash
kubectl apply -f jenkins-rbac.yaml
```

### Manifest Breakdown (`jenkins-rbac.yaml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-admin
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/exec", "pods/log", "persistentvolumeclaims", "events"]
    verbs: ["*"]
  - apiGroups: ["apps"]
    resources: ["deployments", "daemonsets", "statefulsets"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin-binding
subjects:
  - kind: ServiceAccount
    name: jenkins-admin
    namespace: jenkins
roleRef:
  kind: ClusterRole
  name: jenkins-admin
  apiGroup: rbac.authorization.k8s.io
---
# For Kubernetes 1.24+: long-lived token secret
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-admin-token
  namespace: jenkins
  annotations:
    kubernetes.io/service-account.name: jenkins-admin
type: kubernetes.io/service-account-token
```

---

## Step 2: Extract Cluster Information

Run the following commands to retrieve the values needed for Jenkins:

### 1. Kubernetes API Server URL
```bash
kubectl cluster-info | grep "control plane"
# Or:
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```
*(Example: `https://192.168.49.2:8443` or `https://k8s-api.example.com:6443`)*

### 2. Kubernetes Server CA Certificate
```bash
# Linux / macOS (Base64 decoded certificate):
kubectl get secret jenkins-admin-token -n jenkins -o jsonpath='{.data.ca\.crt}' | base64 --decode

# Windows PowerShell:
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl get secret jenkins-admin-token -n jenkins -o jsonpath='{.data.ca\.crt}')))
```

### 3. ServiceAccount Bearer Token
```bash
# Linux / macOS:
kubectl get secret jenkins-admin-token -n jenkins -o jsonpath='{.data.token}' | base64 --decode

# Windows PowerShell:
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl get secret jenkins-admin-token -n jenkins -o jsonpath='{.data.token}')))
```

---

## Step 3: Add Credentials in Jenkins

1. Go to **Manage Jenkins** > **Credentials** > **System** > **Global credentials (unrestricted)**.
2. Click **Add Credentials**:
   - **Kind**: `Secret text`
   - **Scope**: `Global (Jenkins, nodes, items, all child items, etc)`
   - **Secret**: Paste the base64-decoded token retrieved in Step 2.
   - **ID**: `k8s-jenkins-token`
   - **Description**: `Kubernetes ServiceAccount Token for Jenkins`
3. Click **Create**.

---

## Step 4: Configure the Kubernetes Cloud in Jenkins

1. Navigate to **Manage Jenkins** > **Clouds** (in older versions: **Manage Jenkins** > **Nodes and Clouds** > **Configure Clouds**).
2. Click **New cloud** (or **Add a new cloud**) and select **Kubernetes**.
3. Fill in the configuration details:

| Field | Scenario A: Jenkins inside K8s | Scenario B: Jenkins outside K8s |
|---|---|---|
| **Name** | `kubernetes` | `kubernetes` |
| **Kubernetes URL** | `https://kubernetes.default.svc` *(or leave blank)* | `https://<K8S_API_SERVER_IP>:6443` |
| **Kubernetes Certificate key** | *(Leave blank to trust cluster CA)* | Paste the decoded `ca.crt` (or check *Disable https certificate check* for lab tests) |
| **Kubernetes Namespace** | `jenkins` | `jenkins` |
| **Credentials** | Select `k8s-jenkins-token` | Select `k8s-jenkins-token` |
| **Jenkins URL** | `http://jenkins.jenkins.svc.cluster.local:8080` | `http://<JENKINS_HOST_IP>:8080` *(accessible from agent pods)* |
| **Jenkins tunnel** | `jenkins-agent.jenkins.svc.cluster.local:50000` | `<JENKINS_HOST_IP>:50000` |

4. Click **Test Connection**.
   - If successful, you will see: **`Connected to Kubernetes v1.x.x`** in blue text.
5. Click **Save**.

---

## Step 5: Verify End-to-End with a Pipeline

To verify that Jenkins cannot only authenticate but can also dynamically provision agent pods, execute jobs, and tear them down:

1. Create a new Pipeline job:
   - In Jenkins, click **New Item**.
   - Enter name: `k8s-connection-test`.
   - Select **Pipeline** and click **OK**.
2. Scroll to the **Pipeline** section and choose **Pipeline script**.
3. Paste the pipeline script from [`Jenkinsfile`](./Jenkinsfile):

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: jenkins-agent-test
spec:
  containers:
  - name: test-runner
    image: alpine:latest
    command:
    - cat
    tty: true
'''
        }
    }
    stages {
        stage('Verify Kubernetes Dynamic Agent') {
            steps {
                container('test-runner') {
                    sh '''
                        echo "=== Verifying Kubernetes Agent Connection ==="
                        uname -a
                        echo "Hostname: $(hostname)"
                        echo "Date: $(date)"
                        echo "Connection to Kubernetes cluster successfully established and verified!"
                    '''
                }
            }
        }
    }
}
```

4. Click **Save** and then click **Build Now**.
5. Observe the behavior:
   - On Kubernetes: Run `kubectl get pods -n jenkins -w` to see the agent pod spin up.
   - In Jenkins: Check **Build Executor Status** on the left menu to see the dynamic agent register.
   - In Console Output: Verify build succeeds with exit code 0.
   - Once completed, Kubernetes automatically deletes the agent pod.

---

## Troubleshooting Checklist

| Symptom | Common Cause | Resolution |
|---|---|---|
| **`Invalid agent type "kubernetes" specified. Must be one of [any, label, none]`** | Kubernetes Plugin not installed or Jenkins not restarted | Install the **Kubernetes Plugin** and restart Jenkins (`http://<jenkins-url>/safeRestart`), or use the Scripted Pipeline / label-based syntax. |
| **`Error testing connection: 401 Unauthorized`** | Expired or incorrect token | Verify the `Secret text` token matches `kubectl get secret jenkins-admin-token -n jenkins` output. |
| **`PKIX path building failed / certificate unknown`** | Untrusted API server TLS cert | Paste the cluster `ca.crt` into the *Kubernetes server certificate key* field, or check *Disable https certificate check*. |
| **`Agent pod starts but never connects / times out`** | Jenkins tunnel or Jenkins URL unreachable from pod | Ensure `Jenkins URL` and `Jenkins tunnel` use IPs/hostnames that pods inside the cluster can route to, and TCP port `50000` is open. |
| **`Pod cannot be scheduled / 403 Forbidden`** | RBAC permissions missing | Verify `ClusterRoleBinding` is bound to the `jenkins-admin` ServiceAccount in namespace `jenkins`. |

