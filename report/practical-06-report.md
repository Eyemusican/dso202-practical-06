# DSO202 Practical 6 Report: Packaging and Deploying with Helm


## 1. Aim

The aim of this practical was to learn how Helm packages, configures and manages Kubernetes applications. I wanted to use a published chart through its full release lifecycle, build my own chart from scratch, configure it for different environments with values files, and validate it before it reaches the cluster.

---

## 2. Introduction

In Practical 5 I used Kustomize to manage one app across environments by patching plain YAML. Helm does a similar job in a different way. Instead of patching, it renders templates with values. It also keeps a record of every installation, so I can upgrade, roll back and uninstall a whole app as one unit.

The main Helm terms I used in this practical are:

- **Chart:** a folder with `Chart.yaml`, `values.yaml` and a `templates/` folder. It describes objects but creates nothing on its own.
- **Release:** one installation of a chart in one namespace.
- **Revision:** a numbered snapshot of a release. Install makes revision 1, and every upgrade or rollback makes the next number.
- **Values:** the settings that fill in the templates.

I used Helm v4.3.0 on my kind cluster from Practical 1.

---

## 3. Use Case

The use case is a team that needs to run the same nginx web page in dev and prod with small differences: replica count, resources, service type and page content. Writing the manifests twice would make them drift apart. With Helm, I wrote the chart once, kept one values file per environment, and Helm renders the right manifests for each. Before that, I practised on podinfo, which is how most teams use Helm in real life: installing charts other people publish.

---

## 4. Architecture

![Architecture](../evidence/architecture.png)

This shows how the webapp chart deploys when installed with the dev values. In Stages 3 to 5 I rendered and validated it locally, following the notes.

My cluster is a kind cluster called `dso202` with 1 control plane and 2 workers, running Kubernetes v1.36.1. Host port 30080 is mapped to the control plane, which the dev release uses for its NodePort.

The webapp chart creates four objects:

| Object | Purpose |
| --- | --- |
| ConfigMap | Holds the HTML page, built from values |
| Deployment | Runs nginx and mounts the page |
| Service | Gives the pods a stable address (ClusterIP or NodePort) |
| Test Pod | Only run by `helm test`, checks the page shows the right environment |

---

---

## 5. Implementation

### Stage 0: Installing Helm and preparing the cluster

I checked my Practical 1 cluster was still running, installed Helm 4 with the official script, and confirmed there was no link to the unsafe `baltocdn` mirror. My kubectl context was still set to the Assignment 1 namespace, so I reset it to `default` to match the notes. My Assignment 1 frontend Service was also holding port 30080, so I deleted it.

### Stage 1: Consuming a published chart

I added the podinfo repo, searched it, read the chart and its values, then installed it as `my-podinfo` with 2 replicas and a custom message. My first install timed out because the image pull inside the kind nodes was very slow. I pulled the image with docker on my laptop, loaded it into kind, uninstalled the failed release and installed again, which worked straight away.

After that I inspected the release, decoded its record Secret, did a wrong upgrade on purpose to see values get lost, fixed it with `--reuse-values`, rolled back to revision 1, uninstalled, and finally installed the same chart from an OCI registry.

### Stage 2: Chart structure

I generated a scaffold with `helm create` to see the standard layout, deleted it, then started my own `webapp` chart with `Chart.yaml`, `values.yaml` and `.helmignore`.

### Stage 3: Writing the templates

I wrote `_helpers.tpl`, `configmap.yaml`, `deployment.yaml`, `service.yaml`, `NOTES.txt` and `tests/test-connection.yaml`, plus the two environment files `dev.yaml` and `prod.yaml` outside the chart. Then I rendered everything locally with `helm template`.

### Stage 4: Values precedence and merging

I tested how `--set` and `-f` files override each other, layered the two environment files in both orders, compared `--set` with `--set-string`, and deleted a default with `null`.

### Stage 5: Validation

