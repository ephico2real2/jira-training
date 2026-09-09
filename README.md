# Group Sync Platform — JIRA Epic and Stories

Ready-to-paste JIRA content for the three repositories that make up the group-sync platform:
one **epic** linking them, **three stories** for the Helm chart design, and **nine stories** for
onboarding each component into the ArgoCD ApplicationSet — three per repository.

Copy field-by-field into JIRA. Replace the project key `GSP` with your real one.

---

## 1. The three repositories

Verified against the working copies in `~/gitRepos` on 2026-09-09.

| Repo | Chart | Chart ver | appVersion | Templates | Legacy manual manifest |
|---|---|---|---|---|---|
| `group-sync-operator-helm-chart` | `group-sync-operator-helm` | 0.13.1 | 1.1 | 19 | `argocd-application.yaml` present |
| `group-sync-dashboard` | `group-sync-dashboard` | 0.20.0 | 0.18.0 | 27 | none |
| `openshift-rbac-automation` | `openshift-rbac-automation` | 0.22.0 | 1.2.6 | 16 (+2 CRDs) | `argocd-application.yaml` present |

### Delivery model

Applications are **not** deployed from hand-authored `argocd-application.yaml` files. Delivery runs
through the day-2 operations automation: an **ApplicationSet** generates the ArgoCD Applications,
fed by a **`config.json` feeder** carrying one entry per deployed application. Onboarding a component
means adding it to the ApplicationSet and creating its feeder entry — not writing an Application manifest.

The two `argocd-application.yaml` files still sitting in the repos predate this and are not the
deployment path. They are useful only as a record of the sync semantics each chart needs; retiring
them is tracked in the stories so nobody deploys from them by mistake.

### How the three fit together

The three are one pipeline, which is why they belong under a single epic:

1. **`group-sync-operator-helm-chart`** deploys the `redhat-cop` group-sync-operator, which pulls
   groups out of LDAP and materialises them as OpenShift `Group` objects.
2. **`openshift-rbac-automation`** deploys the namespace-configuration-operator (NCO), which binds
   those synced groups to roles — turning group membership into actual cluster access.
3. **`group-sync-dashboard`** is read-only observability over both: what synced, when, who ended up
   with which access, and reporting on top of it.

Group identity flows left to right; the dashboard observes the whole chain. A break in step 1
silently changes access in step 2, and step 3 is how anyone finds out.

---

## 2. EPIC

> **Project:** `GSP`  **Issue type:** Epic

**Epic Name:** `Group Sync Platform — unified Helm packaging and GitOps onboarding`

**Summary:** `Package the three group-sync platform components as coordinated Helm charts and onboard each into the ArgoCD ApplicationSet`

**Description:**

```
h2. Context

The group-sync platform is three repositories that operate as one system:

* group-sync-operator-helm-chart — deploys the redhat-cop group-sync-operator, which syncs
  LDAP groups into OpenShift Group objects.
* openshift-rbac-automation — deploys the namespace-configuration-operator, which binds those
  synced groups to RBAC roles.
* group-sync-dashboard — read-only observability and reporting across both.

Group identity flows operator -> RBAC automation -> dashboard. The three are installed together,
upgraded together, and fail together, but today they are packaged and delivered independently.

Deployment runs through the day-2 operations automation: an ApplicationSet generates the ArgoCD
Applications from a config.json feeder holding one entry per deployed application. None of the three
components is onboarded to it. Two of them still carry stale hand-written argocd-application.yaml
files from before that model, which are not the deployment path and should not be mistaken for it.

h2. Goal

Every component installs from a Helm chart that is documented, versioned and CI-tested, and every
component reaches a cluster through the ApplicationSet and its feeder entry — no hand-authored
Application manifests, and no manual step between merge and running workload.

h2. Scope

* Helm chart design, values contract and install documentation for all three components (3 stories)
* ApplicationSet onboarding for all three, split per component into the feeder entry, the sync
  semantics, and validation on a cluster (9 stories)

h2. Out of scope

* Application feature work in any of the three repos
* Changes to the day-2 automation or the feeder mechanism itself — this epic consumes it as-is
* Multi-cluster fleet rollout beyond the target cluster

h2. Definition of done

* Three charts install hands-free from a documented command, each with a values contract and CI lint/template coverage.
* Three components are onboarded to the ApplicationSet with their feeder entries, syncing Healthy/Synced on a real cluster.
* The stale argocd-application.yaml files are retired.
* A documented install order brings the platform up on an empty cluster through GitOps alone.
```

