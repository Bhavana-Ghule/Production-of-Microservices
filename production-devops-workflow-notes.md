# Production DevOps Workflow — Jenkins, GitOps, Argo CD, Kubernetes, AWS Load Balancer Controller, RBAC

> **Purpose:** Clean GitHub-ready version of the handwritten notes in the supplied 12-page PDF. The original topics and examples are retained where legible. Important explanations and corrections are highlighted for easier learning.
>
> **Reading note:** Some handwriting and a few command names in the photos are not fully legible. Those parts are reconstructed only where the surrounding context makes the meaning clear. Added explanations are labelled **Correction / clarification** so they are not confused with a literal transcription.

---

## 1. Big picture: how a request reaches an application

A typical Kubernetes production path is:

```text
User / Browser
      |
      v
DNS name (for example, Route 53)
      |
      v
AWS Application Load Balancer (ALB)
      |
      v
Kubernetes Ingress / routing rules
      |
      v
Kubernetes Service
      |
      v
Application Pods
```

The ALB is the external entry point. It receives client traffic and routes it according to configured listener rules. Kubernetes Services provide stable access to matching Pods.

**Important:** DNS, ALB, Ingress, Service, and Pod are different things. DNS resolves a name; the ALB accepts and routes network traffic; Ingress describes HTTP/HTTPS routing rules; a Service provides stable in-cluster access to Pods; Pods run the containers.

### Example architecture from the notes

The notes show a user working with GitHub and a CI/CD flow that builds and deploys application changes. They also show multiple application components/replicas behind Kubernetes routing.

A simplified flow is:

```text
Developer
   |
   v
GitHub repository
   |
   v
Jenkins CI pipeline
   |-- checkout source code
   |-- build / test
   |-- build container image
   |-- push image to registry
   |
   v
Update deployment configuration in Git
   |
   v
Argo CD watches the Git repository
   |
   v
Argo CD syncs desired state to Kubernetes
   |
   v
Deployment -> ReplicaSet -> Pods
   |
   v
Service -> Ingress / ALB -> User
```

**Important distinction:** Jenkins commonly performs **CI** (build, test, image creation and publishing). Argo CD commonly performs **GitOps-based CD** (reconciling Kubernetes with the desired configuration stored in Git). A team can design the workflow differently, but this is the pattern discussed in these notes.

---

## 2. Route 53 and DNS

The notes mention **Route 53** and DNS.

- **DNS** translates a domain name into the destination needed by a client.
- **Amazon Route 53** is AWS's DNS service.
- A DNS record can point a domain or subdomain to an AWS load balancer using an appropriate alias record.

Example concept:

```text
app.example.com  --->  ALB DNS name  --->  Kubernetes application
```

**Correction / clarification:** Route 53 does not itself deploy an application or balance application requests. It helps the client find the endpoint. The load balancer handles incoming traffic.

---

## 3. Jenkins and the CI pipeline

Jenkins is an automation server used to run a pipeline when a trigger occurs, such as a code change or a manual run.

A typical pipeline:

1. **Checkout:** download source code from Git.
2. **Build:** compile or package the application, if required.
3. **Test:** run automated tests and checks.
4. **Build image:** create a container image.
5. **Push image:** publish the image to a registry such as Docker Hub or Amazon ECR.
6. **Update deployment configuration:** update the image tag or other desired configuration in Git, where that is the chosen workflow.
7. **Deploy / hand off to GitOps:** Argo CD detects the Git change and syncs the desired state to the cluster.

### Example pipeline concept

```text
Git commit
   -> Jenkins starts
   -> checkout
   -> test
   -> docker build
   -> push image
   -> update image tag in deployment repository
   -> Argo CD sync
   -> Kubernetes rollout
```

**Important:** A successful Jenkins build does not automatically mean the application is successfully running in production. Check the image push, Git configuration change, Argo CD sync/health, Kubernetes rollout, and application health as separate stages.

