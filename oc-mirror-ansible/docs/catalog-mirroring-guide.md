# Operator Catalog Mirroring with oc-mirror v2

Oct 6, 2026 · Lateef Onifade

## Overview

Operator catalogs are mirrored in their **own runs**, separate from the OpenShift release images. They're usually the biggest and slowest part of a disconnected mirror. The same oc-mirror v2 playbook does the work, with an ImageSet that lists only catalogs and only the operators you use.

Red Hat publishes four catalogs, one image per OpenShift minor version. The tag is `v` + the version, so this guide builds every catalog name from one variable:

```
OCP_VERSION=4.18
echo registry.redhat.io/redhat/redhat-operator-index:v$OCP_VERSION
```

| Catalog | Image | Content |
| --- | --- | --- |
| Red Hat | `registry.redhat.io/redhat/redhat-operator-index:v$OCP_VERSION` | Operators built and supported by Red Hat (ODF, Update Service, logging, and so on) |
| Certified | `registry.redhat.io/redhat/certified-operator-index:v$OCP_VERSION` | Partner operators certified by Red Hat, supported by the partner |
| Marketplace | `registry.redhat.io/redhat/redhat-marketplace-index:v$OCP_VERSION` | Certified operators bought through Red Hat Marketplace |
| Community | `registry.redhat.io/redhat/community-operator-index:v$OCP_VERSION` | Community operators, no Red Hat support |

Three rules shape everything below:

1. **Mirror only what you use.** A catalog listed without packages mirrors the head version of every operator in it, which can be hundreds of gigabytes. This guide lists packages explicitly.
2. **oc-mirror v2 does not pull in dependencies.** Red Hat: "You must explicitly specify any required dependent packages and their versions." If an operator depends on another, list both.
3. **Each OpenShift version needs its own catalog.** A 4.18 cluster uses `v4.18` catalogs, and a 4.20 cluster `v4.20`. Before each update hop, mirror the catalogs for the next version.

## Before you start

This guide builds on the release mirroring guide, "oc-mirror v2 with Ansible — Host Proxy Guide". Finish its Steps 1–7 first; they set up everything catalogs need too:

- Proxy on the mirror host, with `NO_PROXY` for `localhost`, `127.0.0.1` and the registry (its Step 1)
- `ansible-core`, `tmux`, `podman`, `skopeo`, `jq` installed (its Step 2)
- The **latest** `oc-mirror` in `~/oc-mirror-ansible` (its Step 3)
- The Red Hat pull secret in `vars/redhat-pull-secret.json` and the registry login in `vars/credentials.yml`, both Vault-encrypted (its "Ansible Vault in this project" section and Step 6)
- The registry host, path and login scope agreed with the Artifactory admin (its Step 5)
- `oc` access to the cluster, read-only is enough, for Step 1 below

The Red Hat pull secret covers all four catalogs; they're all on `registry.redhat.io`. The certified and marketplace operator images themselves can come from other registries (often `registry.connect.redhat.com`); the pull secret normally covers that one too.

Catalog content is big. Plan for a separate disk budget and agree it with your lead after the dry-run in Step 5.

## Step 1 — Find what the cluster uses

Start from the cluster, not the catalog. Everything installed today must stay available after the mirror, at a version for each OpenShift release on the path. These commands only read; run them with `oc` logged in to the cluster.

1. **Catalog sources** the cluster knows about, and the image each one points to:

   ```
   oc get catalogsource -n openshift-marketplace \
     -o custom-columns=NAME:.metadata.name,IMAGE:.spec.image,DISPLAY:.spec.displayName
   ```

   On a disconnected cluster these point at your registry (for example `…/redhat/redhat-operator-index:v4.18`). The tag shows which catalog version the cluster uses now.
2. **Installed operators**: one line per Subscription, with its catalog source, package, channel and installed version:

   ```
   oc get subscriptions.operators.coreos.com -A -o json \
     | jq -r '.items[] | [.metadata.namespace, .spec.source, .spec.name, .spec.channel, (.status.installedCSV // "-")] | @tsv' \
     | sort -t$'\t' -k2,3 | tee ~/operator-inventory.tsv
   ```

   Dependencies that OLM installed automatically get their own Subscription, so they appear here too. That matters because oc-mirror v2 won't add them for you.