I broke the chart in four ways (missing required value, bad indentation, number instead of text, misspelled field), checked which tool catches each one, added a `values.schema.json`, fixed everything, and confirmed both environments pass.

---

## 6. Results and Evidence

### Stage 0

My cluster was still there with all 3 nodes Ready.

![Cluster nodes](../evidence/stage0-cluster-nodes.png)

Helm v4.3.0 installed, and the checksum was verified.

![Helm install](../evidence/stage0-helm-install.png)

No APT source points to the unsafe mirror.

![No baltocdn](../evidence/stage0-no-baltocdn.png)

The version matches the notes. Only GoVersion differs, which the notes say changes between builds.

![Helm version](../evidence/stage0-helm-version.png)

After resetting my context, `HELM_NAMESPACE` showed `default`, and `HELM_MAX_HISTORY` is 10.

![Helm env](../evidence/stage0-helm-env.png)

`helm list -A` printed only headings, so Helm could reach the cluster and no releases existed yet.

![Helm list empty](../evidence/stage0-helm-list-empty.png)

After deleting my old Assignment 1 Service, port 30080 was free, and the cluster maps that port from the host.

![Port free](../evidence/stage0-port-30080-free.png)

![Port mapping](../evidence/stage0-port-mapping.png)

### Stage 1

I added the podinfo repo and downloaded its index.

![Repo add](../evidence/stage1-repo-add-update.png)

The search shows two version columns: chart version and app version.

![Search repo](../evidence/stage1-search-repo.png)

I read the chart before installing. It uses `apiVersion: v1`, the old Helm 2 format, which Helm 4 still installs.

![Show chart](../evidence/stage1-show-chart.png)

![Show values](../evidence/stage1-show-values.png)

My first install timed out because the pods were stuck pulling the image.

![Install timeout](../evidence/stage1-install-timeout.png)

![Image pull slow](../evidence/stage1-image-pull-slow.png)

I pulled the image on my laptop and loaded it into kind. After that the pods were running.

![Docker pull](../evidence/stage1-docker-pull-podinfo.png)

![Kind load](../evidence/stage1-kind-load-podinfo.png)

![Pods running](../evidence/stage1-pods-running-after-fix.png)

I uninstalled the failed release and installed again. This time it was deployed as revision 1.

![Helm install](../evidence/stage1-helm-install.png)

The release only shows when I give `-n dso202-helm`, which proves releases belong to a namespace.

![Helm list](../evidence/stage1-helm-list.png)

Helm created a Deployment, a Service and 2 pods.

![Objects](../evidence/stage1-objects.png)

With a port forward, curl showed my custom message, so the `--set` override worked.

![Curl podinfo](../evidence/stage1-curl-podinfo.png)

`helm status` in Helm 4 also lists the live resources.

![Helm status](../evidence/stage1-helm-status.png)

`helm get values` shows only my values, `--all` shows everything merged, and `get manifest` shows which template made each object.

![Helm get](../evidence/stage1-helm-get.png)

Helm saved the release as a Secret, and its labels show the release name, revision and status.

![Release secret](../evidence/stage1-release-secret.png)

When I decoded it, my values were there in plain text, and `apply_method` was `ssa`.

![Release decoded](../evidence/stage1-release-decoded.png)

The wrong upgrade with only `--set ui.color` dropped my replicas from 2 to 1 and removed my message.

![Values lost](../evidence/stage1-upgrade-values-lost.png)

Using `--reuse-values` kept the old values and added the new colour.

![Reuse values](../evidence/stage1-upgrade-reuse-values.png)

Rolling back to revision 1 created revision 4 instead of deleting revisions 2 and 3. This is the history table the checkpoint asks for.

![Rollback history](../evidence/stage1-rollback-history.png)

Uninstall removed all objects and records, but the namespace stayed.

![Uninstall](../evidence/stage1-uninstall.png)

Installing from OCI needed no `repo add`. Helm printed the digest of the exact chart it pulled.

![OCI install](../evidence/stage1-oci-install.png)

