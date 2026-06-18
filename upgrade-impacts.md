# TMC SM 1.4.2 → 1.4.4 Upgrade: Guest-Cluster Impacts & Harbor Modeling

_Companion to `tmc-upgrade-considerations.md` (raw session transcript) and `migration-repurpose.md` (Phase B sequencing). This doc is the visualization-and-mental-model layer: what the upgrade actually does to managed (guest) clusters, where the surprises live, and how to lay out Harbor so the package flow is legible from the Harbor UI._

---

## 1. Two upgrades hide inside "the TMC upgrade"

The installer makes it look like one operation. It is two, with two different blast radii, and they are not separable from the guest's point of view.

| # | What ships | Repo on the **TMC SM** cluster | Repo on **each managed guest** | Who owns the `PackageRepository` on the guest |
|---|---|---|---|---|
| 1 | TMC SM platform (control plane services) | `tanzu-mission-control-packages` in `tmc-local` | _Not present_ — TMC SM platform doesn't run on the guest. | n/a |
| 2 | Tanzu Standard Package Repository (Contour, cert-manager, **fluxcd2**, external-dns, Harbor, Prometheus, Grafana, etc.) | `tanzu-standard` in `tkg-system` | `tanzu-standard` in `tkg-system` | TMC SM — annotated `tanzu.vmware.com/owner: tmc` |

The TMC SM upgrade reaches into **both** repos on the TMC SM cluster *and* reaches into the `tanzu-standard` repo on every managed guest. That second reach is the part that surprises operators, because the installer UI gives no signal that guests are being touched.

```
                ┌────────────────────────────────────────────────┐
                │            TMC SM target cluster               │
                │                                                │
                │  PackageRepository: tanzu-mission-control-     │
                │     packages   (tmc-local)         ─── 1.4.4   │
                │                                                │
                │  PackageRepository: tanzu-standard             │
                │     (tkg-system)                ── v2026.1.21  │
                │                                                │
                │  PackageInstall: tanzu-mission-control         │
                │     (tmc-local)                       1.4.4    │
                └──────────────────┬─────────────────────────────┘
                                   │  TMC pushes new URL into
                                   │  guest's PackageRepository
                                   ▼
        ┌──────────────────────┐ ┌──────────────────────┐  ┌──────────────────────┐
        │  Managed guest #1    │ │  Managed guest #2    │  │  Managed guest #N    │
        │                      │ │                      │  │                      │
        │  PackageRepository:  │ │  PackageRepository:  │  │  PackageRepository:  │
        │   tanzu-standard     │ │   tanzu-standard     │  │   tanzu-standard     │
        │   v2025.6.18 →       │ │   v2025.6.18 →       │  │   v2025.6.18 →       │
        │   v2026.1.21         │ │   v2026.1.21         │  │   v2026.1.21         │
        │   (TMC-owned)        │ │   (TMC-owned)        │  │   (TMC-owned)        │
        └──────────────────────┘ └──────────────────────┘  └──────────────────────┘
```

**Practical consequence:** the `v2026.1.21` bundle must already be in Harbor at the path the guest's `PackageRepository` will resolve, *before* the TMC SM 1.4.2 → 1.4.4 upgrade is initiated. Otherwise every guest enters `ReconcileFailed` (`NOT_FOUND`) the moment TMC pushes the new URL, and stays there until you push the bundle. We hit exactly this in lab1; recipe is in `tmc-upgrade-considerations.md` under "Resolution (2026-06-15)".

---

## 2. What changes on a guest during the TMC SM upgrade

Three classes of change, in roughly the order they materialize:

### 2.1 The Tanzu Standard `PackageRepository` URL is bumped

```yaml
# Before
spec.fetch.imgpkgBundle.image: harbor.lab1.mmtm.ai/tmc-sm/<ecr-path>/packages/standard/repo:v2025.6.18

# After  (written by TMC SM 1.4.4, not by you or kapp-controller)
spec.fetch.imgpkgBundle.image: harbor.lab1.mmtm.ai/tmc-sm/<ecr-path>/packages/standard/repo:v2026.1.21
```