3. **Versions actually running** (skips the copies OLM places in every namespace):

   ```
   oc get csv -A -l '!olm.copiedFrom' \
     -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,VERSION:.spec.version,PHASE:.status.phase
   ```
4. **Operators installed with OLM v1**, if any (newer clusters): `oc get clusterextensions 2>/dev/null`.

Map each Subscription's source to its Red Hat catalog. On a disconnected cluster, read the catalog from the CatalogSource image in item 1 instead.

| Subscription source | Catalog |
| --- | --- |
| `redhat-operators` | `redhat-operator-index` |
| `certified-operators` | `certified-operator-index` |
| `redhat-marketplace` | `redhat-marketplace-index` |
| `community-operators` | `community-operator-index` |

Record the result in the inventory below. It becomes the package list in Step 3 and the checklist for each update hop. Add one row per operator; the first row is the Update Service from the release guide.

| Catalog | Package | Channel | Installed version | Owner / why we need it |
| --- | --- | --- | --- | --- |
| redhat-operator-index | cincinnati-operator | v1 |  | OpenShift Update Service |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |

## Step 2 — Explore the catalogs

Use `oc-mirror list` (the v2 form, available in the latest oc-mirror) to see what each catalog offers for a given OpenShift version. These commands read from `registry.redhat.io` through the proxy; nothing is copied.

1. **Log in** with your team's Red Hat account. The login is held in memory-backed storage until you log out:

   ```
   cd ~/oc-mirror-ansible
   ls ~/.docker/config.json 2>/dev/null && echo "Move this file aside first"
   podman login registry.redhat.io
   OCP_VERSION=4.18
   ```
2. **Which catalogs exist** for that version:

   ```
   ./oc-mirror list operators --version $OCP_VERSION --catalogs --v2
   ```
3. **Which packages each catalog has.** Save each list; each takes a few minutes:

   ```
   for c in redhat-operator-index certified-operator-index redhat-marketplace-index community-operator-index; do
     ./oc-mirror list operators --catalog=registry.redhat.io/redhat/$c:v$OCP_VERSION --v2 \
       > ~/catalog-$c-v$OCP_VERSION.txt
     echo "$c: $(wc -l < ~/catalog-$c-v$OCP_VERSION.txt) lines"
   done
   ```
4. **Channels of one package**, including its default channel:

   ```
   ./oc-mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v$OCP_VERSION \
     --package=<package> --v2
   ```
5. **Versions in one channel:**

   ```
   ./oc-mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v$OCP_VERSION \
     --package=<package> --channel=<channel> --v2
   ```
6. **Log out** when done: `podman logout registry.redhat.io`.

### Check every installed operator exists in each target version

Repeat item 3 with `OCP_VERSION=4.20` and `4.22` (and the hop versions, Step 7). Then check each package from your inventory is still offered:

```
for v in 4.18 4.20 4.22; do
  for pkg in $(cut -f3 ~/operator-inventory.tsv | sort -u); do
    grep -qw -- "$pkg" ~/catalog-*-v$v.txt || echo "v$v: $pkg NOT FOUND"
  done
done
```

A `NOT FOUND` means the operator was renamed, moved to another catalog, or dropped in that version. Settle it with the operator's owner before the cluster update, not after.

## Step 3 — Build the catalog ImageSet

You describe each group of catalogs in a small, non-secret vars file. The playbook turns it into an ImageSet with the catalog tag set to `v` + the version. Use **two groups**, because the Red Hat catalog keeps its image signatures and the partner catalogs don't (see Step 5):

| Group file | Catalogs | Signatures |
| --- | --- | --- |
| `vars/catalogs-redhat.yml` | `redhat-operator-index` | Mirrored (default) |
| `vars/catalogs-partner.yml` | `certified-operator-index`, `redhat-marketplace-index`, `community-operator-index` (only the ones you use) | Off: `--remove-signatures` |

Red Hat says that for the certified, marketplace and community catalogs it "does not ensure the availability or validity of their signatures. In such cases, you must disable signature mirroring."

Example `vars/catalogs-redhat.yml`. Replace the packages with your Step 1 inventory:

```
# SAMPLE ONLY: IBM Fusion 2.14's sample setup + the Update Service. Replace with your Step 1 inventory.
# Red Hat catalog (signatures mirrored).
catalog_group: redhat
operator_catalogs:
  - name: redhat-operator-index
    packages:
      - name: cincinnati-operator
      - name: kubernetes-nmstate-operator
      - name: redhat-oadp-operator
        channels:
          - name: stable-1.4
            minVersion: "1.4.2-0"
          - name: stable
            minVersion: "1.5.0-0"
      - name: amq-streams
        channels:
          - name: stable
            minVersion: "3.0.0-0"
      - name: nfd
      - name: node-maintenance-operator
      - name: rhods-operator
      - name: openshift-cert-manager-operator
      - name: kubevirt-hyperconverged
      - name: metallb-operator
      - name: multicluster-engine
      - name: local-storage-operator
      - name: lvms-operator
      - name: kernel-module-management
      - name: openshift-gitops-operator
      - name: submariner
      - name: advanced-cluster-management
# Extra images; the version is filled in from catalog_ocp_version
additional_images:
  - registry.redhat.io/ubi9/ubi:latest
  - registry.redhat.io/ubi9/ubi-minimal:latest
  - registry.redhat.io/ubi9/ubi-micro:latest
  - "registry.redhat.io/openshift4/driver-toolkit-rhel9:v{{ catalog_ocp_version }}"
```

Example `vars/catalogs-partner.yml`:

```
# SAMPLE ONLY: IBM Fusion 2.14's sample setup. Replace with your Step 1 inventory.
# Certified catalog (run with --remove-signatures).
catalog_group: partner
operator_catalogs:
  - name: certified-operator-index
    packages:
      - name: cloudnative-pg
      - name: ibm-storage-odf-operator
        channels:
          - name: stable-v1.9
      - name: ibm-block-csi-operator
        defaultChannel: "stable-1.13.2"
        channels:
          - name: stable-1.13.2
```

These files are a sample: IBM Fusion 2.14's sample setup, split into the two groups, to show the format. They are not our operator list. Build your real files from the Step 1 inventory: keep a sample entry only if the inventory has it, and add every operator the inventory shows. The only entry carried over on purpose is cincinnati-operator, the Update Service from the release guide. Things to know about them:

- **Everything under `packages` goes into the ImageSet as written**, so any field oc-mirror accepts works there.
- **`minVersion: "1.4.2-0"`**: the `-0` makes the range include every build of that version (`1.4.2-1`, `1.4.2-2`, …), which operators often publish as pre-release-style versions. Keep the quotes.
- **`defaultChannel`**: `ibm-block-csi-operator` mirrors a channel that isn't its catalog default, so IBM names the default explicitly. Red Hat requires this whenever the filtered channels leave out the default.
- **`additional_images`** (optional): images that aren't operators. Write `{{ catalog_ocp_version }}` where the OpenShift version belongs; the playbook fills it in, so `driver-toolkit-rhel9:v4.18` becomes `v4.20` when you mirror 4.20 catalogs. Put them in the Red Hat group, since they come from `registry.redhat.io`.

### Choosing what each package entry mirrors

From Red Hat's filtering rules for oc-mirror v2:

| You write | oc-mirror mirrors | Use it for |
| --- | --- | --- |
| `- name: <pkg>` | The head (newest) version of every channel | Operators you'll install fresh, or that jump straight to the newest version |
| `channels: [name: <ch>]` | The head of that channel only | Pinning one channel. Include the package's **default channel** too, or set `defaultChannel` |
| `channels: [name: <ch>, minVersion: <installed>]` | That channel from your installed version up to its head | **Installed operators**: keeps every version OLM may need to update through |
| Package-level `minVersion` / `maxVersion` | That range across all channels | Rarely; can pull many versions |

Rules Red Hat marks as "do not use":

- Don't combine a channel filter with a package-level `minVersion`/`maxVersion`.
- Don't combine `full: true` with `minVersion`/`maxVersion`.
- Don't leave out `packages`: a catalog with no package list mirrors the head of **every** operator in it. The playbook refuses this.

Always add the dependencies of each operator as packages of their own. oc-mirror v2 won't add them for you.

### How this compares with IBM's procedure

IBM Fusion 2.14 mirrors the same catalogs with a single `oc mirror --v2` command. This guide uses IBM's sample only as a format example and changes how the runs are done:

