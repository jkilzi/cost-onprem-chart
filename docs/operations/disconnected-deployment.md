# Disconnected (Air-Gapped) Deployment Guide

Deploy Cost Management On-Premise in disconnected OpenShift environments using `oc-mirror` for chart and image mirroring.

## Overview

In disconnected environments, clusters have no direct internet access. The `oc-mirror` tool mirrors Helm charts and container images from public registries to an internal mirror registry. The cost-onprem chart is designed to support offline templating -- `helm template` works with default values only (no `--set` flags required), which is exactly how `oc-mirror` discovers images.

> **Important:** Some images used by the chart cannot be auto-discovered by
> `oc-mirror` (for example, images referenced only in Helm hooks such as
> `pre-install`/`pre-upgrade`). Those **must** be listed explicitly in the
> `additionalImages` section of the `ImageSetConfiguration`. Use
> [Discovering container images](#discovering-container-images) to generate
> the current list from your chart revision, and see
> [Step 1](#step-1-create-imagesetconfiguration) for a minimal configuration
> example.

## Prerequisites

- **oc-mirror v2** installed ([installation guide](https://docs.okd.io/latest/disconnected/mirroring/about-installing-oc-mirror-v2.html))
- Access to a mirror registry (e.g., `mirror.example.com:5000`)
- A connected workstation with internet access for running `oc-mirror`
- OpenShift CLI (`oc`) configured for the disconnected cluster

## Discovering container images

Image tags and repositories change with chart releases. Instead of copying a
static list into this document, generate the set you need from the chart and
from `oc-mirror` before each mirror run.

**Auto-discovered:** Any image that appears in the manifests `oc-mirror`
renders from the Helm chart (it runs `helm template` internally) is mirrored
automatically when you list the chart under `mirror.helm`.

**`additionalImages`:** Images that are *not* visible during that pass (commonly
resources behind [Helm hooks](https://helm.sh/docs/topics/charts_hooks/) such
as `pre-install`/`pre-upgrade`) must be added manually. The repository keeps the
canonical hook list in `.github/workflows/lint-and-validate.yml` under
`additionalImages` so CI matches disconnected mirroring; align your
`ImageSetConfiguration` with that block when you upgrade the chart.

**Not from the chart:** `install-helm-chart.sh` can pull `amazon/aws-cli` for
one-shot S3 bucket creation. That image is not part of the Helm chart; add it
to `additionalImages` only if you use that script path, or set `S3_CLI_IMAGE` to
a mirrored image, or use `SKIP_S3_SETUP=true` / manual bucket creation.

### List images from `helm template` (chart defaults)

Use the same inputs `oc-mirror` uses for discovery: render with **no** `--set`
flags so defaults match offline discovery.

```bash
cd /path/to/cost-onprem-chart/cost-onprem

helm template cost-onprem . > /tmp/cost-onprem-rendered.yaml

awk -F': ' '/^[[:space:]]+image:/{gsub(/"/,"",$2); print $2}' \
  /tmp/cost-onprem-rendered.yaml | sort -u
```

### List images from an `oc-mirror` plan (`mapping.txt`)

Run `oc-mirror` in dry-run mode with the same `ImageSetConfiguration` you will
use for mirroring (Helm section plus `additionalImages`). The tool writes a
`mapping.txt` file under the workspace directory listing every source image in
the plan.

The project CI uses a throwaway registry only as a destination for planning; you
can mirror the same way. Example:

```bash
# Example: local chart path (adjust to your layout).
cat > /tmp/imageset-config.yaml <<'EOF'
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  helm:
    local:
      - name: cost-onprem
        path: /path/to/cost-onprem-chart/cost-onprem
  additionalImages:
    # Copy from .github/workflows/lint-and-validate.yml for hook-only images.
    - name: quay.io/insights-onprem/postgresql:16
EOF

# Local registry as dry-run destination (same pattern as CI; stop/remove when done).
podman run -d --name oc-mirror-registry -p 5050:5000 docker.io/library/registry:2

oc-mirror --v2 --config /tmp/imageset-config.yaml \
  --workspace file:///tmp/oc-mirror-workspace \
  docker://localhost:5050 --dry-run --dest-tls-verify=false

MAPPING="$(find /tmp/oc-mirror-workspace -name mapping.txt -type f | head -1)"
cut -d= -f1 "$MAPPING" | sed 's|docker://||' | sort -u
```

For mirroring from the published Helm repo instead of a local path, use
`mirror.helm.repositories` as in [Step 1](#step-1-create-imagesetconfiguration)
and the same `oc-mirror` / `find` / `cut` sequence with your workspace path.

> **Why hooks matter:** `oc-mirror` discovers images by rendering the chart the
> same way Helm does for that pass. Anything not included there must appear in
> `additionalImages`. CI verifies that every image in `helm template` output is
> covered by the mirror plan; see `.github/workflows/lint-and-validate.yml`.

## Step 1: Create ImageSetConfiguration

Create a file named `imageset-config.yaml`. The `additionalImages` section
lists images that `oc-mirror` cannot discover from the Helm chart
automatically (see [Discovering container images](#discovering-container-images)).

```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  helm:
    repositories:
      - name: cost-onprem
        url: https://insights-onprem.github.io/cost-onprem-chart
        charts:
          - name: cost-onprem
            version: "0.2.10"
  # Images that oc-mirror cannot auto-discover from the Helm chart.
  # Align with .github/workflows/lint-and-validate.yml additionalImages.
  additionalImages:
    - name: registry.redhat.io/rhel10/postgresql-16:10.1
    # Only needed if using install-helm-chart.sh for bucket creation:
    - name: amazon/aws-cli:latest
```

## Step 2: Mirror to Disk

On the connected workstation, mirror the chart and images to a local archive:

```bash
oc-mirror --v2 -c imageset-config.yaml file://mirror-output
```

This creates a directory `mirror-output/` containing:
- The packaged Helm chart
- All container images as OCI archives
- A mapping file for the mirror registry

## Step 3: Transfer to Disconnected Environment

Copy the `mirror-output/` directory to the disconnected environment using your preferred transfer method (USB drive, secure file transfer, etc.).

## Step 4: Mirror to Internal Registry

On the disconnected cluster (or a bastion host with access to the mirror registry):

```bash
oc-mirror --v2 -c imageset-config.yaml \
  --from file://mirror-output \
  docker://mirror.example.com:5000
```

## Step 5: Apply ICSP/IDMS

After mirroring, `oc-mirror` generates `ImageContentSourcePolicy` (ICSP) or `ImageDigestMirrorSet` (IDMS) resources. Apply them to the cluster:

```bash
oc apply -f mirror-output/results-*/
```

This configures the cluster to pull images from the mirror registry instead of the original registries.

## Step 6: Install the Chart

Install the chart from the mirrored registry. Use the install script with the local chart:

```bash
# Option A: Use the mirrored chart directly
helm install cost-onprem oci://mirror.example.com:5000/cost-onprem/cost-onprem \
  --version 0.2.10 \
  --namespace cost-onprem \
  --create-namespace

# Option B: Use the install script with the extracted chart
USE_LOCAL_CHART=true LOCAL_CHART_PATH=./cost-onprem \
  ./scripts/install-helm-chart.sh
```

The ICSP/IDMS applied in Step 5 ensures that all image pulls are redirected to the mirror registry automatically.

## Verification

After installation, verify that all pods are running and images are pulled from the mirror registry:

```bash
# Check all pods are running
kubectl get pods -n cost-onprem -l app.kubernetes.io/instance=cost-onprem

# Verify images come from mirror registry
kubectl get pods -n cost-onprem -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u
```

## Updating Images

When new versions are released, bump the chart version in
`ImageSetConfiguration`, re-run the commands under
[Discovering container images](#discovering-container-images), and repeat the
mirror process (Steps 2-5). The install script supports version pinning:

```bash
CHART_VERSION=0.2.10 ./scripts/install-helm-chart.sh
```

> **Remember:** If `helm template` shows an image that your dry-run
> `mapping.txt` does not, add it to `additionalImages` (and update CI’s list in
> `.github/workflows/lint-and-validate.yml` when contributing upstream). CI
> fails when `helm template` images are not fully covered by `oc-mirror`.

## References

- [oc-mirror v2 documentation](https://docs.okd.io/latest/disconnected/mirroring/about-installing-oc-mirror-v2.html)
- [oc-mirror ImageSetConfiguration design](https://github.com/openshift/oc-mirror/blob/main/docs/design/imageset-configuration.md)
- [Helm chart values reference](../operations/configuration.md)