You **cannot** revert this by editing the `PackageRepository` — the annotation `tmc.cloud.vmware.com/managed-tanzu-package-repository: c:<id>/tkg-system/tanzu-standard` makes it TMC-reconciled, and TMC will re-write it back. The only knob you control is **what's at that URL in Harbor.**

### 2.2 kapp-controller reconciles every Tanzu Standard package that's currently `PackageInstall`-ed

Once the new bundle lands, kapp-controller walks every `PackageInstall` in `tkg-system` (and other namespaces) and reconciles them against the new package versions. In this lab the bumped components include — at minimum:

| Package | Why it matters on a guest cluster |
|---|---|
| `fluxcd2.tanzu.vmware.com` | **Highest-risk component.** See §3. |
| `contour.tanzu.vmware.com` | Ingress controller. Bump is generally clean but a Contour minor bump can change Envoy version and reload behavior — watch `LoadBalancer` IPs and active HTTPProxies. |
| `cert-manager.tanzu.vmware.com` | If a guest has its own cert-manager (separate from TMC SM's), watch for CRD storage-version changes. |
| `external-dns.tanzu.vmware.com` | Low risk; only present where deliberately installed. |
| `harbor.tanzu.vmware.com` / `prometheus` / `grafana` / `fluent-bit` | Only relevant where actually installed. |

**Important nuance:** the new bundle ships *catalog* entries for many package versions, but kapp-controller only reconciles the ones with a live `PackageInstall`. The bundle being staged in Harbor is necessary; it does not by itself force unwanted upgrades of every Standard package.

### 2.3 The TMC agent on the guest is bumped

The TMC SM 1.4.4 upgrade rolls a newer cluster-agent into every managed cluster's `vmware-system-tmc` namespace. Health-check this before touching anything else on the guest; if the agent is unhappy, TMC's view of the cluster is stale and downstream operations (policy reconciliation, CD reconciliation) will give misleading signals.

### 2.4 What does **not** change on the guest (in this upgrade)

These are TMC-managed surfaces, but TMC SM 1.4.4 does not appear to mutate them as part of the upgrade itself:

- **Policies & policy assignments.** Cluster-group and cluster-scoped policies (security, image-registry, network, custom) reconcile against the new agent and stay where they were. Schema is additive 1.4.2 → 1.4.4.
- **TMC-managed secrets / secret-exports.** TMC re-asserts them via the agent; values are unchanged.
- **CD enablement state (`continuousdelivery`).** Enable/disable per cluster or cluster-group is preserved.
- **Git repository credentials, GitRepositories, Kustomizations, HelmReleases managed by TMC CD.** Object state is preserved; the controllers underneath them (Flux) may change — that's §3.
- **Cluster-group / workspace memberships.**

Any drift in these surfaces post-upgrade is a finding, not expected — capture it before chasing the Flux-flavored noise in §3.

### 2.5 Component versions: v2025.6.18 vs. v2026.1.21

The Tanzu Standard Repository bundle is a *catalog* — each release contains a multi-year history of every Standard package, multiple versions deep. The version that actually runs on a cluster is whichever the cluster's `PackageInstall.spec.packageRef.versionSelection.constraints` resolves to within the catalog. So a bundle bump on its own does not force any single version onto a cluster; the constraint does.

Two things worth tracking per component:

1. **Catalog max** — the cap on what can be installed from that bundle.
2. **What the cluster's PackageInstall pins** — what kapp-controller will actually reconcile to after the bundle bumps.

**Flux & Contour — catalog max observed in lab1's `pkgr/tanzu-standard.status.template.stderr`:**

| Component | Observed catalog max |
| --- | --- |
| `fluxcd-source-controller` | `v1.1.2_vmware.5-tkg.1` |
| `fluxcd-kustomize-controller` | `v0.32.0_vmware.1-tkg.4` |
| `fluxcd-helm-controller` | `v0.36.2_vmware.1-tkg.1` |
| `contour.tanzu.vmware.com` | `v1.32.0_vmware.1-tkg.1` |

**Attribution caveat:** these were observed during the v2025.6.18 → v2026.1.21 transition window recorded in `tmc-upgrade-considerations.md`. The `template` stage shows the *last successful* reconcile result, so the entries reflect whichever bundle generation last fetched and templated cleanly. Definitive per-bundle labeling requires inspecting each bundle directly (below). Note that an earlier `migration-repurpose.md` draft listed `v1.5.0` / `v1.5.1` / `v1.2.0` Flux versions for v2025.6.18 based on upstream-Flux assumptions; those do not match what the catalog actually shipped and should be treated as superseded by direct inspection.

**Direct bundle inspection — definitive:**

```bash
# Pull either bundle from Harbor and walk it
imgpkg pull -b harbor.lab1.mmtm.ai/<path>:v2025.6.18 -o /tmp/std-v2025.6.18
imgpkg pull -b harbor.lab1.mmtm.ai/<path>:v2026.1.21 -o /tmp/std-v2026.1.21

# Each Package CR shipped (one file per version of each package)
ls /tmp/std-v2026.1.21/packages/fluxcd2.tanzu.vmware.com/
ls /tmp/std-v2026.1.21/packages/contour.tanzu.vmware.com/

# What each PackageInstall on the cluster currently pins to
kubectl get packageinstall -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {.spec.packageRef.refName} @ {.spec.packageRef.versionSelection.constraints}{"\n"}{end}'

# What's actually running, post-reconcile
kubectl get deploy -A -o wide | grep -E 'fluxcd|contour'
```

A wide constraint (`>=0.0.0`, the default in TMC-managed PackageInstalls) means a bundle bump WILL uptake the new catalog max. A tight constraint (`^1.1.2`) holds the cluster at the old version even after the bundle bumps. Worth checking before the upgrade, not after — see also the pre-Phase B inventory commands in §3.2.

---

## 3. Continuous Delivery: the only place a Standard Repo bump can silently break a managed cluster

This is the single risk that matters for clusters that use TMC CD (Flux/Helm/Kustomize). It exists because of one sentence in the TMC SM enable-CD docs:

> If Flux CRDs are present, TMC uses the currently installed instance rather than installing a new one. If the CRDs are not present, TMC installs the Flux source controller and Kustomize controller and subsequently manages their lifecycles.

Translation: **on any guest where the Tanzu Standard `fluxcd2` package is installed, kapp-controller — not TMC — owns Flux's lifecycle.** TMC's CD piggybacks on whatever CRDs `fluxcd2` provides.

Concretely, the failure shape on Phase B:

```
TMC SM upgrade kicks off
       │
       ▼
PackageRepository tanzu-standard bumped on guest  (v2025.6.18 → v2026.1.21)
       │
       ▼
kapp-controller reconciles fluxcd2 PackageInstall to new version
       │
       ▼
New fluxcd-* controller images roll;
   CRD storage version for *.toolkit.fluxcd.io may change
       │
       ▼
TMC CD-managed GitRepository/Kustomization/HelmRelease reconciliations
   start failing with:
     "failed to get API group resources:
      unable to retrieve the complete list of server APIs:
      helm.toolkit.fluxcd.io/v2"
```

**The TMC SM upgrade itself succeeds. The guest cluster looks `HEALTHY` in TMC. CD silently stops reconciling.** This is the failure mode `migration-repurpose.md:188` calls "the main break vector" and it is the reason Phase B verification has Flux-specific steps.

### 3.1 Decision tree per guest cluster

```
For each managed guest cluster:

    Is TMC CD enabled on this cluster?
       ├─ No  ─►  Standard Repo bump is low-risk here.
       │         Confirm Contour/other-package behavior post-upgrade, done.
       │
       └─ Yes ─►  Is `fluxcd2` PackageInstall present in tkg-system?
                    │
                    ├─ No  ─►  TMC owns Flux. Standard Repo bump cannot
                    │          mutate Flux versions here. Safe.
                    │
                    └─ Yes ─►  Dual-ownership. Standard Repo bump WILL
                               re-template Flux. Three live options:
                                 (a) Pre-stage: disable TMC CD on this
                                     cluster before Phase B; re-enable
                                     after. TMC re-owns Flux on re-enable.
                                     Brief CD outage. Lowest risk.
                                 (b) Co-resident: leave both. Treat the
                                     first CD reconcile after Phase B as
                                     the canary; force a `flux reconcile
                                     kustomization …` and tail controller
                                     logs. Halt on API-group errors.
                                 (c) Remove `fluxcd2` PackageInstall
                                     entirely (KB 375864 cleanup applies).
                                     Long-term clean, short-term churn.
```

### 3.2 Pre-Phase B inventory per CD-enabled guest

Run these against each guest before the TMC SM upgrade and save the output:

```bash
# Is fluxcd2 installed via the Tanzu Standard repo?
tanzu package installed list -A | grep -i flux

# Current controller image tags
kubectl get deploy -A -l app.kubernetes.io/part-of=flux -o wide

# CRD storage versions — these are what changes underneath you
for crd in helmreleases.helm.toolkit.fluxcd.io \
           kustomizations.kustomize.toolkit.fluxcd.io \
           gitrepositories.source.toolkit.fluxcd.io \
           helmrepositories.source.toolkit.fluxcd.io \
           ocirepositories.source.toolkit.fluxcd.io \
           helmcharts.source.toolkit.fluxcd.io \
           buckets.source.toolkit.fluxcd.io; do
  echo "=== $crd ==="
  kubectl get crd $crd -o jsonpath='{.spec.versions[?(@.storage)].name}{"\n"}' 2>/dev/null
done

# Non-TMC Flux objects (anything you didn't expect)
kubectl get gitrepositories,kustomizations,helmreleases -A
```

This is the "did anything actually change" baseline. After the upgrade, diff it.

### 3.3 Post-upgrade canary per CD-enabled guest

```bash
# Force a real CD reconciliation against a known-good GitRepository
flux reconcile source git <gr-name> -n <ns>
flux reconcile kustomization <ks-name> -n <ns>

# Tail Flux controllers during the reconcile
kubectl -n <flux-ns> logs -l app=source-controller    --tail=200 -f
kubectl -n <flux-ns> logs -l app=kustomize-controller --tail=200 -f
kubectl -n <flux-ns> logs -l app=helm-controller      --tail=200 -f
```

**Halt criterion:** any `failed to get API group resources … helm.toolkit.fluxcd.io/v2` (or any `*.toolkit.fluxcd.io` group). That's the documented API-group resolution error from [KB 369984]; once you see it, no further Phase C migrations until CD is restored on the affected cluster.

---

## 4. Contour on managed clusters

Contour is the other place where a Standard Repo bump can have user-visible impact, even though it is far less prone to silent breakage than Flux.

If Contour is installed on a guest via the Tanzu Standard `contour.tanzu.vmware.com` package, the v2026.1.21 bundle will reconcile it to a newer Contour/Envoy pair. Things to watch:

- **`HTTPProxy` schema additions.** Bumps are additive in 1.x; older HTTPProxies remain valid.
- **Envoy pod roll.** Brief LB connection churn during the roll; pin to a maintenance window if the cluster fronts real traffic.
- **`LoadBalancer` Service IP retention.** Service object isn't touched, but if the underlying cloud-provider integration was reconciled in the same window, double-check IPs.
- **Ingress class.** Unchanged unless you've customized it.

If Contour was installed by TMC's extensions catalog (not via the Standard Repo `PackageInstall`), the Standard Repo bump does not move it — TMC owns its lifecycle. Inventory with `tanzu package installed list -A` to know which case you're in.

---

## 5. Per-cluster runbook (what to actually do)

Compressed to the minimum useful actions. Expand details from §3 / `migration-repurpose.md` as needed.

### 5.1 Before Phase B (TMC SM 1.4.2 → 1.4.4)

1. **Stage `tanzu-standard:v2026.1.21` into Harbor** at the path mirroring the AWS ECR layout TMC SM stamps into guest `PackageRepository` objects:
   ```
   <harbor>/<project>/<ecr-mirror-path>/packages/standard/repo:v2026.1.21
   ```
   Example from lab1: `harbor.lab1.mmtm.ai/tmc-sm/498533941640.dkr.ecr.us-west-2.amazonaws.com/packages/standard/repo:v2026.1.21`. Recipe in `tmc-upgrade-considerations.md` "Resolution (2026-06-15)". Validate the artifact in Harbor UI before proceeding.
2. **Inventory every CD-enabled guest** per §3.2. Save the output.
3. **Choose the Flux strategy per guest** per §3.1. Disable TMC CD on dual-ownership clusters now if you picked option (a).
4. **Read the v2026.1.21 release notes** for `fluxcd2`, `contour`, `cert-manager` deltas and package renames. Hold Phase B if notes aren't available.

### 5.2 During Phase B

5. Run the TMC SM upgrade per the [official UI flow][tmc-upgrade-ui].
6. Verify TMC SM cluster: `tanzu-mission-control-packages` at 1.4.4, `tanzu-standard` at v2026.1.21, all TMC SM pods on new images, UI shows 1.4.4.
7. Verify each guest cluster's `tanzu-standard` `PackageRepository`:
   ```bash
   kubectl -n tkg-system get pkgr tanzu-standard -o yaml | grep -E 'image|Reconcile'
   ```
   Expect `ReconcileSucceeded` and the `:v2026.1.21` URL. If not, see `tmc-upgrade-considerations.md` failure analysis.

### 5.3 After Phase B, per CD-enabled guest

8. Diff §3.2 inventory against current state. Note any controller image changes and CRD storage-version changes.
9. Run §3.3 canary. Halt on API-group errors.
10. If option (a) chosen, re-enable TMC CD now and confirm re-bootstrapped state.
11. Verify Contour-fronted ingress on the guest still serves traffic.
12. Verify TMC policies, secrets, and CD records are intact via `tmc` CLI and UI.

---

## 6. Modeling Harbor repositories for visualization

Two structural facts about how `imgpkg copy` lands bundles in Harbor constrain what any layout change can achieve. Cover those first, then the layout options.

### 6.1 Why Harbor looks flat under each bundle path

Each bundle path holds:

- **One human-tagged manifest per bundle version** — e.g. `:v2025.6.18`, `:v2026.1.21`, `:1.4.4`.
- **All referenced images as untagged `@sha256:…` siblings** in the same repo — by design, because `imgpkg copy` co-locates referenced images so kapp-controller (on the TMC SM cluster or on the guest) can resolve every digest the bundle records.

OCI has no subdirectories under a repo, so the Harbor UI cannot show a package-by-package structure. That structure lives *inside* the bundle, in `.imgpkg/images.yml`:

```bash
imgpkg pull -b harbor.lab1.mmtm.ai/<path>:v2026.1.21 -o /tmp/bundle
cat /tmp/bundle/.imgpkg/images.yml
```

That file maps each referenced image's original registry path (`projects.registry.vmware.com/tkg/packages/standard/contour:v1.32.0_…`) to its digest in Harbor — the "hierarchy" you'd want, just not anywhere Harbor can render it. Re-projecting to Option B/C below does not change this; SHA-flatness is per-bundle, not per-project.

### 6.2 Two repo families coexist after install

| Harbor path | Tags | What it is | Consumed by |
|---|---|---|---|
| `tmc-sm/<ecr-mirror>/packages/standard/repo` | `:v2025.6.18`, `:v2026.1.21` | Tanzu Standard Package Repository (Contour, cert-manager, fluxcd2, external-dns, Harbor, Prometheus, Grafana, …) | TMC SM cluster **and** every managed guest |
| `tmc-sm/package-repository` | `:1.4.2`, `:1.4.4` | TMC SM platform's own `tanzu-mission-control-packages` bundle | TMC SM cluster only |

A 1.4.2 → 1.4.4 upgrade adds one new tag in each (`:v2026.1.21`, `:1.4.4`); old tags coexist, nothing is overwritten. The layout options below choose where these two repos *sit*, not what's inside them.

The current path under both is a literal copy of the AWS ECR mirror path TMC SM was originally installed against — e.g. `tmc-sm/498533941640.dkr.ecr.us-west-2.amazonaws.com/packages/standard/repo`. That works mechanically but reads poorly in the Harbor UI: project named for an AWS account number, "standard repo" vs. "TMC SM platform" split invisible.

### 6.3 Option A — keep the ECR-shaped path (status quo)

```
harbor.lab1.mmtm.ai/
  tmc-sm/
    498533941640.dkr.ecr.us-west-2.amazonaws.com/
      packages/
        standard/repo:v2025.6.18
        standard/repo:v2026.1.21
        tanzu-mission-control/...
```

- **Pros:** matches exactly what TMC SM stamps into guest `PackageRepository.spec.fetch.imgpkgBundle.image`. Zero rewrite needed.
- **Cons:** UI confuses operators (why is there an AWS account number in *our* Harbor?). Versions of the standard repo are visually flat. Hard to spot "we have v2025.6.18 staged but not v2026.1.21" without drilling in.

### 6.4 Option B — re-project by semantic role

Create one Harbor *project* per semantic role, with the on-disk path under that project mirroring the source-of-truth registry:

```
harbor.lab1.mmtm.ai/
  tmc-standard-packages/                            # ← project
    packages/standard/repo:v2025.6.18
    packages/standard/repo:v2026.1.21
  tmc-sm-platform/                                  # ← project
    packages/tanzu-mission-control/...
    packages/tanzu-mission-control-packages/repo:1.4.2
    packages/tanzu-mission-control-packages/repo:1.4.4
  tmc-extensions/                                   # ← project
    packages/contour-extension/...
    packages/cd-extension/...
```

- **Pros:** Harbor's project list immediately shows the three independent lifecycles. RBAC can be set per project (e.g. "platform ops can push to `tmc-sm-platform`, app teams can read `tmc-standard-packages`"). Quotas and replication policies become per-project.
- **Cons:** **TMC SM stamps the path into guest `PackageRepository`s itself** — re-projecting requires either (i) waiting until the next major TMC SM release where the path is configurable, or (ii) running both paths during a transition and weaning guests off the old one. Not a casual move.

### 6.5 Option C — sidecar metadata project (additive, low-risk)

Keep the ECR-shaped path as the runtime path, **and** add a parallel Harbor project (`tmc-catalog` or similar) that is *empty of artifacts* but holds Harbor labels, retention policies, and Harbor robot-account scoping that *describes* the three semantic groups. Use Harbor's "tag immutability" and "tag retention" rules at the runtime path to encode the same information operators want at a glance:

- Immutability rules on `**/standard/repo:v*` so v2025.6.18 / v2026.1.21 can't be silently overwritten.
- Retention policy: keep the two most recent standard-repo tags, retain TMC SM platform tags indefinitely.
- Robot accounts named after the role (`tmc-sm-pull`, `tanzu-standard-pull`) even though they point at paths inside the ECR-shaped tree.

- **Pros:** Zero risk to the live `PackageRepository` URLs. Operator visibility improves without a path migration.
- **Cons:** Indirection — the project structure doesn't itself show the semantic split; operators need to know which Harbor labels and robot accounts encode the model.

### 6.6 Recommendation

For a lab, **Option A + Harbor labels + an `IMPORTS.md` cheat-sheet** is enough. For prod, **Option C** is the right pragmatic choice — keeps the live URLs stable, encodes the model in Harbor metadata, and stages Option B for the next major TMC SM release when the path becomes configurable.

Either way, the things to be able to see at a glance in Harbor are:

1. **All currently-required standard-repo tags** in one place (e.g. `v2025.6.18` and `v2026.1.21` side-by-side, with both pinned by retention).
2. **Which TMC SM platform version corresponds to which standard-repo version** — encode this in the artifact-level labels (`tmc-sm: "1.4.4"`, `tanzu-standard: "v2026.1.21"`).
3. **Whether a given tag has been actually pulled by a guest** — Harbor's pull-count column, surfaced per-tag, is the cheap proxy for "is anything still on v2025.6.18".

---

## 7. References

### TMC Self-Managed (Broadcom techdocs)
- [Upgrading TMC Self-Managed — UI flow][tmc-upgrade-ui]
- [TMC SM 1.4 documentation root](https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-mission-control-self-managed/1-4/tmc-self-managed-documentation.html)
- [Stage images in Harbor (prep step for upgrade)](https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-mission-control-self-managed/1-4/tmc-self-managed-documentation/install-and-run-tmc-self-managed/upgrading-tmc-self-managed.html#stage-images)
- [Enable Continuous Delivery for a cluster or cluster-group](https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-mission-control-self-managed/1-4/tmc-self-managed-documentation/using-tmc/managing-cluster-resources-with-continuous-delivery/enable-continuous-delivery-for-a-cluster-or-cluster-group.html) — source of the "if Flux CRDs are present, TMC uses the currently installed instance" behavior
- TMC SM 1.4.3 release notes — Standard Repository rebrand known-issue (duplicate Available Packages entries)
- TMC SM 1.4.4 release notes — confirm Flux version envelope and standard-repo rename surface area before Phase B

### Tanzu Standard Package Repository
- [Tanzu Standard Package Repository overview (TKG techdocs)](https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-kubernetes-grid/2-5/tkg/workload-packages-index.html) — package list, supported matrix
- Tanzu Standard Repository release notes for `v2026.1.21` — confirms component versions and rename surface area
- [Carvel kapp-controller `PackageRepository` reference](https://carvel.dev/kapp-controller/docs/develop/packaging/) — semantics of `imgpkgBundle.image`, reconcile cadence, annotations

### Carvel / imgpkg
- [`imgpkg copy` usage](https://carvel.dev/imgpkg/docs/develop/copy/) — the right command for moving bundles between registries (note: not `imgpkg push`)
- [Air-gapped relocation with `--to-tar` / `--tar`](https://carvel.dev/imgpkg/docs/develop/air-gapped-workflow/)

### FluxCD
- [Flux upgrade compatibility statement](https://fluxcd.io/flux/installation/upgrade/) — "any v2.x to any other v2.x"
- [Flux CRD reference](https://fluxcd.io/flux/components/) — controllers, CRDs, and their API groups

### Broadcom Knowledge Base
- [KB 369984 — "Enable Continuous Delivery for a TKGs cluster" / API-group resolution error](https://knowledge.broadcom.com/external/article/369984/enable-continuous-delivery-for-a-tkgs-cl.html) — documents the `failed to get API group resources` failure mode
- [KB 375864 — How to remove the Flux CD package after disable](https://knowledge.broadcom.com/external/article/375864/how-to-remove-the-flux-cd-package-after.html) — orphan `tanzu-fluxcd-packageinstalls` cleanup after `continuousdelivery disable`

### Harbor
- [Harbor projects & RBAC](https://goharbor.io/docs/latest/working-with-projects/) — basis for Option B/C in §6
- [Tag retention policies](https://goharbor.io/docs/latest/working-with-projects/working-with-images/create-tag-retention-rules/) — useful for pinning v2025.6.18 / v2026.1.21 side-by-side
- [Tag immutability rules](https://goharbor.io/docs/latest/working-with-projects/working-with-images/create-tag-immutability-rules/) — prevent silent overwrite of staged standard-repo tags

### Companion docs in this repo
- `migration-repurpose.md` — Phase A/B/C sequencing, FluxCD dual-ownership section (§ "FluxCD and Standard Repository upgrade"), Pre-Phase B inventory
- `tmc-upgrade-considerations.md` — raw session transcript with the lab1 reproduction, validated `imgpkg copy` recipe, and resolution log

[tmc-upgrade-ui]: https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-mission-control-self-managed/1-4/tmc-self-managed-documentation/install-and-run-tmc-self-managed/upgrading-tmc-self-managed.html#upgrade-tmc-ui