| IBM Fusion 2.14 | This guide | Why |
| --- | --- | --- |
| Certified and Red Hat catalogs in one ImageSet | Two groups, the certified one with `--remove-signatures` | Red Hat: certified, marketplace and community signatures aren't guaranteed, so signature mirroring must be off for them |
| `podman login -u <user> -p <password>` | Credentials from Ansible Vault, auth file deleted after each run | A password on the command line lands in shell history and the process list |
| `--dest-tls-verify=false` | TLS verified; the registry's CA is trusted on the host | Turning off TLS checks hides a man-in-the-middle; fix trust instead (release guide, Step 1) |
| `--workspace file://./` (current folder) | One fixed workspace per version and group under `/data/oc-mirror` | Runs can't overwrite each other's cluster resources |
| Example output mentions `oc-mirror-workspace/results-…` and ICSP manifests | `working-dir/cluster-resources` with IDMS/ITMS | That output is from oc-mirror v1; v2 writes IDMS/ITMS, and ICSP is deprecated |
| Applies the IDMS as generated (`idms-operator-0`) | Renames IDMS/ITMS per run first (Step 6) | Every run uses the same names, so one would replace another |
| Edits the `redhat-operators` CatalogSource image | Supported as an option (Step 6, item 5) after turning off default sources | Without that, the Marketplace Operator reverts the edit |

IBM's `TARGET_PATH` (`$LOCAL_ISF_REGISTRY/$LOCAL_ISF_REPOSITORY`, e.g. `registryhost.com:443/fusion-mirror`) is this guide's `target_registry_fqdn` + `target_registry_path`. Keep the port in `target_registry_fqdn`.

## Step 4 — Add the catalog playbook

Catalogs get their own playbook, `catalogs-playbook.yml`, next to the release playbook in `~/oc-mirror-ansible`. It reuses the same Vault files, registry settings, cache, retries and auth-file cleanup. Each catalog version and group gets its own workspace (for example `/data/oc-mirror/workspace-catalogs-v4.18-redhat` and `…-v4.18-partner`), so one run's cluster resources never overwrite another's. Each vars file names its group with `catalog_group`.

1. Create the template:

   ```
   cd ~/oc-mirror-ansible
   cat > templates/catalogs-imageset.yaml.j2 <<'EOF'
   apiVersion: mirror.openshift.io/v2alpha1
   kind: ImageSetConfiguration
   mirror:
     operators:
   {% for cat in operator_catalogs %}
       - catalog: registry.redhat.io/redhat/{{ cat.name }}:v{{ catalog_ocp_version }}
         packages:
           {{ cat.packages | to_nice_yaml(indent=2) | indent(8) }}
   {% endfor %}
   {% if additional_images | default([]) | length > 0 %}
     additionalImages:
   {% for img in additional_images %}
       - name: {{ img }}
   {% endfor %}
   {% endif %}
   EOF
   ```
2. Create `vars/catalogs-redhat.yml` and, if you use partner catalogs, `vars/catalogs-partner.yml` from Step 3. These hold no secrets; don't encrypt them.
3. Create `catalogs-playbook.yml` with the content below (`vi catalogs-playbook.yml`, `i`, paste, `Esc`, `:wq`), then check it: `ansible-playbook --syntax-check --ask-vault-pass catalogs-playbook.yml`.

The playbook stops before doing anything if no catalog list is passed, or if a catalog has no `packages` list.

| Variable | Default | Purpose |
| --- | --- | --- |
| `catalog_ocp_version` | `4.18` | OpenShift minor version; the catalog tag becomes `v4.18` |
| `operator_catalogs` | none | Catalogs and packages; pass with `-e @vars/catalogs-<group>.yml` |
| `oc_mirror_extra_args` | empty | Extra oc-mirror flags, e.g. `--remove-signatures` for the partner group |
| `oc_mirror_dry_run` | `false` | `true` lists what would be mirrored and copies nothing |
| `target_registry_fqdn`, `target_registry_path`, `target_auth_key` | as in the release guide | Same registry settings as the release runs |
| `workspace_path` | `/data/oc-mirror/workspace-catalogs-v<version>` | One workspace per catalog version |
| `catalog_group` | `redhat` | Set in each vars file (redhat, partner); part of the workspace name |
| `additional_images` | none | Optional non-operator images, set in a vars file; may contain `{{ catalog_ocp_version }}` |