**Labels:** `group-sync` `helm` `gitops` `argocd` `applicationset` `openshift` `platform`

**Components:** `group-sync-operator`, `rbac-automation`, `dashboard`

---

## 3. Stories — Helm chart design

### GSP-H1 · group-sync-operator-helm chart

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 5

**Summary:** `Design and document the Helm chart for the group-sync operator`

**Description:**

```
The chart group-sync-operator-helm (currently 0.13.1, appVersion 1.1, 19 templates) deploys the
redhat-cop group-sync-operator and its GroupSync CRs for LDAP synchronisation.

Settle and document the chart's contract so it can be consumed by the ApplicationSet without
tribal knowledge:

* CRD ownership. The GroupSync CRD arrives from the catalog via the OLM Subscription; OLM is its
  sole owner and the chart must not manage it. crd.install stays the bootstrap-only escape hatch.
* Install ordering for the OLM path — Subscription, then the approver, then the CRs that depend on
  the CRD existing.
* The values contract: which values are required, which are discovered from the cluster, and what
  the chart does when a required value is absent.
* The dual behaviour of groupSync.url, which is the chart's sharpest edge: under a direct helm
  install it is derived from the cluster's OAuth CR, but under any offline render it cannot be, and
  the chart deliberately fails rather than emitting a CR with an empty url. Document it as a rule
  per deployment path.
* LDAPS trust: the trustedCA flow, and which ConfigMap the cluster fills rather than the chart.
```

**Acceptance criteria:**

- [ ] `helm lint` and `helm template` pass on the chart with default values and with each documented example values file.
- [ ] The values contract is documented in the chart README: every required value, every optional value, its default, and the failure message when a required value is missing.
- [ ] The `groupSync.url` rule is documented as a table by deployment path, including the exact render-time failure under an offline render.
- [ ] The chart renders no CustomResourceDefinition by default; CRD ownership is stated explicitly in the README.
- [ ] Install order is documented, and any hook or sync-wave annotations needed to enforce it are in the templates.
- [ ] A hands-free install from a single documented command produces a healthy operator on a clean cluster.
- [ ] The LDAPS/trusted-CA flow is documented, including which field the cluster fills in.

---

### GSP-H2 · openshift-rbac-automation chart

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 5

**Summary:** `Design and document the Helm chart for the RBAC automation (NCO)`

**Description:**

```
The chart openshift-rbac-automation (currently 0.22.0, appVersion 1.2.6, 16 templates and 2 CRDs)
deploys the namespace-configuration-operator and the GroupConfig/NamespaceConfig objects that turn
synced LDAP groups into role bindings.

This is the chart with genuine CRDs in the tree, so the design work centres on ownership and on what
happens when configuration narrows:

* CRD delivery: what lives in crds/, what that means for helm upgrade (which never updates crds/),
  and how a CRD change is rolled out.
* The narrowing problem: reducing the set of managed objects does not delete the objects already
  created. Displaced objects need a deliberate, documented cleanup path.
* Guarded renders: a configuration that legitimately produces an empty render should be quiet, not broken.
* The values contract for the group-to-role mapping — the security-relevant surface of the platform.
```

**Acceptance criteria:**

- [ ] `helm lint` and `helm template` pass with default values and with the example values files.
- [ ] CRD delivery and upgrade behaviour is documented, including the `helm upgrade` limitation on `crds/`.
- [ ] The narrowing behaviour is documented: what happens to objects no longer in the config, and the operator-facing procedure to clean them up.
- [ ] The group-to-role values contract is documented with a worked example mapping one LDAP group to one role in one namespace.
- [ ] A render with an empty managed set produces valid output and a clear message, not an error.
- [ ] Chart version bumps follow a stated semver policy tied to appVersion changes.

---

### GSP-H3 · group-sync-dashboard chart

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 5

**Summary:** `Design and document the Helm chart for the group sync dashboard`

**Description:**

```
The chart group-sync-dashboard (currently 0.20.0, appVersion 0.18.0, 27 templates — the largest of
the three) deploys read-only observability over the operator and the RBAC automation.

Design work:

* The image/appVersion chain: how image tags resolve from Chart.appVersion, and the guard that keeps
  a deployed image from disagreeing with the version the chart advertises.
* Exposure: the Route as the default ingress path, so a GitOps consumer needs no extra values to get
  a reachable dashboard.
* Access-control values: the reader tiers, and which are cookie-only versus token-usable — that
  difference decides how a deployment can actually be verified.
* The reporting service and its own image, which must be published and version-locked alongside the
  dashboard image rather than drifting.
* Optional subsystems (monitoring, alerts): default-on or default-off, stated per switch with the reason.
```