### Triggering Jenkins

Common triggers include:
- Manual build.
- Webhook from a source-code platform.
- Polling the source repository (SCM polling).

**Correction / clarification:** A webhook sends a notification when an event occurs. SCM polling checks the repository on a schedule. They are different trigger mechanisms.

---

## 4. GitOps

**GitOps** means using Git as the version-controlled source of truth for the desired state of the system.

In a Kubernetes GitOps workflow:

1. Kubernetes manifests or Helm/Kustomize configuration are stored in Git.
2. A developer changes the configuration and commits it.
3. A GitOps controller such as Argo CD compares the Git desired state with the live cluster state.
4. If the states differ, Argo CD can sync the changes according to its configuration.
5. The controller continues monitoring for drift.

### Why GitOps is useful

- Configuration history and change review are available in Git.
- Changes can be reviewed through pull requests.
- The desired state is repeatable and easier to audit.
- Drift between Git and the cluster can be detected.
- Rollback can often be done by reverting a Git change and syncing again.

**Important:** GitOps does not mean that every change is automatically approved. Automatic sync, pruning, and self-healing depend on the application's Argo CD configuration and team policy.

---

## 5. Argo CD

**Argo CD** is a declarative, GitOps continuous-delivery tool for Kubernetes. It watches a Git repository and compares the desired configuration with the live Kubernetes resources.

### Argo CD application

An Argo CD **Application** connects:
- A source repository.
- A path within that repository (or a chart source).
- A destination Kubernetes cluster.
- A destination namespace.
- Sync and health settings.

Example concept:

```text
Application
  Source:
    Git repository
    Path: k8s/ or deploy/
  Destination:
    Cluster: target Kubernetes cluster
    Namespace: application namespace
```

### Sync

**Sync** means applying the desired resources from the configured source to the destination cluster so that the live resources move toward the Git-defined desired state.

- **OutOfSync:** the live state differs from the desired state in Git.
- **Synced:** the live state matches the desired state for the resources Argo CD tracks.
- **Healthy:** resources have reached their expected health state; this is separate from whether they are synced.

**Important:** `Synced` and `Healthy` are not synonyms. An application can be synced but unhealthy, for example if a Pod cannot start.

### Automatic sync

If automated sync is enabled, Argo CD can apply detected Git changes without a person clicking **Sync** in the UI. Options such as pruning and self-healing are separate settings and should be enabled intentionally.

### Argo CD installation and access

The notes refer to installing Argo CD in Kubernetes and accessing its UI. The common namespace is `argocd`.

Check resources:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

A common local access method is port-forwarding:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open `https://localhost:8080` in the same computer's browser and handle the development/self-signed certificate warning only if you understand the context.

**Security note:** Do not expose the Argo CD UI publicly without proper TLS, authentication, network restrictions, and access control. Do not commit admin passwords or tokens to Git.

---

## 6. Kubernetes manifests

A **manifest** is a YAML file that declares the desired configuration of a Kubernetes resource. For example, a Deployment manifest declares the application image and replica count.

Example Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web-app
          image: nginx:stable
          ports:
            - containerPort: 80
```

Apply a manifest manually:

```bash
kubectl apply -f deployment.yaml
```

Inspect resources:

```bash
kubectl get deployments
kubectl get pods
kubectl describe deployment web-app
```

**Correction / clarification:** In a GitOps-managed application, avoid making routine changes only with manual `kubectl apply`. Make the desired change in Git and let Argo CD reconcile it, unless the team's documented incident or emergency procedure says otherwise.

### Deployment, ReplicaSet, and Pod

```text
Deployment
   |
   v
ReplicaSet
   |
   v
