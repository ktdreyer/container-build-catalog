# Pushing the buildah-remote-oci-ta bundle to quay.io

This fork is not published through the usual konflux-ci release machinery
(`task/tkn-bundle` run by CI). Push the task bundle by hand with `tkn`
(tested with tkn v0.45.2).

## Prerequisites

Log in to quay.io once with the account that owns
`quay.io/ktdreyer/tekton-catalog`:

```
podman login quay.io
```

`tkn bundle push` reads registry credentials from the default docker config
(`~/.docker/config.json`) or podman's auth file
(`~/.config/containers/auth.json`), so after `podman login quay.io` no extra
flags are needed for the push. (This is how the pushes of `0.14.1-cache1`
through `0.14.1-cache3` were authenticated: `~/.docker/config.json` did not
even exist; tkn found quay.io credentials in podman's auth file.)

## Push

From the clone root, with `$REVISION` set to the commit you are publishing
(the commit containing the task YAML) and `$VERSION` set to the task's
`app.kubernetes.io/version` label:

```
REVISION=$(git rev-parse HEAD)
VERSION=0.14.1-cache3

tkn bundle push \
  --annotate "org.opencontainers.image.source=https://github.com/ktdreyer/container-build-catalog" \
  --annotate "org.opencontainers.image.revision=${REVISION}" \
  --annotate "org.opencontainers.image.url=https://github.com/ktdreyer/container-build-catalog/tree/${REVISION}/task/buildah-remote-oci-ta" \
  --annotate "org.opencontainers.image.documentation=https://github.com/ktdreyer/container-build-catalog/tree/${REVISION}/task/buildah-remote-oci-ta/README.md" \
  --annotate "org.opencontainers.image.description=Buildah task builds source code into a container image and pushes the image into container registry using buildah tool." \
  --annotate "org.opencontainers.image.version=${VERSION}" \
  quay.io/ktdreyer/tekton-catalog/task-buildah-remote-oci-ta:${VERSION} \
  -f task/buildah-remote-oci-ta/buildah-remote-oci-ta.yaml
```

The annotation set mirrors what upstream's `tkn-bundle` task sets
(`task/tkn-bundle/tkn-bundle.yaml` in konflux-ci/container-build-catalog);
the `description` annotation is the first paragraph of the task's
`spec.description`.

The exact invocation used to publish `0.14.1-cache3` (revision
`6bddec84ae03eb666e539c2fc0249c551ea9ae51`) was:

```
tkn bundle push \
  --annotate "org.opencontainers.image.source=https://github.com/ktdreyer/container-build-catalog" \
  --annotate "org.opencontainers.image.revision=6bddec84ae03eb666e539c2fc0249c551ea9ae51" \
  --annotate "org.opencontainers.image.url=https://github.com/ktdreyer/container-build-catalog/tree/6bddec84ae03eb666e539c2fc0249c551ea9ae51/task/buildah-remote-oci-ta" \
  --annotate "org.opencontainers.image.documentation=https://github.com/ktdreyer/container-build-catalog/tree/6bddec84ae03eb666e539c2fc0249c551ea9ae51/task/buildah-remote-oci-ta/README.md" \
  --annotate "org.opencontainers.image.description=Buildah task builds source code into a container image and pushes the image into container registry using buildah tool." \
  --annotate "org.opencontainers.image.version=0.14.1-cache3" \
  quay.io/ktdreyer/tekton-catalog/task-buildah-remote-oci-ta:0.14.1-cache3 \
  -f task/buildah-remote-oci-ta/buildah-remote-oci-ta.yaml
```

It printed:

```
Pushed Tekton Bundle to quay.io/ktdreyer/tekton-catalog/task-buildah-remote-oci-ta@sha256:3c049e798659306cda516ee94ed243e893e33b2bca012bcc850bd2f700f1c5a8
```

## Verify

The quay repository is public, so everything can be verified anonymously.

Inspect the tag (shows the digest and confirms anonymous pulls work). The
OCI annotations are on the image *manifest*, so use `--raw` to see them:

```
skopeo inspect --no-creds docker://quay.io/ktdreyer/tekton-catalog/task-buildah-remote-oci-ta:0.14.1-cache3
skopeo inspect --raw --no-creds docker://quay.io/ktdreyer/tekton-catalog/task-buildah-remote-oci-ta:0.14.1-cache3 | python3 -m json.tool
```

Pull the bundle back anonymously and diff the embedded task against the YAML
file. Bundles store resources as JSON, so compare the parsed objects rather
than the raw text:

```
skopeo copy --src-no-creds \
  docker://quay.io/ktdreyer/tekton-catalog/task-buildah-remote-oci-ta:0.14.1-cache3 \
  dir:/tmp/bundle
LAYER_DIGEST=$(python3 -c "import json; print(json.load(open('/tmp/bundle/manifest.json'))['layers'][0]['digest'].split(':')[1])")
mkdir /tmp/bundle-layer && tar -xzf "/tmp/bundle/${LAYER_DIGEST}" -C /tmp/bundle-layer
python3 - <<'EOF'
import json, yaml
bundle = json.load(open('/tmp/bundle-layer/buildah-remote-oci-ta'))
source = yaml.safe_load(open('task/buildah-remote-oci-ta/buildah-remote-oci-ta.yaml'))
assert bundle == source, "bundle does not match the YAML"
print("bundle matches task/buildah-remote-oci-ta/buildah-remote-oci-ta.yaml")
EOF
```

## Pitfalls

- Do **not** pass `--remote-skip-tls` to `tkn bundle push`: tkn v0.45.2
  segfaults on it (it dereferences a nil TLS config) regardless of the
  target registry. There is no need for it anyway: quay.io uses regular TLS,
  so push straight to quay.io. (This is also why a detour through a local
  plain-HTTP registry does not work with this tkn version.)
- `skopeo copy` has no `--no-creds` flag; use `--src-no-creds` to fetch
  anonymously. `skopeo inspect` does accept `--no-creds`.
- `tkn bundle push` creates the image config with a 1970-01-01 creation
  timestamp by default, so `skopeo inspect` reporting a 1969/1970 date for
  the bundle is expected.