**Acceptance criteria:**

- [ ] `helm lint` and `helm template` pass with default values and every example values file.
- [ ] A default install exposes a reachable dashboard with no extra values supplied.
- [ ] The appVersion-to-image-tag resolution is documented, and a test holds them together.
- [ ] Every optional subsystem's default state is documented in the chart README with the reason.
- [ ] The access-control tiers are documented, including how to verify each one against a running instance.
- [ ] Both images the chart references are published at the chart's appVersion before the chart is released.

---

## 4. Stories — ApplicationSet onboarding

Nine stories, three per component, in the same shape each time:

| Phase | What it delivers |
|---|---|
| **Feeder entry** | The component is registered in the ApplicationSet and has its `config.json` entry. The generated Application renders without error. |
| **Sync semantics** | Prune, selfHeal, ignored differences, CRD handling and ordering are chosen deliberately for that component and recorded in its entry. |
| **Validation** | The generated Application syncs Healthy on a real cluster, the component does its job, and the stale manifest is retired. |

No `argocd-application.yaml` is authored anywhere — the Application is generated. The feeder entry's
shape follows the automation's existing conventions; these stories say what each component needs
from its entry, not how the entry is structured.

---

### 4.1 · group-sync operator

#### GSP-G1 · Feeder entry for the group-sync operator

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 3
> **Depends on:** GSP-H1

**Summary:** `Register the group-sync operator in the ApplicationSet and create its feeder entry`

**Description:**

```
Add the group-sync operator to the ApplicationSet and create its config.json feeder entry so the
day-2 automation generates and manages its Application.

Two things this entry must get right, both of which block rendering:

* groupSync.url must be set explicitly in the per-cluster values. ArgoCD's repo-server renders
  offline, so the chart's discovery of the url from the cluster's OAuth CR resolves to nothing there,
  and the chart's guard refuses to emit a GroupSync with an empty url — the Application fails at
  render with a ComparisonError. This is the chart's documented rule for the GitOps path, not an
  edge case, so the values source has to exist and carry the url before the entry goes live.
* The chart version the entry pins. The stale manifest in the repo pins a range five minor versions
  behind the current chart, which is the failure mode a reviewed, recorded pin prevents.
```

**Acceptance criteria:**

- [ ] The operator is registered in the ApplicationSet and has a feeder entry in `config.json`.
- [ ] The per-cluster values source exists and supplies `groupSync.url`.
- [ ] The generated Application renders without error — no `ComparisonError`, nothing partially applied.
- [ ] The pinned chart version is current, and the bump policy is recorded with the entry.
- [ ] The entry is reviewed by someone other than its author before it goes live.

---

#### GSP-G2 · Sync semantics for the group-sync operator

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 5
> **Depends on:** GSP-G1

**Summary:** `Set the sync semantics for the generated group-sync operator Application`

**Description:**

```
Choose and record the sync behaviour the operator chart requires. This is the most demanding of the
three components, because two of its behaviours can destroy cluster state if left on defaults:

* CRD protection. The GroupSync CRD is owned by OLM and delivered via the Subscription. ArgoCD
  renders Helm sources with --include-crds, so CRDs can appear where a plain render would not emit
  them; with prune and selfHeal active, a sync could delete a CRD OLM owns, cascading to every
  GroupSync CR on the cluster. The generated Application must not manage that CRD.
* Injected-CA drift. The trusted-CA ConfigMap is shipped empty and filled by OpenShift's network
  operator. Without ignored differences, ArgoCD sees permanent drift and selfHeal empties it on
  every sync, breaking LDAPS verification until the operator refills it. Ignoring the difference is
  only half the fix — the sync options have to respect that instruction too.
* Install ordering. Subscription, then the approver, then the CRs that need the CRD to exist. The
  chart's own ordering has to survive ArgoCD's sync-wave handling rather than fight it.
* Prune and selfHeal, chosen deliberately for an operator deployment rather than inherited.
```

**Acceptance criteria:**

- [ ] The generated Application does not manage the OLM-owned CRD, and a sync cannot delete it — verified by attempting one.
- [ ] The injected trusted-CA ConfigMap does not show as drifted, and selfHeal does not empty it.
- [ ] Install ordering holds under a sync from empty: the Subscription settles before the CRs are applied.
- [ ] Prune and selfHeal are set deliberately, with the reasoning recorded alongside the entry.
- [ ] Each semantic choice is documented where the next person will look — not only in the entry.