Pods
```

- **Deployment:** manages declarative application updates and replica configuration.
- **ReplicaSet:** keeps the required number of Pod replicas running.
- **Pod:** the smallest deployable Kubernetes unit; it contains one or more containers.

A Deployment with `replicas: 3` asks Kubernetes to maintain three Pod replicas, subject to scheduling and resource availability.

---

## 7. Kubernetes namespace and useful commands

A **namespace** groups namespaced Kubernetes resources within a cluster. It is useful for separating applications or environments, but it is not by itself a strong security boundary.

```bash
kubectl get namespaces
kubectl create namespace my-app
kubectl get pods -n my-app
kubectl get deployments -n my-app
kubectl get services -n my-app
kubectl get all -n my-app
```

To create a Deployment:

```bash
kubectl create deployment web-app --image=nginx
```

To scale it:

```bash
kubectl scale deployment web-app --replicas=3
```

To inspect Pods across namespaces:

```bash
kubectl get pods -A
```

**Important correction:** `kubectl get all` does not literally show every resource type in the cluster. It displays a common subset of resource types.

---

## 8. Ingress, ALB, and AWS Load Balancer Controller

The notes discuss exposing Kubernetes applications through an AWS load balancer and mention an Ingress annotation.

### Main components

- **Ingress:** Kubernetes API resource describing HTTP/HTTPS routing rules.
- **AWS Load Balancer Controller:** a controller that watches Kubernetes resources and provisions/configures AWS load-balancing resources.
- **ALB:** Application Load Balancer, commonly used for HTTP/HTTPS routing.
- **Service:** provides a stable network endpoint for application Pods.
- **Pod:** runs the application container(s).

### How it works

1. Install and configure the AWS Load Balancer Controller in the EKS cluster.
2. Create the required IAM permissions and associate them with the controller's Kubernetes ServiceAccount (commonly using IRSA or EKS Pod Identity, depending on setup).
3. Create an Ingress manifest with the required class and rules.
4. The controller observes the Ingress and provisions/configures an ALB and related AWS resources.
5. Configure DNS, for example with Route 53, to direct the application hostname to the ALB.
6. The ALB routes traffic through the configured Kubernetes targets to the application.

**Important:** The controller does not work just because an Ingress YAML exists. The controller must be installed, have correct permissions, and have the required networking and cluster configuration.

### Example Ingress for an ALB

> This is an illustrative example, not a complete production configuration. Verify the annotations against the AWS Load Balancer Controller version and your cluster/network setup.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-app
                port:
                  number: 80
```

Related commands:

```bash
kubectl get ingress -A
kubectl describe ingress web-app
kubectl get svc
kubectl get pods -o wide
```

### Does using ALB require manual work?

- **Initial setup:** normally requires a person/team to install the controller, configure IAM permissions, network prerequisites, and cluster settings.
- **After setup:** creating or updating an eligible Ingress can cause the controller to create or modify AWS load-balancer resources automatically.
- **Changes:** you can change the Ingress manifest (including supported annotations), commit it to Git, and allow Argo CD to sync it if the resource is GitOps-managed.

**Important correction:** You generally do not need to manually create a separate ALB for every Ingress when the controller is managing it. The controller's grouping and configuration rules determine which load balancer is created or reused.

### `internet-facing` versus `internal`

- `internet-facing`: intended for access from the internet, subject to routing and security controls.
- `internal`: intended for private/internal access through the VPC network.

Do not choose `internet-facing` merely to make testing easier; select the exposure model required by the application.

---

## 9. RBAC — Role-Based Access Control

The notes include **RBAC (Role-Based Access Control)** and connect it to access control and security.

RBAC determines which authenticated identities can perform which actions on Kubernetes resources.

Main Kubernetes RBAC resources:

- **Role:** permissions within one namespace.
- **ClusterRole:** permissions that can apply cluster-wide or be reused in namespace bindings.
- **RoleBinding:** grants a Role or ClusterRole to users, groups, or ServiceAccounts within a namespace.
- **ClusterRoleBinding:** grants a ClusterRole at cluster scope.

### Basic concept

```text
User / Group / ServiceAccount
             |
             v
       RoleBinding
             |
             v
            Role
             |
             v
       Allowed actions
```