```
---
- name: Mirror OpenShift operator catalogs with oc-mirror v2
  hosts: localhost
  connection: local
  gather_facts: false
  vars_files:
    - vars/credentials.yml
  vars:
    # Internal registry: host (add :port if it uses one) and the repository path under it
    target_registry_fqdn: "artifactory.internal.repo"
    target_registry_path: "ocp4"                    # e.g. docker-local/ocp4
    target_registry_url: "{{ target_registry_fqdn }}/{{ target_registry_path }}"
    # Login scope in the auth file: the host (default) or the full path,
    # if the service account only has rights on that one repository
    target_auth_key: "{{ target_registry_fqdn }}"

    # Working directories on the data volume (Step 2)
    base_path: "/data/oc-mirror"
    workspace_path: "{{ base_path }}/workspace-catalogs-v{{ catalog_ocp_version }}-{{ catalog_group }}"
    cache_path: "{{ base_path }}/cache"
    auth_file_path: "{{ base_path }}/private/mirroring.json"
    imageset_path: "{{ workspace_path }}/imageset.yaml"

    # Catalog version: the OpenShift minor version, used as the catalog tag v<version>
    catalog_ocp_version: "4.18"
    # Which catalogs and packages: pass a vars file with -e @vars/catalogs-<group>.yml
    catalog_group: "redhat"        # set in each vars file; names the workspace
    operator_catalogs: []
    oc_mirror_dry_run: false        # true = list what would be mirrored, copy nothing
    oc_mirror_extra_args: ""        # e.g. --remove-signatures for certified/marketplace/community

    # Timeout handling: oc-mirror's own retries, then whole-run retries
    oc_mirror_tuning: >-
      --parallel-images 4
      --parallel-layers 4
      --image-timeout 30m
      --retry-times 5
    oc_mirror_retries: 3
    oc_mirror_retry_delay: 120

    # Proxy comes from the host environment (HTTPS_PROXY / NO_PROXY).
    # NO_PROXY must include localhost, 127.0.0.1 and the target registry.

  tasks:
    - name: Check a catalog list was given
      ansible.builtin.assert:
        that:
          - operator_catalogs | length > 0
          - operator_catalogs | selectattr('packages', 'undefined') | list | length == 0
        fail_msg: "Pass a catalog list, e.g. -e @vars/catalogs-redhat.yml, and give every catalog a packages list."
        quiet: true

    - name: Ensure working directories exist
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: directory
        mode: "{{ item.mode }}"
      loop:
        - { path: "{{ workspace_path }}", mode: '0755' }
        - { path: "{{ cache_path }}", mode: '0755' }
        - { path: "{{ auth_file_path | dirname }}", mode: '0700' }

    - name: Main Execution Block
      block:
        - name: Load Red Hat pull secret from Vault
          ansible.builtin.set_fact:
            redhat_pull_secret: "{{ lookup('ansible.builtin.file', playbook_dir + '/vars/redhat-pull-secret.json') | from_json }}"
          no_log: true

        - name: Check the pull secret covers the Red Hat registries
          ansible.builtin.assert:
            that:
              - "'quay.io' in redhat_pull_secret.auths"
              - "'registry.redhat.io' in redhat_pull_secret.auths"
            fail_msg: "vars/redhat-pull-secret.json is missing quay.io or registry.redhat.io. Copy it again from console.redhat.com (Step 4)."
            quiet: true

        - name: Merge Red Hat pull secret with internal registry credentials
          ansible.builtin.copy:
            dest: "{{ auth_file_path }}"
            mode: '0600'
            content: "{{ merged_auths | to_nice_json }}"
          no_log: true
          vars:
            base_secret_dict: "{{ redhat_pull_secret }}"
            target_auth_base64: "{{ (target_registry_user + ':' + target_registry_password) | b64encode }}"
            merged_auths: "{{ base_secret_dict | combine({'auths': {target_auth_key: {'auth': target_auth_base64}}}, recursive=True) }}"

        - name: Render ImageSet template
          ansible.builtin.template:
            src: "./templates/catalogs-imageset.yaml.j2"
            dest: "{{ imageset_path }}"
            mode: '0644'

        - name: Execute oc-mirror v2 for catalogs (retried on failure)
          ansible.builtin.command: >
            {{ playbook_dir }}/oc-mirror --v2
            --authfile {{ auth_file_path }}
            --workspace file://{{ workspace_path }}
            --cache-dir {{ cache_path }}
            {{ oc_mirror_tuning }}
            {{ '--dry-run' if oc_mirror_dry_run | bool else '' }}
            {{ oc_mirror_extra_args }}
            -c {{ imageset_path }}
            docker://{{ target_registry_url }}
          register: oc_mirror_exec
          retries: "{{ oc_mirror_retries }}"
          delay: "{{ oc_mirror_retry_delay }}"
          until: oc_mirror_exec.rc == 0
          changed_when: oc_mirror_exec.rc == 0

      always:
        - name: Cleanup auth file (runs even on failure)
          ansible.builtin.file:
            path: "{{ auth_file_path }}"
            state: absent
```