---

#### GSP-G3 · Validate the group-sync operator deployment

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 3
> **Depends on:** GSP-G2

**Summary:** `Validate the generated group-sync operator Application on a cluster and retire the stale manifest`

**Description:**

```
Prove the generated Application works against a real cluster, then remove the hand-written manifest
so the generated one is unambiguously the only deployment path.

Validation is not "the Application went green" — it is that the operator does its job: LDAP groups
arrive as OpenShift Group objects, which is the input the whole rest of the platform depends on.
```

**Acceptance criteria:**

- [ ] The Application reaches Healthy/Synced on the target cluster.
- [ ] LDAP groups materialise as OpenShift `Group` objects carrying the operator's sync-provider label.
- [ ] A deliberately introduced drift is corrected by selfHeal.
- [ ] A resync from scratch on a clean namespace reproduces the result.
- [ ] The stale `argocd-application.yaml` is removed from the repo.

---

### 4.2 · RBAC automation (NCO)

#### GSP-G4 · Feeder entry for the RBAC automation

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 2
> **Depends on:** GSP-H2

**Summary:** `Register the RBAC automation in the ApplicationSet and create its feeder entry`

**Description:**

```
Add the namespace-configuration-operator deployment to the ApplicationSet and create its config.json
feeder entry.

The trap here is the values file. The stale manifest points at a CRC values file — a local
development target — and the feeder entry is exactly where that mistake would persist unnoticed into
production. The entry names the values the generated Application renders with, so it needs the
production values and a pinned chart version chosen on purpose.
```

**Acceptance criteria:**

- [ ] The component is registered in the ApplicationSet and has a feeder entry in `config.json`.
- [ ] The entry references the production values; no CRC-only values on the production path.
- [ ] The generated Application renders without error.
- [ ] The pinned chart version is current, with the bump policy recorded.
- [ ] The entry is reviewed before it goes live — this component grants cluster access.

---

#### GSP-G5 · Sync semantics for the RBAC automation

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 3
> **Depends on:** GSP-G4

**Summary:** `Set the sync semantics for the generated RBAC automation Application`

**Description:**

```
Choose and record the sync behaviour for a component whose output is cluster access.

* CRD ownership. The chart ships 2 CRDs and the stale manifest skipped them, which implies something
  other than ArgoCD installs them. Name that path, or change the decision — a clean-cluster install
  must produce the CRDs by a route someone can point at.
* Prune against narrowing. ArgoCD prune deletes what the render drops, which interacts directly with
  the objects NCO leaves behind when its configuration narrows (see GSP-H2). The interaction of those
  two behaviours decides whether stale role bindings survive a config change, so it is a deliberate
  decision rather than a default to inherit.
* Sync policy chosen in the knowledge that a bad sync here grants or revokes access.
```

**Acceptance criteria:**

- [ ] CRD installation ownership is documented, and a clean-cluster install produces the CRDs by the named route.
- [ ] Prune behaviour is deliberately chosen and documented against the narrowing case.
- [ ] The operator-facing cleanup procedure for displaced objects is written down and linked from the entry.
- [ ] Prune and selfHeal are set deliberately, with reasoning recorded.
- [ ] A narrowing change is rehearsed in a non-production namespace and behaves as documented.

---

#### GSP-G6 · Validate the RBAC automation deployment

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 3
> **Depends on:** GSP-G5, GSP-G3

**Summary:** `Validate the generated RBAC automation Application on a cluster and retire the stale manifest`

**Description:**

```
Prove the generated Application works against a real cluster and actually binds access, then remove
the hand-written manifest.

This story depends on GSP-G3 as well as its own predecessor: NCO binds groups to roles, so validating
it needs real groups, which only exist once the operator is deployed and syncing.
```

**Acceptance criteria:**

- [ ] The Application reaches Healthy/Synced on the target cluster.
- [ ] An LDAP group synced by the operator receives its intended role binding — verified on the cluster, not inferred from the render.
- [ ] A user in that group can perform an action the role grants, and a user outside it cannot.
- [ ] A deliberately introduced drift is corrected by selfHeal.
- [ ] The stale `argocd-application.yaml` is removed from the repo.

---

### 4.3 · Dashboard

#### GSP-G7 · Feeder entry for the dashboard

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 3
> **Depends on:** GSP-H3

