# Pattern Catalog

A catalog of [Validated Patterns](https://validatedpatterns.io) metadata, served as a static nginx container for consumption by the [patterns-operator](https://github.com/validatedpatterns/patterns-operator).

## How it works

The `generate-catalog.sh` script queries the GitHub API for all repositories tagged with `pattern` in the `validatedpatterns` and `validatedpatterns-sandbox` organizations. For each repository it:

1. Fetches `pattern-metadata.yaml` and normalizes it into a consistent schema
1. Fetches `values-secret.yaml.template` if present
1. Writes per-pattern files under `catalog/<pattern-name>/`
1. Generates a `catalog/catalog.yaml` index listing all discovered patterns

The resulting `catalog/` directory is served by an nginx container image that the patterns-operator deploys on-cluster.

## Prerequisites

- `gh` (GitHub CLI, authenticated)
- `yq` v4+
- `jq`
- `podman`

## Usage

### Regenerate the catalog

The catalog data must be regenerated locally and committed before pushing.
The CI pipeline does **not** run `generate-catalog.sh` — it only builds the
container from whatever is already in `catalog/`.

```sh
make generate-catalog   # or ./generate-catalog.sh directly
git add catalog/
git commit -m "Update catalog"
git push origin main    # or stable-v1
```

### List all pattern repositories

```sh
./list-all-patterns.sh
```

### Build and push the container image locally

```sh
make pattern-ui-catalog-build
make pattern-ui-catalog-push
```

## CI/CD

A GitHub Actions workflow (`.github/workflows/build-and-push.yml`) triggers on
pushes to `main` or `stable-v1` when files in `catalog/`, `templates/`, or the
workflow itself change:

1. **validate-yaml** — validates all YAML files under `catalog/` with `yamllint`
1. **build-container** — builds the image for amd64 and arm64 in parallel on native runners
1. **push-multiarch-manifest** — assembles a multi-arch manifest, pushes to Quay, and signs with cosign (only in the upstream `validatedpatterns/pattern-ui-catalog` repo)

| Branch | Image tag |
|-------------|---------------|
| `main` | `:latest` |
| `stable-v1` | `:stable-v1` |

The workflow requires `QUAY_USERNAME` and `QUAY_PASSWORD` secrets configured in a `quay` environment.

## Creating a custom catalog

You can build your own catalog image containing a curated set of patterns. This
is useful when you maintain internal or third-party patterns that are not part of
the upstream `validatedpatterns` organizations.

If the patterns you want to include are all hosted on GitHub you can just run the
following script:

```sh
TOPIC=pattern ORGS="org1 org2" make generate-catalog
```

The above script will generate the catalog bit by using
any repo with the `pattern` topic (topics are some sort of tags on GitHub)
in `org1` and `org2` on github. Then you can just skip to section
[Build and push the image](#2-build-and-push-the-image).

```sh
make UPLOADREGISTRY=quay.io/<my_own_org> pattern-ui-catalog-build pattern-ui-catalog-push
```

### 1. Prepare the catalog directory

Create a `catalog/` tree with the following structure:

```txt
catalog/
  catalog.yaml
  my-pattern/pattern.yaml
  my-pattern/values-secret.yaml.template   # optional
  another-pattern/pattern.yaml
```

**`catalog/catalog.yaml`** — the index file listing every pattern in the catalog:

```yaml
generated_at: "2026-04-15T00:00:00Z"
generator_version: "1.0"
catalog_description: "My custom pattern catalog"
patterns:
  - my-pattern
  - another-pattern
```

**`catalog/<name>/pattern.yaml`** — metadata for each pattern. Required fields
are defined by `pattern.schema.json`; a minimal example:

```yaml
metadata_version: "1.0"
name: my-pattern
pattern_version: "1.0"
display_name: My Pattern
repo_url: https://github.com/my-org/my-pattern
docs_repo_url: https://github.com/my-org/my-pattern-docs
issues_url: https://github.com/my-org/my-pattern/issues
docs_url: https://my-org.github.io/my-pattern/
ci_url: https://my-org.github.io/my-pattern/ci
tier: sandbox                       # maintained | tested | sandbox
owners:
  - my-github-user
requirements:
  hub:
    compute:
      aws:
        replicas: 3
        type: m5.2xlarge
    controlPlane:
      aws:
        replicas: 3
        type: m5.xlarge
extra_features:
  hypershift_support: false
  spoke_support: false
external_requirements: null
org: my-org
spoke: null
```

**`catalog/<name>/values-secret.yaml.template`** - metadata for secret material
for the pattern. See [here](https://validatedpatterns.io/learn/secrets-management-in-the-validated-patterns-framework/) for more information.

You can optionally validate your files against the JSON schemas shipped in this
repository:

```sh
make schema-validate
```

### 2. Build and push the image

Override `UPLOADREGISTRY` (and optionally `VERSION`) to point at your own
registry:

```sh
make UPLOADREGISTRY=quay.io/my-org VERSION=1.0.0 pattern-ui-catalog-build
make UPLOADREGISTRY=quay.io/my-org VERSION=1.0.0 pattern-ui-catalog-push
```

This produces `quay.io/my-org/pattern-ui-catalog:1.0.0`.

### 3. Point the patterns-operator at your catalog image

The [patterns-operator](https://github.com/validatedpatterns/patterns-operator)
reads an optional `catalog.image` key from its ConfigMap. By default it deploys
`quay.io/validatedpatterns/pattern-ui-catalog:stable-v1`. To override it, patch
(or create) the `patterns-operator-config` ConfigMap in the operator namespace
(`patterns-operator` by default):

```sh
oc patch configmap patterns-operator-config \
  -n patterns-operator \
  --type merge \
  -p '{"data":{"catalog.image":"quay.io/my-org/pattern-ui-catalog:1.0.0"}}'
```

If the ConfigMap does not exist yet:

```sh
oc create configmap patterns-operator-config \
  -n patterns-operator \
  --from-literal=catalog.image=quay.io/my-org/pattern-ui-catalog:1.0.0
```

The operator manager pod needs to be deleted for the change to be picked up.
After that the UI will point to the new catalog.

## Authenticated container registries

If your custom catalog image is hosted on a registry that requires
authentication (e.g. a private Quay repository), you need to configure a pull
secret so that the cluster can pull the image.

### 1. Create a pull secret

Create an authentication secret in the operator namespace:

```sh
oc create secret docker-registry my-catalog-pull-secret \
  -n patterns-operator \
  --docker-server=quay.io \
  --docker-username='<username>' \
  --docker-password='<password>'
```

### 2. Link the secret to the operator service account

The catalog deployment runs under the `patterns-operator-controller-manager`
service account. Link the pull secret to it:

```sh
oc secrets link patterns-operator-controller-manager my-catalog-pull-secret \
  -n patterns-operator \
  --for=pull
```

### 3. Alternative: global pull secret

If multiple workloads need access to the same registry, you can add the
credentials to the cluster-wide pull secret instead. Extract the current secret,
merge your credentials, and apply:

```sh
oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' > /tmp/pull-secret.json

# Authenticate to your registry (this merges the new entry into the file)
oc registry login --registry=quay.io \
  --auth-basic='<username>:<password>' \
  --to=/tmp/pull-secret.json

oc set data secret/pull-secret -n openshift-config \
  --from-file=.dockerconfigjson=/tmp/pull-secret.json
```

> **Note:** Updating the global pull secret triggers a rolling reboot of all
> nodes in the cluster. Prefer the per-service-account approach when only the
> catalog needs access.

## Repository structure

```
catalog/                  # Generated catalog data (served by nginx)
  catalog.yaml            # Index of all patterns
  <pattern>/pattern.yaml  # Normalized metadata per pattern
  <pattern>/values-secret.yaml.template  # Secret template (if available)
.github/workflows/        # CI/CD workflow
templates/                # Dockerfile template
generate-catalog.sh       # Catalog generation script
list-all-patterns.sh      # Lists all pattern repos
Makefile                  # Build targets
```