![OCI uninstall](../evidence/stage1-oci-uninstall.png)

### Stage 2

The Helm 4 scaffold includes `httproute.yaml` for the Gateway API next to `ingress.yaml`.

![Scaffold](../evidence/stage2-helm-create-scaffold.png)

My `Chart.yaml` and `values.yaml`:

![Chart yaml](../evidence/stage2-chart-yaml.png)

![Values yaml](../evidence/stage2-values-yaml.png)

The chart skeleton matches the Stage 2 checkpoint.

![Chart skeleton](../evidence/stage2-chart-skeleton.png)

### Stage 3

The full render worked and produced 4 objects in kind order: ConfigMap, Service, Deployment, then the test Pod.

![Render objects](../evidence/stage3-template-render-objects.png)

The Service name is just `webapp-dev`, the selector uses only the 2 stable labels, and `nodePort: 30080` appears because the type is NodePort.

![Render service](../evidence/stage3-render-service.png)

In the Deployment, the image became `nginx:1.30-alpine` from `appVersion`, and `toYaml` printed `limits` before `requests`.

![Render deployment 1](../evidence/stage3-render-deployment-1.png)

![Render deployment 2](../evidence/stage3-render-deployment-2.png)

My checksum is different from the notes because the hash depends on the exact bytes of the rendered ConfigMap. Rendering 3 times gave the same hash, so it is repeatable.

![Checksum repeatable](../evidence/stage3-checksum-repeatable.png)

The rendered ConfigMap has every value filled in.

![Render configmap](../evidence/stage3-render-configmap.png)

### Stage 4

`--set replicaCount=2` beat the value in `dev.yaml`.

![Set overrides file](../evidence/stage4-set-overrides-file.png)

With dev then prod, the "prod" render still had dev's NodePort, because prod.yaml never mentions `service`.

![Dev then prod](../evidence/stage4-dev-then-prod-leak.png)

With prod then dev, the "dev" render kept prod's CPU values, because dev.yaml never mentions `resources`.

![Prod then dev](../evidence/stage4-prod-then-dev-leak.png)

`--set` turned `true` into a boolean, while `--set-string` kept it as text.

![Set vs set string](../evidence/stage4-set-vs-set-string.png)

The boolean rendered fine, but a real install was rejected by the API server.

![Annotation install error](../evidence/stage4-annotation-install-error.png)

Setting `resources.limits=null` removed the limits completely.

![Null deletes limits](../evidence/stage4-null-deletes-limits.png)

### Stage 5

**Failure 1.** Without an environment, `helm template` stopped with my own `required` message.

![Required template](../evidence/stage5-f1-required-template.png)

But `helm lint` only warned and exited with 0, even with `--strict`.

![Lint](../evidence/stage5-f1-lint.png)

![Lint strict](../evidence/stage5-f1-lint-strict.png)

**Failure 2.** Removing `nindent` broke the YAML. `--debug` showed `requests:` stuck at column zero. Restoring the line fixed it.

![Broken line](../evidence/stage5-f2-broken-line.png)

![YAML error](../evidence/stage5-f2-yaml-error.png)

![Debug indent](../evidence/stage5-f2-debug-indent.png)

![F2 fixed](../evidence/stage5-f2-fixed.png)

**Failure 3.** An unquoted `tag: 1.30` silently became `nginx:1.3`.

![Tag float](../evidence/stage5-f3-tag-float.png)

After adding `values.schema.json`, the same mistake failed straight away. The schema also reported every problem at once.

![Schema tag error](../evidence/stage5-schema-tag-error.png)

![Schema multiple errors](../evidence/stage5-schema-multiple-errors.png)

**Failure 4.** I renamed `replicas:` to `replica:`. `helm lint` and `helm install --dry-run=server` both passed, and only `kubectl apply --dry-run=server` caught it.

![Misspelled field](../evidence/stage5-f4-misspelled-field.png)

![Three checks](../evidence/stage5-f4-three-checks.png)