**Summary:** `Register the dashboard in the ApplicationSet and create its feeder entry`

**Description:**

```
Add the dashboard to the ApplicationSet and create its config.json feeder entry. This component has
never had any GitOps entry point — there is no prior manifest to adapt, so every choice is being made
for the first time rather than inherited from something that already half-worked.

The precondition specific to this component is image publication: the chart references two images,
the dashboard and its reporting service, and both must exist at the chart's appVersion before a sync
can succeed. A sync against unpublished images fails at pull time, which looks like a deployment
problem and is actually a release problem. Verify publication rather than assuming it.
```

**Acceptance criteria:**

- [ ] The dashboard is registered in the ApplicationSet and has a feeder entry in `config.json`.
- [ ] Both referenced images are confirmed published at the chart's appVersion before the entry goes live.
- [ ] The generated Application renders without error.
- [ ] The pinned chart version is current, with the bump policy recorded.

---

#### GSP-G8 · Sync semantics for the dashboard

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 2
> **Depends on:** GSP-G7

**Summary:** `Set the sync semantics for the generated dashboard Application`

**Description:**

```
Choose and record the sync behaviour for the platform's read-only component.

* Prune and selfHeal for a read-only workload — the least dangerous of the three to sync
  aggressively, which is worth stating explicitly rather than leaving as an accident.
* Whether the reporting service holds persistent state, and if so what that means for prune: state
  that can be destroyed by a sync changes the answer.
* The chart defaults its exposure on for GitOps, so the generated Application should need no extra
  values for the dashboard to be reachable. Confirm that rather than trusting the default.
```

**Acceptance criteria:**

- [ ] Prune and selfHeal are set deliberately, with the read-only reasoning recorded.
- [ ] The reporting service's state handling is established, and prune behaviour matches it.
- [ ] A default sync produces a reachable dashboard with no extra values supplied.
- [ ] Any ignored differences are recorded with the reason.

---

#### GSP-G9 · Validate the dashboard and prove the chain end to end

> **Issue type:** Story  **Epic Link:** the epic above  **Estimate:** 5
> **Depends on:** GSP-G8, GSP-G3, GSP-G6

**Summary:** `Validate the generated dashboard Application and prove the full platform chain on a cluster`

**Description:**

```
Prove the generated Application works, and use it to close the epic.

The dashboard observes the operator and the RBAC automation, so it is the natural place to prove the
whole platform works together: if the dashboard shows the groups the operator synced and the access
the RBAC automation bound, all three deployments are correct and correctly wired. That end-to-end
check is this story's real deliverable — the Application going Healthy is only the precondition for it.

The access-control tiers also need verifying against a GitOps-deployed instance specifically, since
some tiers behave differently depending on how they are reached, and a manual install is not proof
for the generated one.
```

**Acceptance criteria:**

- [ ] The Application reaches Healthy/Synced on the target cluster.
- [ ] The reader tiers are verified against the GitOps-deployed instance, each by its documented method.
- [ ] The dashboard shows the groups synced by the GSP-G3 deployment.
- [ ] The dashboard shows the access bound by the GSP-G6 deployment.
- [ ] A clean-cluster bring-up following the documented order produces the whole platform through GitOps alone.
- [ ] The install order is documented in a place a new operator would find it.

---

## 5. Sequencing

```
                      ┌────────────────────────────────────────┐
                      │  EPIC: Group Sync Platform             │
                      └────────────────────────────────────────┘
                                        │
         ┌──────────────────────────────┼──────────────────────────────┐
         │                              │                              │
    ┌─────────┐                    ┌─────────┐                    ┌─────────┐
    │ GSP-H1  │                    │ GSP-H2  │                    │ GSP-H3  │   Helm design
    │operator │                    │  NCO    │                    │dashboard│   (parallel)
    └────┬────┘                    └────┬────┘                    └────┬────┘
         │                              │                              │
         ▼                              ▼                              ▼
    ┌─────────┐                    ┌─────────┐                    ┌─────────┐
    │ GSP-G1  │                    │ GSP-G4  │                    │ GSP-G7  │   feeder entry
    └────┬────┘                    └────┬────┘                    └────┬────┘
         ▼                              ▼                              ▼
    ┌─────────┐                    ┌─────────┐                    ┌─────────┐
    │ GSP-G2  │                    │ GSP-G5  │                    │ GSP-G8  │   sync semantics
    └────┬────┘                    └────┬────┘                    └────┬────┘
         ▼                              ▼                              ▼
    ┌─────────┐   groups must     ┌─────────┐                    ┌─────────┐
    │ GSP-G3  │ ─────────────────▶│ GSP-G6  │                    │ GSP-G9  │   validation
    └─────────┘   exist to bind   └────┬────┘                    └────┬────┘
         │                              │                              │
         └──────────────────────────────┴──────────────┬───────────────┘
                                                       ▼
                                      GSP-G9 needs G3 and G6: the dashboard
                                      is where the whole chain is proven
```