## Step 5 — Dry-run, size and run

Always dry-run a catalog ImageSet first. A wrong package list is the most common way to fill a disk.

1. **Dry-run each group.** Use your registry values; `$REG` keeps the commands short:

   ```
   cd ~/oc-mirror-ansible
   REG="-e target_registry_fqdn=artifactory.internal.repo -e target_registry_path=ocp4"

   ansible-playbook catalogs-playbook.yml --ask-vault-pass $REG \
     -e catalog_ocp_version=4.18 -e @vars/catalogs-redhat.yml \
     -e oc_mirror_dry_run=true

   ansible-playbook catalogs-playbook.yml --ask-vault-pass $REG \
     -e catalog_ocp_version=4.18 -e @vars/catalogs-partner.yml \
     -e oc_mirror_extra_args=--remove-signatures -e oc_mirror_dry_run=true
   ```
2. **Read the results.** For each group, count the images and check nothing is missing:

   ```
   for g in redhat partner; do d=/data/oc-mirror/workspace-catalogs-v4.18-$g/working-dir/dry-run
     echo "== $g: $(wc -l < $d/mapping.txt) images"; cat $d/missing.txt 2>/dev/null; done
   ```

   Skim `mapping.txt`. You should recognise every operator in it. A count in the thousands usually means a catalog is missing its `packages` list, or `full: true` slipped in.
3. **Agree the disk size** with your lead from the image counts. Catalog images are cached in `/data/oc-mirror/cache` alongside the release images, so check `df -h /data` first.
4. **Run for real** in tmux (release guide, Step 7), Red Hat group first. Use the same commands as item 1 without `-e oc_mirror_dry_run=true`. Catalog runs often take hours; the same timeout tuning and retries as the release runs apply.
5. **Check the result:**

   ```
   REG_HOST=artifactory.internal.repo; REG_PATH=ocp4
   podman login $REG_HOST
   skopeo list-tags docker://$REG_HOST/$REG_PATH/redhat/redhat-operator-index | jq -r '.Tags[]'
   podman logout $REG_HOST
   ls /data/oc-mirror/workspace-catalogs-v4.18-redhat/working-dir/cluster-resources/
   ```

   Expect the tag `v4.18`, and in `cluster-resources` an `idms-oc-mirror.yaml`, possibly an `itms-oc-mirror.yaml`, and one `cs-<catalog>-*.yaml` CatalogSource per catalog (plus `cc-*.yaml` ClusterCatalogs for OLM v1).

## Step 6 — Use the mirrored catalogs on the cluster

A cluster admin applies each run's `cluster-resources` folder, in an approved change window. The mirror host only produces the files.

### 1. Give each run's mirror sets a unique name

oc-mirror always names its mirror sets by category only: `idms-release-0`, `idms-operator-0`, `idms-generic-0` (and the same with `itms-`). Every run, catalog or release, reuses those names, so applying one run's files **replaces** the previous run's mappings on the cluster. For example, the partner run would wipe the Red Hat run's operator mappings, and the release run's Update Service mapping would be replaced too.

Before handover, add the run's name to every IDMS/ITMS in its folder. The cluster combines the mappings from all of them:

```
cd /data/oc-mirror/workspace-catalogs-v4.18-redhat/working-dir/cluster-resources
RUN=catalogs-v4-18-redhat
sed -i -E "s/^(\s*name: )(i[dt]ms-[a-z]+-[0-9]+)\s*$/\1\2-$RUN/" i[dt]ms-oc-mirror.yaml
grep -h 'name: i' i[dt]ms-oc-mirror.yaml
```

Repeat for every folder: each catalog group and version, and the release runs from the release guide (e.g. `RUN=release-run-a`). Use lowercase letters, digits and hyphens only.

### 2. Turn off the default online catalogs

A disconnected cluster can't reach Red Hat's default catalogs, and they get in the way of the mirrored ones. If this wasn't done at install time (check with `oc get operatorhub cluster -o jsonpath='{.spec.disableAllDefaultSources}'`), run:

```
oc patch operatorhub cluster -p '{"spec": {"disableAllDefaultSources": true}}' --type=merge
```

### 3. Apply each folder

```
oc apply -f /path/to/workspace-catalogs-v4.18-redhat/working-dir/cluster-resources/
oc apply -f /path/to/workspace-catalogs-v4.18-partner/working-dir/cluster-resources/
```

This creates the renamed IDMS/ITMS and one CatalogSource per catalog. CatalogSource names include the catalog and its tag (for example `cs-redhat-operator-index-v4-18`), so versions don't collide.

If you use option A in item 5 (keep the `redhat-operators` name), move the generated `cs-*.yaml` files out of the folder before applying it.

### 4. Check

```
oc get imagedigestmirrorset
oc get catalogsource -n openshift-marketplace
oc get pods -n openshift-marketplace
oc get packagemanifests | grep -E '<package-1>|<package-2>'
```

Each CatalogSource's pod should be `Running`, and `oc get packagemanifests` should list every package from your inventory.

### 5. Connect installed operators to the mirrored catalog

Each installed operator's Subscription names a catalog source (`spec.source`), usually `redhat-operators` or `certified-operators`. Pick **one** of these two ways and use it for every catalog.

**Option A: keep the default names (IBM's approach, fewer changes).** Create your own CatalogSource called `redhat-operators` that points at the mirrored catalog. Subscriptions keep working unchanged. This needs default sources turned off first (item 2); otherwise the Marketplace Operator reverts the change. Don't also apply the generated `cs-redhat-operator-index-*.yaml`, or the same operators appear twice.

```
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhat-operators
  namespace: openshift-marketplace
spec:
  displayName: Red Hat Operators (mirror)
  image: artifactory.internal.repo/ocp4/redhat/redhat-operator-index:v4.18
  publisher: Red Hat
  sourceType: grpc
EOF
```

Use your registry host and path in `image`. Do the same for `certified-operators` with `…/redhat/certified-operator-index:v4.18`. At each upgrade jump you then only change the tag, for example `v4.18` to `v4.20`.

**Option B: use the generated names.** Apply the generated `cs-*.yaml` files (item 3), then move each Subscription to its new source:

```
oc patch subscription.operators.coreos.com <name> -n <namespace> --type merge \
  -p '{"spec":{"source":"cs-redhat-operator-index-v4-18","sourceNamespace":"openshift-marketplace"}}'
```

With option B, every version has its own CatalogSource, so each upgrade jump means patching every Subscription again.

Either way, do it per operator during the change window and check its CSV stays `Succeeded`: `oc get csv -n <namespace>`.

## Catalogs across the upgrade path

The cluster goes 4.18 → 4.20 → 4.22 (release guide). Each version it settles on needs its own catalogs, mirrored and applied **before** the cluster gets there:

| Catalog version | Mirror it | Why |
| --- | --- | --- |
| `v4.18` | Now | What the cluster runs today; also the source for operator updates before the first jump |
| `v4.20` | Before the 4.18 → 4.20 jump | Operators after the cluster reaches 4.20 |
| `v4.22` | Before the 4.20 → 4.22 jump | Operators after the cluster reaches 4.22 |
| `v4.19`, `v4.21` | Only if an operator needs it | The cluster passes through these briefly. Some operators track the OpenShift minor version; check each one's compatibility with its owner |

Red Hat's Control Plane Only procedure starts each jump by updating installed operators to versions compatible with the target version. So before each jump: mirror the next version's catalogs, update the operators, then update the cluster.

### Keep a vars file per version

Channel names often contain the version, for example `stable-4.18` becomes `stable-4.20`. Copy the group files per version and adjust the channels, using Step 2 against that version's catalog:

```
cp vars/catalogs-redhat.yml vars/catalogs-redhat-v4.20.yml
cp vars/catalogs-redhat.yml vars/catalogs-redhat-v4.22.yml
```

Then run one version at a time. The tag, workspace and CatalogSource name all follow `catalog_ocp_version`:

```
for v in 4.20 4.22; do
  ansible-playbook catalogs-playbook.yml --ask-vault-pass $REG \
    -e catalog_ocp_version=$v -e @vars/catalogs-redhat-v$v.yml -e oc_mirror_dry_run=true
done
```

Dry-run first as in Step 5, then run for real. Remove `-e oc_mirror_dry_run=true` and run one version per tmux session, because each takes hours.

### On the cluster, per jump

1. Apply the new version's catalog folders (Step 6, with renamed mirror sets).
2. Update operators that need it to versions that support the target OpenShift version.
3. Update the cluster (release guide).
4. Switch to the new version's catalog (Step 6, item 5): with option A change the CatalogSource image tag (for example v4.18 to v4.20); with option B patch each Subscription. Move channels where the operator's owner says to.
5. Keep the previous version's CatalogSource until every operator has moved, then delete it.

## Troubleshooting

For proxy, Vault, registry and timeout problems, see the release guide's troubleshooting table. These are specific to catalogs:

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Playbook stops at "Check a catalog list was given" | No `-e @vars/...` file, or a catalog without `packages` | Pass the vars file; add a package list (Step 3) |
| Dry-run lists thousands of images | A catalog with no package filter, or `full: true` | Fix the vars file; never mirror a whole catalog by accident |
| Operator installs but its dependency is missing | oc-mirror v2 doesn't add dependencies | Add the dependency as its own package (Step 3) |
| Operator won't update after a mirror | Only the channel head was mirrored, and the update needs an in-between version | Set `minVersion` on the channel to the installed version (Step 3 table) and mirror again |
| Error about the default channel, or a channel with multiple heads | Filtered channels exclude the package's default channel | Include the default channel, or set `defaultChannel` (Step 2 shows it) |
| Partner run fails on signatures | Certified, marketplace or community images lack valid signatures | Run that group with `-e oc_mirror_extra_args=--remove-signatures` |
| "Instructed to preserve digests" mirroring a catalog | Older oc-mirror with a registry lacking OCI support | Use the latest oc-mirror (release guide, Step 3) |
| Package in inventory shows `NOT FOUND` for a version | Renamed, moved or dropped in that catalog version | Agree a replacement with its owner before the cluster update (Step 2) |
| CatalogSource pod not `Running` | Cluster can't pull the catalog image | Check the IDMS were applied, the pull secret covers your registry, and its CA is trusted |
| Operators lose image mappings after applying a folder | Two runs' IDMS/ITMS shared a name | Rename per run before applying (Step 6, item 1) and apply again |
| Package missing in `oc get packagemanifests` | Not in the vars file, or catalog pod still starting | Wait for the pod; check the dry-run `mapping.txt` |

## Sources

Read directly:

- Red Hat OpenShift 4.22 documentation source, [openshift/openshift-docs `enterprise-4.22`](https://github.com/openshift/openshift-docs/tree/enterprise-4.22/modules):

  - `oc-mirror-operator-catalog-filtering.adoc`: the filtering scenarios, no dependency resolution, the "do not use" combinations
  - `oc-mirror-building-image-set-config-v2.adoc`: always include the default channel; `oc mirror list ... --v2`
  - `microshift-oc-mirror-list-ops-catalogs.adoc`: the `list operators --catalogs / --catalog / --package / --channel` steps
  - `oc-mirror-imageset-config-parameters-v2.adoc`: `full`, `minVersion`, `maxVersion`, `defaultChannel`, `targetCatalog`, `targetTag`
  - `oc-mirror-signature-mirroring.adoc`: signatures mirrored by default; disable for certified, marketplace and community catalogs
  - `oc-mirror-updating-cluster-manifests-v2.adoc`: apply the `cluster-resources` folder
  - `disabling-catalogsource-objects.adoc`: `disableAllDefaultSources`
- [openshift/oc-mirror](https://github.com/openshift/oc-mirror) source: `list operators` flags; fixed IDMS/ITMS names per category (`idms-<category>-0`); CatalogSource names from catalog and tag; `--remove-signatures`

Provided by the guide's owner, pasted from IBM's documentation (the site itself is blocked from the environment this guide was written in):

- [IBM Fusion 2.14: Mirroring Red Hat operator images to enterprise registry](https://www.ibm.com/docs/en/fusion-software/2.14.0?topic=installation-mirroring-red-hat-operator-images-enterprise-registry): the `v$OCP_VERSION` catalog tags, the sample Red Hat and certified package lists and additional images shown in Step 3 (a sample setup, not our operator list), the `redhat-operators` CatalogSource approach in Step 6 (option A), and the single-command procedure compared in Step 3
