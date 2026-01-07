```markdown
# Apigee Hybrid Configuration Manager

This project manages Apigee Hybrid configuration entities (Developers, API Products, KVMs, TargetServers, etc.) as code using the [Apigee Config Maven Plugin](https://github.com/apigee/apigee-config-maven-plugin).

It is designed for **Apigee Hybrid (v1 API)** and uses a multi-file directory structure.

---

## 📋 Prerequisites

Ensure you have the following installed to use this repository:

1.  **Argo Workflows** (Running in your Kubernetes/GKE cluster)
2.  **Maven 3.x** (For local testing)
3.  **Google Cloud SDK** (For local authentication)
4.  **Access:** A Google Cloud Service Account with **Apigee Organization Admin** role.

---

## 📂 Project Structure

The plugin enforces a strict directory convention. All configuration files must reside in `resources/edge`.

```text
.
├── pom.xml                 # Child POM (Execution logic)
├── shared-pom.xml          # Parent POM (Org/Env definitions)
└── resources
    └── edge
        ├── org             # Organization-level configs
        │   ├── developers.json
        │   └── apiProducts.json
        └── env             # Environment-level configs
            └── walmart
                └── targetServers.json

```

---

## 🚀 Deployment (Argo Workflows)

This is the standard process for deploying configuration changes to the environment.

### 1. Register the Workflow Template

This template defines the logic for cloning the repo, fetching authentication, and running the Maven plugin. You only need to apply this once.

**File:** `apigee-config-deployer-template.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: apigee-config-deployer
  namespace: argo