Each column is one repository and runs top to bottom. The three columns are independent until the
validation row, where two cross-dependencies bite:

- **G6 needs G3** — NCO binds groups to roles, so validating it requires groups the operator has actually synced.
- **G9 needs G3 and G6** — the dashboard's closing criterion is showing data the other two produced.

Feeder entries and sync semantics for all three can proceed in parallel; only validation serialises.

**Suggested order:** H1–H3 parallel → G1/G4/G7 parallel → G2/G5/G8 parallel → G3 → G6 → G9.

---

## 6. Summary table

| Key | Type | Component | Phase | Summary | Est. | Depends on |
|---|---|---|---|---|---|---|
| GSP-E1 | Epic | all | — | Unified Helm packaging and GitOps onboarding | — | — |
| GSP-H1 | Story | operator | Helm | Chart design for the group-sync operator | 5 | — |
| GSP-H2 | Story | NCO | Helm | Chart design for the RBAC automation | 5 | — |
| GSP-H3 | Story | dashboard | Helm | Chart design for the dashboard | 5 | — |
| GSP-G1 | Story | operator | feeder | Register in ApplicationSet, create feeder entry | 3 | H1 |
| GSP-G2 | Story | operator | semantics | CRD protection, CA drift, ordering | 5 | G1 |
| GSP-G3 | Story | operator | validate | Sync healthy, groups materialise, retire manifest | 3 | G2 |
| GSP-G4 | Story | NCO | feeder | Register in ApplicationSet, create feeder entry | 2 | H2 |
| GSP-G5 | Story | NCO | semantics | CRD ownership, prune vs narrowing | 3 | G4 |
| GSP-G6 | Story | NCO | validate | Sync healthy, access bound, retire manifest | 3 | G5, G3 |
| GSP-G7 | Story | dashboard | feeder | Register in ApplicationSet, create feeder entry | 3 | H3 |
| GSP-G8 | Story | dashboard | semantics | Prune for read-only, exposure default | 2 | G7 |
| GSP-G9 | Story | dashboard | validate | Sync healthy, tiers verified, chain proven | 5 | G8, G3, G6 |

Totals: **12 stories, 44 points** — 15 for Helm design, 29 for onboarding.

---

## 7. Cross-cutting decisions to settle in the epic

These affect more than one story, so they belong on the epic:

1. **Where per-cluster values live.** All three feeder entries need a values home, and the operator's
   required `groupSync.url` makes it mandatory rather than optional for at least one of them. One
   convention across the three keeps the platform legible.
2. **CRD ownership per component.** The operator's CRD is owned by OLM and must never be managed by
   ArgoCD. The RBAC chart ships CRDs but the old manifest skipped them. These are different answers to
   the same question and both need writing down.
3. **Version pinning policy in the feeder.** The stale manifest's five-minor-version drift is what
   happens without one. Decide whether entries track a minor range or an exact version, and where the
   bump is reviewed.
4. **Retiring the stale manifests.** Two repos still carry `argocd-application.yaml` files that are not
   the deployment path. G3 and G6 remove their own; the epic should confirm none is left.

---

## 8. Notes on this document

- Chart versions, template counts and the contents of the stale manifests were read from the working
  copies in `~/gitRepos` on 2026-09-09. Chart versions move; re-check before creating the issues.
- The `groupSync.url` behaviour in H1 and G1 is quoted from the operator chart's own README and
  `values.yaml`, which document it as a rule by deployment path.
- The ApplicationSet and the `config.json` feeder live in the day-2 operations automation, which is not
  among the working copies here. The stories therefore say *what each component needs from its entry*
  and leave the entry's shape to the automation's existing conventions.
- The `GSP-` keys are placeholders. JIRA assigns real keys on creation.
- Estimates are relative sizing, not measurements. G2 carries the most risk of the nine (CRD deletion
  cascade, CA drift); G9 is a 5 because it owns the end-to-end proof for the whole epic; G4 and G8 are
  small because their decisions are largely settled by the stories before them.