A Role defines permissions using API groups, resources, and verbs. Common verbs include `get`, `list`, `watch`, `create`, `update`, `patch`, and `delete`.

**Important:** Kubernetes RBAC is about authorization (what an identity is allowed to do), not authentication (proving who the identity is).

### Example Role and RoleBinding

This example grants read access to Pods in the `default` namespace to a ServiceAccount named `viewer`.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: viewer
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: ServiceAccount
    name: viewer
    namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

Apply and test (the exact command depends on the chosen identity and cluster access):

```bash
kubectl apply -f rbac.yaml
kubectl auth can-i list pods --as=system:serviceaccount:default:viewer -n default
```

**Security principle:** grant only the permissions needed (least privilege). Avoid broad `cluster-admin` access for routine workloads.

---

## 10. AWS IAM roles and Kubernetes ServiceAccounts

The notes mention AWS credentials, IAM roles, and ServiceAccounts.

For workloads running in EKS, avoid embedding long-lived AWS access keys in application code or container images.

Common approaches include:

- **IRSA (IAM Roles for Service Accounts):** associates an IAM role with a Kubernetes ServiceAccount using the cluster's OIDC provider.
- **EKS Pod Identity:** another supported way to associate IAM permissions with workloads, depending on cluster configuration and requirements.

The AWS Load Balancer Controller needs AWS permissions to manage the AWS resources it controls. Those permissions should be attached to the controller's workload identity using the documented installation method.

**Important distinction:**
- Kubernetes RBAC controls access to Kubernetes API resources.
- AWS IAM controls access to AWS services and resources.
- An EKS workload may need both, but they solve different authorization problems.

**Never place AWS access keys, passwords, or tokens in a public Git repository.** Use IAM roles, secret-management mechanisms, and appropriate access controls.

---

## 11. Argo CD and microservices

The notes discuss deploying multiple microservices through Argo CD and keeping configuration in Git.

A repository may contain one folder per service, or it may contain a common deployment structure. The repository layout depends on the team.

Example layout:

```text
repository/
├── services/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── payments/
│       ├── deployment.yaml
│       └── service.yaml
└── README.md
```

Argo CD can manage one application per service, or a parent application can manage multiple applications (often called the **App of Apps** pattern). Another option is to manage multiple resources from one Application path.

**Important:** Argo CD does not automatically create a separate Argo CD Application just because a folder exists in Git. The Application or an application-generating pattern must be configured.

### Why use separate applications?

- Each service can have its own source path and destination namespace.
- Sync and health status can be reviewed per service.
- Ownership and release processes can be separated where needed.

The exact design depends on repository layout, team ownership, and deployment requirements.

---

## 12. CRDs — Custom Resource Definitions

The notes mention **CRDs (Custom Resource Definitions)**.

A CRD extends the Kubernetes API with a new resource type. After the CRD is installed, Kubernetes can accept custom resources of that type.

For example, an operator or controller may define a custom resource that describes a higher-level application or monitoring configuration.

**Important:** A CRD defines the resource type/schema. A controller watches instances of that resource and performs the corresponding work. Installing a CRD alone does not necessarily make the desired automation happen; the matching controller/operator may also be required.

---

## 13. Helm and package management

The notes refer to package management and installation commands.

**Helm** is a package manager for Kubernetes. A Helm chart bundles templates and configuration used to create Kubernetes resources.

Common commands:

```bash
helm repo add <repo-name> <repo-url>
helm repo update
helm search repo <keyword>
helm install <release-name> <chart>
helm list -A
helm upgrade <release-name> <chart>
helm uninstall <release-name>
```

Replace placeholders with the real repository, chart, and release names.

**Correction / clarification:** Helm is not the same thing as Argo CD. Helm packages/templates Kubernetes resources; Argo CD reconciles the desired state from a configured source. Argo CD can deploy a Helm chart from a repository.

---

