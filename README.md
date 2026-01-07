```markdown
# Apigee Hybrid Configuration Manager

This project manages Apigee Hybrid configuration entities (Developers, API Products, KVMs, TargetServers, etc.) as code using the [Apigee Config Maven Plugin](https://github.com/apigee/apigee-config-maven-plugin).

It is designed for **Apigee Hybrid (v1 API)** and uses a multi-file directory structure.

---

## 📋 Prerequisites

Ensure you have the following installed before running the scripts:

1.  **Maven 3.x** (`mvn -v`)
2.  **Google Cloud SDK** (`gcloud version`)
3.  **Access:** You must be logged into gcloud with a user that has Apigee Admin permissions.

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
        └── env             # Environment-level configs (optional)
            └── walmart
                └── targetServers.json

```

---

## 🚀 How to Run

### 1. Authentication

We use the Google Cloud CLI to generate a short-lived Bearer token. You do not need hardcoded credentials.

```bash
# Verify you are logged in
gcloud auth login

```

### 2. Common Flags

All commands use these standard flags. Replace values as needed:

| Flag | Description | Value |
| --- | --- | --- |
| `-Pwalmart` | The Maven Profile ID (defined in `shared-pom.xml`) | `walmart` |
| `-Dapigee.org` | Your Apigee Organization ID | `apigee-hybrid-eval-1-xxxxx` |
| `-Dapigee.apiversion` | **CRITICAL:** API Version for Hybrid | `v1` |
| `-Dapigee.bearer` | OAuth Token | `$(gcloud auth print-access-token)` |

---

## 🛠️ Operations Guide

### A. Apply Configuration (Add or Update)

**Use this for daily deployments.**
This command ensures that every entity listed in your JSON files exists on the server.

* **If missing:** It creates the entity.
* **If exists:** It updates the entity to match your JSON.

```bash
mvn clean install -Pwalmart \
  -Dapigee.bearer="$(gcloud auth print-access-token)" \
  -Dapigee.org="apigee-hybrid-eval-1-477813" \
  -Dapigee.config.dir=resources/edge \
  -Dapigee.config.options=update \
  -Dapigee.apiversion=v1

```

> **Note:** We recommend `options=update` over `create`. The `create` option will skip entities if it thinks they already exist, preventing updates to quotas or descriptions.

---

### B. Delete Specific Resources (The "Cleanup" Pattern)

The plugin does **not** automatically delete users just because you removed them from the main `developers.json` file. To delete a specific resource safely:

1. Create a temporary directory (e.g., `resources/cleanup/org`).
2. Create a JSON file containing **ONLY** the items to delete.
3. Run the plugin pointing to that directory with `options=delete`.

**Example: Deleting a single developer**

1. Create the cleanup file:
```bash
mkdir -p resources/cleanup/org
cat > resources/cleanup/org/developers.json <<EOF
[ { "email": "john.wesp@walmart.com" } ]
EOF

```


2. Run the delete command:
```bash
mvn clean install -Pwalmart \
  -Dapigee.bearer="$(gcloud auth print-access-token)" \
  -Dapigee.org="apigee-hybrid-eval-1-477813" \
  -Dapigee.apiversion=v1 \
  -Dapigee.config.dir=resources/cleanup \
  -Dapigee.config.options=delete

```



---

### C. Delete ALL Configuration (⚠️ Danger Zone)

**WARNING:** This will delete **EVERYTHING** listed in your current `resources/edge` folder from the Apigee server. Use this only for tearing down a sandbox environment.

```bash
mvn clean install -Pwalmart \
  -Dapigee.bearer="$(gcloud auth print-access-token)" \
  -Dapigee.org="apigee-hybrid-eval-1-477813" \
  -Dapigee.config.dir=resources/edge \
  -Dapigee.config.options=delete \
  -Dapigee.apiversion=v1

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

```

```