spec:
  serviceAccountName: argo-workflow 
  entrypoint: main
  
  volumeClaimTemplates:
    - metadata:
        name: workdir
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi

  templates:
    - name: main
      inputs:
        parameters:
          - name: repo_url
          - name: config_path      # Path to the folder containing pom.xml
            value: "." 
          - name: apigee_env
          - name: apigee_org       
          - name: action           # 'update' or 'delete'
            value: "update"
      dag:
        tasks:
          - name: clone-code
            template: git-clone
            arguments:
              parameters:
                - name: url
                  value: "{{inputs.parameters.repo_url}}"

          - name: maven-config-deploy
            dependencies: [clone-code]
            template: apigee-config
            arguments:
              parameters:
                - name: apigee_env
                  value: "{{inputs.parameters.apigee_env}}"
                - name: apigee_org
                  value: "{{inputs.parameters.apigee_org}}"
                - name: path
                  value: "{{inputs.parameters.config_path}}"
                - name: action
                  value: "{{inputs.parameters.action}}"

    # STEP 1: GIT CLONE 
    - name: git-clone
      inputs:
        parameters:
          - name: url
      container:
        image: alpine/git:latest
        workingDir: /src
        env:
          - name: GITHUB_TOKEN
            valueFrom:
              secretKeyRef:
                name: github-creds
                key: token
          - name: GITHUB_USER
            valueFrom:
              secretKeyRef:
                name: github-creds
                key: username
        command: [sh, -c]
        args: 
          - |
            set -e
            echo "--- Cleaning workspace ---"
            rm -rf ./* .[a-zA-Z]* || true
            
            echo "--- Cloning repository ---"
            REPO_PATH=$(echo "{{inputs.parameters.url}}" | sed 's~https://~~')
            git clone "https://${GITHUB_USER}:${GITHUB_TOKEN}@${REPO_PATH}" .
        volumeMounts:
          - name: workdir
            mountPath: /src

    # STEP 2: APIGEE CONFIG DEPLOYMENT
    - name: apigee-config
      inputs:
        parameters:
          - name: apigee_env
          - name: apigee_org
          - name: path
          - name: action
      container:
        image: maven:3.8.5-openjdk-11
        workingDir: /src/{{inputs.parameters.path}}
        command: [sh, -c]
        args:
          - |
            set -e
            echo "--- 1. Fetching Token (Metadata Server) ---"
            TOKEN=$(curl -s -H "Metadata-Flavor: Google" \
              "[http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token](http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token)" \
              | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)

            if [ -z "$TOKEN" ]; then echo "Token failed"; exit 1; fi

            echo "--- 2. Applying Config (Action: {{inputs.parameters.action}}) ---"
            
            mvn clean install -P{{inputs.parameters.apigee_env}} \
              -Dapigee.bearer="$TOKEN" \
              -Dapigee.org="{{inputs.parameters.apigee_org}}" \
              -Dapigee.apiversion=v1 \
              -Dapigee.config.dir=resources/edge \
              -Dapigee.config.options={{inputs.parameters.action}}
        volumeMounts:
          - name: workdir
            mountPath: /src

```

**Apply the template to your cluster:**

```bash
kubectl apply -f apigee-config-deployer-template.yaml

```
**Submit a Deployment**

```bash
argo submit --from wftmpl/apigee-config-deployer   -p repo_url="https://github.com/git-azeez/apigee-hybrid-config.git"   -p apigee_env="walmart"   -p apigee_org="apigee-hybrid-eval-1-477813"   -p config_path="."   -p action="update"   --watch

```

### 2. Trigger a Deployment (Declarative)

To execute the deployment, create a Workflow manifest. This file acts as the trigger.

**File:** `run-config-update.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: apigee-config-update-
  namespace: argo
spec:
  workflowTemplateRef:
    name: apigee-config-deployer
  arguments:
    parameters:
    - name: repo_url
      value: "[https://github.com/git-azeez/apigee-hybrid-config.git](https://github.com/git-azeez/apigee-hybrid-config.git)"
    - name: apigee_env
      value: "walmart"
    - name: apigee_org
      value: "apigee-hybrid-eval-1-477813"
    - name: config_path
      value: "." 
    - name: action
      value: "update"

```

**Submit the Workflow:**

```bash
argo submit run-config-update.yaml --watch

```

---

## 💻 Local Development & Testing

Use these commands for local validation before pushing to Git.

### 1. Authentication

```bash
gcloud auth login

```

### 2. Apply Configuration (Local)

```bash
mvn clean install -Pwalmart \
  -Dapigee.bearer="$(gcloud auth print-access-token)" \
  -Dapigee.org="apigee-hybrid-eval-1-477813" \
  -Dapigee.config.dir=resources/edge \
  -Dapigee.config.options=update \
  -Dapigee.apiversion=v1

```

### 3. Deletion / Cleanup (Local)

To delete specific resources, point the plugin to a folder containing **only** the items to delete.

```bash
# Example: Point to a cleanup folder containing just one developer to delete
mvn clean install -Pwalmart \
  -Dapigee.bearer="$(gcloud auth print-access-token)" \
  -Dapigee.org="apigee-hybrid-eval-1-477813" \
  -Dapigee.apiversion=v1 \
  -Dapigee.config.dir=resources/cleanup \
  -Dapigee.config.options=delete

```

---

## 📝 Configuration Guidelines (Hybrid v1 vs Edge)

Apigee Hybrid (Google Cloud) has stricter JSON schemas than legacy Apigee Edge.

### Developers (`developers.json`)

* ❌ **DO NOT** include a `"name"` field.
* ✅ Use `"email"` as the identifier.

```json
[
  {
    "email": "jane.doe@walmart.com",
    "firstName": "Jane",
    "lastName": "Doe",
    "userName": "jane"
  }
]

```

### API Products (`apiProducts.json`)

* ❌ **DO NOT** use objects for `quota`.
* ✅ Quota fields must be flat strings.

```json
{
  "name": "my-product",
  "proxies": ["my-proxy"],
  "quota": "10000",
  "quotaInterval": "1",
  "quotaTimeUnit": "month"
}

```

---

## ❓ Troubleshooting

| Error | Cause | Fix |
| --- | --- | --- |
| `404 Not Found .../null/organizations/...` | Missing API Version. | Add `-Dapigee.apiversion=v1` to your command. |
| `400 Bad Request ... Unknown name "name"` | Invalid JSON Schema. | Remove `"name"` field from `developers.json`. |
| `400 Bad Request ... Starting an object on a scalar field` | Invalid Quota format. | Flatten the quota object in `apiProducts.json`. |
| `Service Account file or bearer token is missing` | Plugin didn't receive auth. | Ensure you use `-Dapigee.bearer`, not just `-Dbearer`. |
| `fatal: repository '...' not found` | Typo in URL or Auth failure. | Check URL spelling. Check `github-creds` secret. |

```

```
---

Reference : https://github.com/apigee/apigee-config-maven-plugin
Reference : https://github.com/argoproj/argo-workflows