After restoring `replicas:`, all three objects were accepted.

![F4 fixed](../evidence/stage5-f4-fixed.png)

Finally, both environments passed lint and template.

![All envs ok](../evidence/stage5-all-envs-ok.png)

---

## 7. Analysis

**Upgrades can silently lose values.** When I upgraded with only one `--set`, Helm started again from the chart defaults. My replicas and message were gone, and Helm still said success. The safest habit is to keep every value in version controlled files and pass all of them with `-f` on every upgrade.

**History is never rewritten.** Rolling back to revision 1 made revision 4 as a copy. That means I can always see what happened and in what order.

**Helm does not keep secrets secret.** The values are saved inside the release record in the cluster, and I could decode them easily. So passwords should not go in plain values.

**Layering environment files is not switching environments.** Maps merge key by key, so anything one file doesn't mention is taken from the other. Each environment file should be used on its own.

**No single check catches everything.**

| Check | Missing required value | Wrong type | Bad indentation | Unknown field |
| --- | --- | --- | --- | --- |
| `helm lint` | Warning only | Yes, with schema | Yes | No |
| `helm template` | Yes | Yes, with schema | Yes | No |
| `helm install --dry-run=server` | Yes | Yes, with schema | Yes | No |
| `helm template \| kubectl apply --dry-run=server` | Yes | Yes, with schema | Yes | Yes |

So CI should run lint and template for every environment, plus a server dry run.

**Section 3.2.3, release lifecycle.** This section has no commands on the page, so I am covering what I learned from it here:

- `helm upgrade --install` installs if the release doesn't exist and upgrades if it does, so it is safe to run again and again in CI.
- `--rollback-on-failure` (called `--atomic` in Helm 3) rolls back automatically if an upgrade fails.
- Rollback does not restore data in databases or volumes, it only reapplies old manifests.
- Helm 4 uses server side apply for new releases. If someone changes a field Helm owns with kubectl, the next upgrade fails with a conflict instead of silently overwriting it. `--force-conflicts` takes the field back. I saw `apply_method: ssa` in my own release record in Stage 1.

---

## 8. Reflection

The biggest thing I learned is that Helm reporting success does not mean things are correct. The upgrade that lost my values said success, and the misspelled field passed lint and a dry run. I now understand why real pipelines use several checks.

The slow image pull was a real problem I had to fix myself. Loading the image into kind with `kind load docker-image` is the same trick I used in Assignment 1, so it was good to recognise it again.

Writing the chart file by file also helped me understand why real charts have things like the checksum annotation and separate selector labels, instead of just copying the scaffold.

One limitation is that the lab page stops at 3.2.3.2, so I could not do Demo Stages 6 to 8 or sections 3.2.4 to 3.2.8. If they are published later, I will add them to this repo.

---

## 9. Conclusion

I completed Demo Stages 0 to 5 of the Helm notes. I used a published chart through its full lifecycle, built my own `webapp` chart with templates, helpers, a values schema and environment files, and showed how values merge and how different checks catch different errors. All screenshots are in the `evidence/` folder and all chart files are in the repo.

---

## 10. Appendix: Review Questions

### 3.2.1.1 The Problem Helm Solves

**1. Why does `kubectl apply -f` leave an orphaned object behind, and how does a release record avoid this?**
`kubectl apply -f` only creates or updates the objects in the files I give it. It has no memory of what it applied before, so if I delete an object from the file, kubectl never knows to delete it from the cluster. Helm saves the full manifest of every revision in the release record. On upgrade it compares the old and new manifests and deletes objects that are gone, and on uninstall it deletes everything listed.

**2. Where can a password in a values file be read after install?**
In the release record Secret in that namespace, which anyone who can read Secrets can decode, or simply with `helm get values`. It is also in the values file itself, for example in the Git repo.

**3. When is Kustomize more suitable, and when is Helm?**
Kustomize suits environment variants of manifests my own team writes, like dev, staging and prod in Practical 5. Helm suits packaging software for others or installing third party software like Prometheus.

