# Kubernetes Rollout Guard

A GitHub Action that **guards** a Kubernetes `Deployment` rollout: it waits for the
rollout to become ready within a timeout and, if it doesn't, it surfaces the
**actual error** (deployment/pod `describe`, current *and* previous container
logs, and namespace events) directly in the GitHub Actions run — then rolls the
Deployment back to the previous revision and fails the job.

Unlike most Kubernetes deploy actions, this one does **not** deploy. It assumes
you've already updated the Deployment (e.g. `kubectl set image`) and `kubectl`
is authenticated against the cluster. It focuses on the part that's usually a
black box in CI: *why didn't the rollout come up, and how do we get back to a
good state automatically?*

## Features

- ⏱️ **Bounded wait** — `kubectl rollout status --timeout` instead of waiting forever.
- 🔎 **Real diagnostics on failure** — collapsible log groups with deployment &
  pod `describe`, current + `--previous` container logs, and recent namespace events.
- 🧾 **Honest job summary** — a `$GITHUB_STEP_SUMMARY` block that reports what
  *actually* happened, including whether the rollback succeeded.
- ⏪ **Automatic rollback** — `kubectl rollout undo` to the previous revision,
  keeping the last stable deployment serving. Toggle off with `rollback: false`.
- ❌ **Fails the job** so the pipeline reflects reality.

## Usage

```yaml
- name: Update image
  run: |
    kubectl set image -n my-namespace \
      deployment/my-app \
      my-app=registry.example.com/my-app@${{ steps.build.outputs.digest }}

- name: Guard the rollout
  uses: sarveshacharya/k8s-rollout-guard@v1
  with:
    deployment: my-app
    namespace: my-namespace
    timeout: 300s
```

`kubectl` must already be configured — for example via
[`azure/setup-kubectl`](https://github.com/marketplace/actions/kubectl-tool-installer)
plus your cloud's credential action
([`google-github-actions/get-gke-credentials`](https://github.com/google-github-actions/get-gke-credentials),
[`aws-actions/amazon-eks`](https://github.com/aws-actions), `az aks get-credentials`, etc.).

### Per-environment timeout

Keep the timeout generous so a slow-but-healthy rollout is not wrongly failed
and rolled back:

```yaml
- uses: sarveshacharya/k8s-rollout-guard@v1
  with:
    deployment: my-app
    namespace: ${{ github.ref_name }}
    timeout: >-
      ${{
        github.ref_name == 'development' && '300s' ||
        '600s'
      }}
```

## Inputs

| Name         | Required | Default | Description |
|--------------|----------|---------|-------------|
| `deployment` | yes      | —       | Deployment name (used as `deployment/<name>`). |
| `namespace`  | yes      | —       | Namespace the deployment lives in. |
| `timeout`    | no       | `300s`  | Max time to wait for the rollout (and for the rollback to become healthy). Accepts any `kubectl --timeout` value, e.g. `300s`, `10m`. |
| `rollback`   | no       | `true`  | Run `kubectl rollout undo` on failure. Set `false` to diagnose + fail without rolling back. |
| `log-tail`   | no       | `200`   | Log lines captured per container when diagnosing a failure. |

## Required cluster permissions (RBAC)

The identity `kubectl` uses needs, in the target namespace:

- `deployments`: `get`, `patch`, `update` (for rollout status + `rollout undo`)
- `replicasets`: `get`, `list`
- `pods` and `pods/log`: `get`, `list`
- `events`: `get`, `list`

If `pods/log`, `events`, or the rollback verbs are missing, diagnostics come back
empty and the rollback reports as failed.

## Notes & caveats

- **`rollout undo` reverts to the _previous_ revision**, which is the last *stable*
  one only if the prior deploy succeeded. A first-ever deploy, or two failures in
  a row, leaves nothing good to roll back to — the summary reports this honestly.
- **Deployment strategy matters.** With `RollingUpdate`, the old pods keep serving
  while the new revision fails, so rollback is seamless. With `Recreate`, there is
  a downtime window until the rollback restores the old pods.
- Pods are matched by name prefix (`<deployment>-…`); ensure your deployment name
  doesn't prefix an unrelated deployment in the same namespace.

## Versioning

Reference the moving major tag for automatic patch/minor updates:

```yaml
uses: sarveshacharya/k8s-rollout-guard@v1
```

Or pin to an exact release / commit SHA for maximum reproducibility:

```yaml
uses: sarveshacharya/k8s-rollout-guard@v1.0.0
```

## Releasing (maintainers)

Cutting a release is a single tag push — the [`Release` workflow](.github/workflows/release.yml)
does the rest:

```bash
git tag v1.0.0
git push origin v1.0.0
```

On that push it (re)points the moving major tag `v1` at the release commit and
creates a GitHub Release with generated notes. Consumers pinned to `@v1` pick up
the new version automatically; nothing else is done by hand.

> First release only: the `v1` tag is created the first time you push a `v1.x.x`
> tag. Until then, `uses: …@v1` cannot resolve.

## License

[MIT](./LICENSE)