## 14. Troubleshooting checklist

When an application does not work, check each layer rather than assuming that a successful sync means everything is healthy.

### Git / Jenkins
- Did the correct commit reach the repository?
- Did Jenkins check out the expected branch?
- Did tests and the image build succeed?
- Was the image pushed to the correct registry and tag?
- Was the intended manifest/image tag committed?

### Argo CD
- Is the Application pointing at the correct repository, revision, and path?
- Is the destination cluster and namespace correct?
- Is the Application `Synced`?
- Is it `Healthy`?
- Are there permission, manifest-rendering, or sync errors?

### Kubernetes
```bash
kubectl get pods -A
kubectl get deployments -A
kubectl get svc -A
kubectl get ingress -A
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

Check image-pull errors, crash loops, readiness/liveness probes, resource limits, Service selectors, and target ports.

### AWS / ALB
- Is the AWS Load Balancer Controller running?
- Does it have the required IAM permissions?
- Is the Ingress class correct?
- Are the subnet tags, security groups, and target type correct?
- Are the targets healthy in the ALB target group?
- Does DNS resolve to the expected endpoint?
- Are the application and load-balancer health checks configured correctly?

**Security reminder:** Avoid sharing secret values, kubeconfig contents, access keys, or passwords when asking for troubleshooting help.

---

## 15. Quick revision — key terms

| Term | Meaning |
|---|---|
| GitOps | Git stores the desired configuration; a controller reconciles the live system to it. |
| Jenkins | Automates CI tasks such as checkout, test, build, and image publishing. |
| Argo CD | GitOps CD tool that compares Git desired state with live Kubernetes state and syncs changes. |
| Manifest | YAML/JSON declaration of a Kubernetes resource's desired state. |
| Deployment | Manages application rollout and replica configuration. |
| ReplicaSet | Maintains the desired number of Pod replicas. |
| Pod | Kubernetes unit that runs one or more containers. |
| Namespace | Logical grouping for namespaced Kubernetes resources. |
| Service | Stable network endpoint for a set of Pods. |
| Ingress | HTTP/HTTPS routing rules into a Kubernetes cluster. |
| ALB | AWS Application Load Balancer; accepts and routes application traffic. |
| AWS Load Balancer Controller | Observes Kubernetes resources and manages supported AWS load-balancer resources. |
| Route 53 | AWS DNS service. |
| RBAC | Kubernetes authorization model based on roles and bindings. |
| IAM | AWS identity and permission system. |
| ServiceAccount | Kubernetes identity used by a workload or process. |
| CRD | Extends the Kubernetes API with a custom resource type. |
| Helm chart | Packaged Kubernetes templates and configuration. |
| Sync | Argo CD operation that applies desired state from the configured source. |
| Drift | Difference between desired configuration and live state. |

---

## 16. End-to-end flow to remember

1. Developer pushes code to GitHub.
2. A webhook or another configured trigger starts Jenkins.
3. Jenkins checks out the code, tests it, builds an image, and pushes the image to a registry.
4. The deployment configuration is updated with the intended image tag (if this is the selected workflow).
5. Argo CD detects the Git change.
6. Argo CD syncs the desired manifests/chart to Kubernetes.
7. Kubernetes creates or updates Deployments, ReplicaSets, Pods, and Services.
8. The AWS Load Balancer Controller processes the configured Ingress and manages the ALB where required.
9. Route 53 DNS directs the application hostname to the load-balancer endpoint.
10. The client sends traffic through DNS → ALB → Kubernetes routing → Service/targets → application Pods.
11. Monitoring, health checks, logs, and alerts help detect problems.

**Final takeaway:** Jenkins builds and publishes the application artifact; Git records the desired deployment configuration; Argo CD reconciles that configuration into Kubernetes; Kubernetes runs the workloads; the AWS Load Balancer Controller connects supported Kubernetes routing resources to AWS load-balancing infrastructure; Route 53 supplies DNS routing.