### 3.2.1.2 Helm Architecture

**1. Name of the Secret for revision 7 of release `payments` in namespace `finance`?**
`sh.helm.release.v1.payments.v7`, stored in the `finance` namespace.

**2. Why did removing Tiller improve security?**
Tiller ran inside the cluster with very broad permissions, so anyone who could reach it could deploy anything. Now Helm runs only on the client with the user's own kubeconfig, so normal RBAC rules apply.

**3. Why can release `web` exist in both `team-a` and `team-b`, and which command lists both?**
Release records are stored in the release's own namespace, so names only need to be unique per namespace. `helm list -A` lists both.

### 3.2.1.4 Consuming a Published Chart

**1. After an install set `b=2`, then `helm upgrade r c --set a=1`, what is `b`?**
`b` goes back to the chart default, because an upgrade with any value flag starts from the defaults. `--reuse-values` or `--reset-then-reuse-values` would have kept it.

**2. Why does `helm rollback r 1` give revision 4?**
Helm never rewrites history. A rollback creates a new revision that copies revision 1, so revisions 2 and 3 stay recorded.

**3. Two reasons to prefer an OCI registry over an HTTP chart repo?**
Organisations already run a container registry, so images and charts can live in one system with the same access control. Also no `helm repo add` or `update` is needed, and a digest points to exactly one chart that can't change.

### 3.2.1.5 Chart Structure

**1. Only the default replica count changes from 1 to 2. Which version field changes, if the current chart version is `1.4.2`?**
The chart `version`, because a file in the chart changed, for example to `1.4.3`. `appVersion` stays the same since the app didn't change.

**2. Why does `_helpers.tpl` never produce an object?**
Files in `templates/` whose names start with an underscore are not rendered as objects. They only hold named templates for other files to include.

**3. Purpose of `-0` in `kubeVersion: ">=1.30.0-0"`?**
It lets the constraint also accept pre release Kubernetes builds, whose versions have suffixes. Without it those clusters would be refused.

### 3.2.2.1 The Template Language

**1. Rewrite `{{ quote (upper .Values.name) }}` as a pipeline.**
`{{ .Values.name | upper | quote }}`

**2. Inside `{{- range .Values.hosts }}`, why does `{{ .Release.Name }}` fail?**
Inside `range`, the dot changes to the current list item, which has no `.Release`. The correct expression is `{{ $.Release.Name }}`, because `$` always means the top level context.

**3. Why use `include` rather than `template` for label blocks?**
`include` returns the text, so I can pipe it into `nindent` to indent the labels correctly. `template` writes straight to the output and can't be piped.

### 3.2.2.3 Values Precedence

**1. `helm install r ./c --set a=1 -f x.yaml` with `x.yaml` setting `a: 2`. Final value of `a`?**
`a` is 1, because `--set` always overrides `-f` files no matter where it is on the command line.

**2. Why can `-f dev.yaml -f prod.yaml` give a prod release with a NodePort?**
Maps merge key by key. prod.yaml never sets `service`, so dev's NodePort settings stay. I showed this in Stage 4.

**3. A value `--set` and a YAML file read differently?**
`1.30`. `--set image.tag=1.30` gives the text `1.30`, but `tag: 1.30` in a YAML file becomes the number `1.3`.

### 3.2.2.4 Validating Charts

**1. Why does `helm lint` succeed for a chart that can't render with its defaults?**
Lint treats a missing `required` value as a warning, not an error, even with `--strict`, so it exits with 0. I showed this in Stage 5 Failure 1.

**2. JSON Schema fragment that restricts `service.port` to 80 or 8080.**

```json
"port": { "enum": [80, 8080] }
```

**3. Why isn't a misspelled field caught by `helm template`?**
`helm template` only produces text and checks that it is valid YAML. It doesn't know which fields Kubernetes allows. Only the API server knows that, which is why `kubectl apply --dry-run=server` caught it.