# oc-mirror v2 with Ansible — Host Proxy Guide

Oct 6, 2026 · Lateef Onifade

## Overview

One Ansible playbook mirrors OpenShift **4.18.17 from the `eus-4.18` channel** into an internal registry with `oc-mirror --v2`, for a **disconnected cluster**. The same run mirrors the OpenShift Update Service (OSUS) operator and builds its update graph, so the cluster can see and apply updates on its own. The cluster has no Red Hat credentials, so the Red Hat pull secret comes from console.redhat.com and is stored in Ansible Vault, next to the internal registry login.

The playbook runs these steps on the mirror host, keeping all its files under /data/oc-mirror:

1. Reads the Red Hat pull secret from a Vault-encrypted file and checks that it covers `quay.io` and `registry.redhat.io`.
2. Adds the internal registry login, also from Vault.
3. Writes the merged auth file with mode `0600` in a private folder and renders the `ImageSetConfiguration`: the release, the update graph (`graph: true`) and the OSUS operator.
4. Runs `oc-mirror --v2` from the Red Hat registries to the internal registry, with up to 4 attempts if it times out.
5. Deletes the auth file in an `always:` block, which runs on success and on failure.

Steps 1–9 run on the mirror host. Step 10 is done on the disconnected cluster by a cluster admin.

The proxy is set on the host, not in the playbook. `oc` and `oc-mirror` inherit `HTTPS_PROXY` and `NO_PROXY` from the shell that starts `ansible-playbook`. This guide adapts Fabio Reis's article *Automating OpenShift oc-mirror v2 with Ansible* (Medium, Aug 18). The article passes the proxy as a playbook variable instead.

## Before you start

Get these from other teams before you touch the host. Each later step assumes you have them.

- [ ] **Mirror host**: a RHEL 9 VM you can SSH to, with `sudo` rights and a registered subscription (`sudo subscription-manager status`)
- [ ] **Disk**: at least 100 GB free on a data volume, e.g. mounted at `/data` (one release is tens of GB, and the cache is kept between runs)
- [ ] **Red Hat account**: the login your team uses for console.redhat.com (ask your lead which one; don't use a personal account unless told to)
- [ ] **Proxy**: the address and port (e.g. `proxy.corp:8080`), and whether it needs a username
- [ ] **Proxy CA certificate**: only if the proxy re-signs HTTPS traffic; ask the network team for the `.crt` file
- [ ] **Internal registry**: the exact **host** (with `:port` if it uses one), the **repository path** under it (e.g. `ocp4` or `docker-local/ocp4`), a service account that can push, and whether that account is limited to that one path (Step 5)
- [ ] **Release**: `eus-4.18` channel, version `4.18.17` (this guide's defaults)
- [ ] **Cluster admin for Step 10**: someone with `cluster-admin` on the disconnected cluster to install the Update Service

You don't need access to the disconnected cluster for Steps 1–9.

All commands below run on the mirror host as your own user unless they start with `sudo`. Replace values in `<angle brackets>` with yours.

## Step 1 — Configure the proxy on the host

Internet traffic goes through the proxy. Local and internal traffic must bypass it, so `NO_PROXY` matters as much as `HTTPS_PROXY`.

![Network routes from the mirror host: proxy vs direct](images/network-routes.png)

If either direct route goes through the proxy instead, the run fails.

Create `/etc/profile.d/proxy.sh`. Replace the proxy address and registry host with your values first:

```bash
sudo tee /etc/profile.d/proxy.sh > /dev/null <<'EOF'
export HTTP_PROXY=http://proxy.corp:8080
export HTTPS_PROXY=http://proxy.corp:8080
export NO_PROXY=localhost,127.0.0.1,.internal.repo,artifactory.internal.repo
export http_proxy=$HTTP_PROXY
export https_proxy=$HTTPS_PROXY
export no_proxy=$NO_PROXY
EOF
sudo chmod 644 /etc/profile.d/proxy.sh
```

Set both upper and lower case: different tools read different spellings. Log out and back in, then check with `env | grep -i _proxy`.

`NO_PROXY` must contain:

| Entry | Why |
| --- | --- |
| `localhost,127.0.0.1` | oc-mirror v2 runs a temporary local registry on `localhost:55000`. Sent through the proxy, the mirror fails. |
| `artifactory.internal.repo` or `.internal.repo` | Pushes to the internal registry must go direct. |

Use the registry **host only** in `NO_PROXY`: no port, no path. `artifactory.internal.repo` covers `artifactory.internal.repo:8443/docker-local/ocp4`.

The update graph build also needs `api.openshift.com` and `registry.access.redhat.com` through the proxy. Ask the network team to allow both, alongside `quay.io` and `registry.redhat.io`.

If the proxy re-signs TLS traffic, trust its CA on the host. `oc` and `oc-mirror` use the system trust store:

```bash
sudo cp corp-proxy-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

Where the variables do not reach:

- **Cron, systemd or CI runners** don't read `/etc/profile.d`. Set the variables in the job definition or unit file.
- **AAP / AWX** runs the job in an execution-environment container that doesn't see host variables. Set them in the job template's environment instead.
- **`sudo`** drops proxy variables unless `env_keep` lists them. The playbook doesn't need root, so run it as your own user.

## Step 2 — Prepare the mirror host

Install the tools and create the data directories. `dnf` does not read the shell proxy variables, so give it its own proxy settings first.

1. Point `dnf` and Red Hat Subscription Manager at the proxy:

   ```bash
   echo 'proxy=http://proxy.corp:8080' | sudo tee -a /etc/dnf/dnf.conf
   sudo subscription-manager config --server.proxy_hostname=proxy.corp --server.proxy_port=8080
   ```
2. Install the packages. `ansible-core` is in the RHEL 9 AppStream repository:

   ```bash
   sudo dnf install -y ansible-core tmux podman skopeo jq tar
   ```
3. Check that each tool runs:

   ```bash
   ansible-playbook --version | head -1
   tmux -V
   podman --version
   jq --version
   ```
4. Create the working directories on the data volume and make them yours. `private` holds the auth file during a run, so only you can read it:

   ```bash
   sudo mkdir -p /data/oc-mirror
   sudo chown "$USER": /data/oc-mirror
   mkdir -p /data/oc-mirror/{workspace,cache,private}
   chmod 700 /data/oc-mirror/private
   df -h /data
   ```

If `dnf install` hangs or reports a timeout, the proxy line in `/etc/dnf/dnf.conf` is wrong. Fix it before you continue.

## Step 3 — Install oc and oc-mirror

The two tools come from different versions:

- **`oc-mirror`: always the latest release**, whatever version you mirror. Red Hat: "Use the latest available version of the oc-mirror plugin v2 regardless of which versions of OpenShift Container Platform you need to mirror." Newer builds carry fixes for mirroring (4.22 fixed catalog mirroring to registries without OCI support, for example).
- **`oc`: the cluster's version** (4.18.17 now). The playbook doesn't use it; the cluster admin needs it in Step 10 and must update it to the target version before each update hop.

1. Set the folders and list what they offer. `stable` always points to the newest stable release:

   ```bash
   OCP_VERSION=4.18.17
   CLIENTS=https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp
   curl -s $CLIENTS/stable/ | grep -oE '(release.txt|oc-mirror[^"]*\.tar\.gz)' | sort -u
   curl -s $CLIENTS/stable/release.txt | grep -m1 'Name:'
   curl -s $CLIENTS/$OCP_VERSION/ | grep -oE 'openshift-client[^"]*\.tar\.gz' | sort -u
   ```

   The second command shows which version `stable` is (the newest 4.22 release when this guide was checked was 4.22.16, issued 29 September 2026). If a file name contains `rhel9`, use that one on RHEL 9. Otherwise use the plain one. The commands below use the plain names.
2. Confirm `4.18.17` is in the `eus-4.18` channel. This asks Red Hat's update service and must print `4.18.17`:

   ```bash
   curl -s -H 'Accept: application/json' \
     'https://api.openshift.com/api/upgrades_info/v1/graph?channel=eus-4.18&arch=amd64' \
     | jq -r '.nodes[].version' | grep -x "$OCP_VERSION"
   ```

   No output means the version isn't in that channel. Stop and check with your lead before going on.
3. Install `oc` at the cluster's version:

   ```bash
   cd /tmp
   curl -fLO $CLIENTS/$OCP_VERSION/openshift-client-linux.tar.gz
   tar -xzf openshift-client-linux.tar.gz oc
   sudo install -m 755 oc /usr/local/bin/oc
   oc version --client
   ```
4. Download the **latest** `oc-mirror` into the project folder:

   ```bash
   mkdir -p ~/oc-mirror-ansible && cd ~/oc-mirror-ansible
   curl -fLO $CLIENTS/stable/oc-mirror.tar.gz
   tar -xzf oc-mirror.tar.gz && rm oc-mirror.tar.gz
   chmod +x oc-mirror
   ./oc-mirror version
   ./oc-mirror --v2 --help | grep -E 'parallel|timeout|retry|cache-dir|remove-signatures'
   ```

   Before each new mirror run, check whether `stable` has moved and download again if it has.

The last command in item 4 above lists the tuning flags your `oc-mirror` build supports. Keep that output; Step 6 uses those flags. If `curl` fails, go back to Step 1 and check the proxy.

## Ansible Vault in this project

Ansible Vault encrypts files with a password (AES-256), so secrets can sit in the project folder without being readable. The playbook decrypts them in memory when it runs. This project keeps two Vault files, both encrypted with the **same** password:

| File | Holds | Created in |
| --- | --- | --- |
| `vars/redhat-pull-secret.json` | The Red Hat pull secret (JSON) for `quay.io` and `registry.redhat.io` | This section, then checked in Step 4 |
| `vars/credentials.yml` | The internal registry service account | Step 6 |

`ansible-vault` comes with `ansible-core` (Step 2). Run every command below from `~/oc-mirror-ansible`.

### 1. Choose and store the Vault password

Pick a long random password, for example 24+ characters from your password manager's generator. Save it in the team password manager **before** creating any file: without it, the files can't be opened again. Use it for both Vault files, because the playbook asks for one password per run.

### 2. Make the editor safe for secrets

`ansible-vault create` and `edit` open the decrypted text in `vi`. By default `vi` can write swap and history files that keep a plain-text copy. Turn them off for this shell session first:

```bash
export EDITOR='vi -n -i NONE'
```

`-n` disables the swap file and `-i NONE` the history file.

### 3. Put the Red Hat pull secret into Vault

The pull secret comes from [console.redhat.com/openshift/install/pull-secret](https://console.redhat.com/openshift/install/pull-secret), logged in with your team's Red Hat account. It is one line of JSON that starts with `{"auths":`. Use one of these two ways.

**Option A: copy and paste (preferred).** The secret never touches the disk unencrypted.

1. On the console page, click **Copy pull secret**.
2. Create the Vault file. Enter the Vault password twice when asked:

   ```bash
   cd ~/oc-mirror-ansible && mkdir -p vars
   ansible-vault create vars/redhat-pull-secret.json
   ```
3. In the editor, type `:set paste` and press Enter, so `vi` doesn't wrap or indent the long line. Press `i`, paste, press `Esc`, then type `:wq` and press Enter. Paste only the JSON, with nothing before or after it.

**Option B: from a downloaded file.** Use this if pasting into the terminal breaks the line.

1. On the console page, click **Download pull secret** and copy `pull-secret.txt` to the mirror host, into `~/oc-mirror-ansible/`.
2. Check it is valid JSON and covers the registries. This must list `quay.io` and `registry.redhat.io`:

   ```bash
   jq -r '.auths | keys[]' pull-secret.txt
   ```
3. Encrypt it into the project, then destroy the plain copy:

   ```bash
   ansible-vault encrypt pull-secret.txt --output vars/redhat-pull-secret.json
   shred -u pull-secret.txt
   ```
4. Delete the downloaded file from your laptop too.

### 4. Check the result

```bash
head -1 vars/redhat-pull-secret.json
ansible-vault view vars/redhat-pull-secret.json | jq -r '.auths | keys[]'
```

The first line must be `$ANSIBLE_VAULT;1.1;AES256`. The second command must list `quay.io` and `registry.redhat.io` (usually also `cloud.openshift.com` and `registry.connect.redhat.com`). A `jq` parse error means the paste is broken: fix it with `ansible-vault edit vars/redhat-pull-secret.json`.

Step 4 then tests the secret against Red Hat's registries.

### 5. Everyday commands

| Task | Command |
| --- | --- |
| Read a file | `ansible-vault view vars/credentials.yml` |
| Change a value | `ansible-vault edit vars/credentials.yml` |
| Replace the pull secret | `ansible-vault edit vars/redhat-pull-secret.json`, delete the old line, paste the new one |
| Change the Vault password on both files | `ansible-vault rekey vars/credentials.yml vars/redhat-pull-secret.json` |
| Check a file is encrypted | `head -1 <file>` shows `$ANSIBLE_VAULT;1.1;AES256` |

After `rekey`, update the password manager straight away. The old password stops working on both files.

### 6. Runs without typing the password (optional)

For scheduled runs, keep the password in a file only your account can read, and point Ansible at it instead of `--ask-vault-pass`:

```bash
( umask 077; read -rsp 'Vault password: ' p; echo; printf '%s\n' "$p" > ~/.vault-pass-oc-mirror )
chmod 400 ~/.vault-pass-oc-mirror
ansible-playbook playbook.yml --vault-password-file ~/.vault-pass-oc-mirror ...
```

This file unlocks both secrets, so keep it out of the project folder and out of Git, and agree its use with your lead.

### Rules

- Never commit a Vault file unless `head -1` shows `$ANSIBLE_VAULT`, and never commit the password or password file.
- Never paste the pull secret into a ticket, chat or the disconnected cluster.
- If the pull secret or Vault password may have leaked, reset the pull secret in the console, update the Vault file, and run `rekey`.

## Step 4 — Store the Red Hat pull secret in Vault

The disconnected cluster has no Red Hat credentials, so you get the pull secret from the Red Hat console and store it encrypted in the project. If you haven't done that yet, follow "Ansible Vault in this project" above first; the items below repeat the short version.

1. Pick a Vault password and save it in your team's password manager. You'll use the **same** password for both Vault files: this one and the registry login in Step 6.
2. Log in to [console.redhat.com/openshift/install/pull-secret](https://console.redhat.com/openshift/install/pull-secret) with your team's Red Hat account. Click **Copy pull secret**. Don't click Download; that saves a plain-text file.
3. Create the Vault file and paste the secret into it:

   ```bash
   cd ~/oc-mirror-ansible
   mkdir -p vars
   ansible-vault create vars/redhat-pull-secret.json
   ```

   Enter the Vault password twice. An editor (`vi`) opens. Press `i`, paste, press `Esc`, then type `:wq` and press Enter. The secret is one long line starting with `{"auths":`; paste nothing else.
4. Check the file is encrypted and the secret is complete:

   ```bash
   head -1 vars/redhat-pull-secret.json
   ansible-vault view vars/redhat-pull-secret.json | jq -r '.auths | keys[]'
   ```

   The first command prints `$ANSIBLE_VAULT;1.1;AES256`. The second must list `quay.io` and `registry.redhat.io`. A `jq` parse error means the paste is broken; fix it with `ansible-vault edit vars/redhat-pull-secret.json`.
5. Test both registries through the proxy. The `<(...)` hands the decrypted secret to `skopeo` without saving it to a file:

   ```bash
   OCP_VERSION=4.18.17
   skopeo inspect --authfile <(ansible-vault view vars/redhat-pull-secret.json) \
     docker://quay.io/openshift-release-dev/ocp-release:${OCP_VERSION}-x86_64 | jq -r .Name
   skopeo inspect --authfile <(ansible-vault view vars/redhat-pull-secret.json) \
     docker://registry.redhat.io/ubi9/ubi:latest | jq -r .Name
   ```

   Each asks for the Vault password, then prints an image name. `unauthorized` means the secret is wrong: copy it again into `ansible-vault edit`. A timeout means the proxy is wrong (Step 1).

### Check the Update Service operator name

The playbook mirrors the OSUS operator as package `cincinnati-operator`, channel `v1`, from the Red Hat operator catalog. That's the name Red Hat's 4.20 documentation uses. This check confirms your catalog matches before the first run; if it shows a different name, you'll set it in Step 8.

1. Log in to `registry.redhat.io` with your team's Red Hat account. The login is kept in memory-backed storage and removed in step 3:

   ```bash
   ls ~/.docker/config.json 2>/dev/null && echo "Move this file aside first"
   podman login registry.redhat.io
   ```
2. Ask the catalog for the package and its default channel. This takes a few minutes:

   ```bash
   ./oc-mirror list operators --v2 \
     --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.18 \
     --package=cincinnati-operator
   ```

   The output should show the package with channel `v1`. If it reports the package isn't found, list all packages (drop the `--package` line) and search for `update-service`.
3. Log out:

   ```bash
   podman logout registry.redhat.io
   ```

If the Red Hat account changes, or someone resets the pull secret in the console, update the file with `ansible-vault edit vars/redhat-pull-secret.json`. Don't copy this secret into the disconnected cluster.

## Step 5 — Prepare the internal registry

The images land in a Docker repository on your registry. Write down three values; they become playbook variables in Step 8:

| Value | Example | Playbook variable |
| --- | --- | --- |
| Host, with port if it uses one | `artifactory.internal.repo` or `artifactory.internal.repo:8443` | `target_registry_fqdn` |
| Repository path under the host | `ocp4` or `docker-local/ocp4` | `target_registry_path` |
| Login scope of the service account | the host, or the full host + path if the account only has rights on that repository | `target_auth_key` |

Ask the Artifactory admin for:

| Ask | Why |
| --- | --- |
| A Docker repository reachable at your host + path | Where oc-mirror pushes |
| Nested paths allowed under it | oc-mirror creates `<path>/openshift/release-images`, `<path>/openshift/release`, `<path>/openshift/graph-image` and `<path>/redhat/redhat-operator-index` |
| A service account with read, write and delete on that repository | Credentials for the Vault file in Step 6 |
| No small upload size limit or short timeout on the load balancer in front | Release layers are large; a cap here looks like a timeout |
| A TLS certificate the host trusts | If it's signed by an internal CA, add that CA as in Step 1; Step 10 needs it on the cluster too |

When you have the account, test a push to the same depth oc-mirror will use. This goes direct, not through the proxy:

```bash
REG_HOST=artifactory.internal.repo        # add :port if used
REG_PATH=ocp4                             # e.g. docker-local/ocp4
podman login $REG_HOST
podman pull registry.access.redhat.com/ubi9/ubi-minimal:latest
podman tag registry.access.redhat.com/ubi9/ubi-minimal:latest \
  $REG_HOST/$REG_PATH/openshift/push-test:latest
podman push $REG_HOST/$REG_PATH/openshift/push-test:latest
podman logout $REG_HOST
```

The push must end with `Writing manifest to image destination`.

If the account is limited to one repository, also test a login scoped to the path, which is what the playbook sends when `target_auth_key` is the full path. The temporary auth file goes in the private folder and is deleted at the end:

```bash
read -rp 'Service account: ' REG_USER; read -rsp 'Password: ' REG_PASS; echo
umask 077
printf '{"auths":{"%s":{"auth":"%s"}}}' "$REG_HOST/$REG_PATH" \
  "$(printf '%s:%s' "$REG_USER" "$REG_PASS" | base64 -w0)" > /data/oc-mirror/private/test-auth.json
skopeo copy --authfile /data/oc-mirror/private/test-auth.json \
  docker://registry.access.redhat.com/ubi9/ubi-minimal:latest \
  docker://$REG_HOST/$REG_PATH/openshift/push-test:scoped
rm -f /data/oc-mirror/private/test-auth.json; unset REG_USER REG_PASS
```

If this works, set `target_auth_key` to the full path in Step 8. Ask the admin to delete `openshift/push-test` afterwards.

## Step 6 — Create the project files

When this step is done, `~/oc-mirror-ansible` holds five items:

```
~/oc-mirror-ansible/
├── oc-mirror                          # from Step 3
├── playbook.yml
├── templates/imageset-config.yaml.j2
├── vars/redhat-pull-secret.json       # Vault-encrypted, from Step 4
└── vars/credentials.yml               # Vault-encrypted
```

1. Create the folders:

   ```bash
   cd ~/oc-mirror-ansible
   mkdir -p templates vars
   ```
2. Create the ImageSet template. By default it mirrors exactly one release (min and max are the same); for an upgrade path you set a range and `shortest_path` (see "Mirroring an upgrade path"). When `mirror_update_service` is on, it also builds the update graph and mirrors the OSUS operator:

   ```bash
   cat > templates/imageset-config.yaml.j2 <<'EOF'
   apiVersion: mirror.openshift.io/v2alpha1
   kind: ImageSetConfiguration
   mirror:
     platform:
       architectures:
         - amd64
       channels:
         - name: {{ ocp_channel }}
           minVersion: {{ ocp_min_version }}
           maxVersion: {{ ocp_max_version }}
   {% if shortest_path | bool %}
           shortestPath: true
   {% endif %}
   {% if mirror_update_service | bool %}
       graph: true
     operators:
       - catalog: {{ operator_catalog }}
         packages:
           - name: {{ osus_operator_package }}
   {% endif %}
   EOF
   ```

   Mirror only one EUS channel per ImageSet. oc-mirror v2 has a known problem when two EUS channels are listed together.
3. Create the registry login Vault file. `ansible-vault create` asks for a Vault password (use the same one as in Step 4), then opens an editor:

   ```bash
   ansible-vault create vars/credentials.yml
   ```

   Type these two lines with the Artifactory service account from Step 5, then save and quit (in `vi`: `i` to type, `Esc`, then `:wq`):

   ```yaml
   target_registry_user: "<service-account>"
   target_registry_password: "<password>"
   ```

   Check it worked: `head -1 vars/credentials.yml` prints `$ANSIBLE_VAULT;1.1;AES256`.
4. Create `playbook.yml` with the content below (`vi playbook.yml`, `i`, paste, `Esc`, `:wq`). Then check it with `ansible-playbook --syntax-check --ask-vault-pass playbook.yml`.

Compared with the original article, this playbook:

- reads the Red Hat pull secret from Vault instead of the cluster, and stops early if it doesn't cover `quay.io` and `registry.redhat.io`;
- takes the registry host, repository path and login scope as separate variables (`target_registry_fqdn`, `target_registry_path`, `target_auth_key`);
- mirrors `eus-4.18` / `4.18.17` by default, plus the update graph and the OSUS operator (`mirror_update_service`);
- takes the proxy from the host (Step 1), not from a variable;
- keeps all files under `/data/oc-mirror`, with the auth file in a `0700` folder instead of `/tmp`;
- keeps the oc-mirror cache in `/data/oc-mirror/cache`, so a rerun skips images already copied;
- adds timeout handling: gentler oc-mirror settings and up to 4 attempts at the mirror run; oc-mirror's own retries back off exponentially;
- calls `oc-mirror` by full path, so it works from any directory.

Before the first run, compare `oc_mirror_tuning` with the flag list you saved in Step 3. Remove any flag your build doesn't list.

```yaml
---
- name: Automate OpenShift oc-mirror v2
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
    workspace_path: "{{ base_path }}/workspace"
    cache_path: "{{ base_path }}/cache"
    auth_file_path: "{{ base_path }}/private/mirroring.json"
    imageset_path: "{{ workspace_path }}/imageset.yaml"

    # Release to mirror (override with -e)
    ocp_channel: "eus-4.18"
    ocp_version: "4.18.17"
    # For an upgrade path, set a range instead of one version (see "Mirroring an upgrade path")
    ocp_min_version: "{{ ocp_version }}"
    ocp_max_version: "{{ ocp_version }}"
    shortest_path: false            # true = only the releases on the update path
    oc_mirror_dry_run: false        # true = list what would be mirrored, copy nothing

    # OpenShift Update Service: mirror the update graph and its operator
    mirror_update_service: true
    osus_operator_package: "cincinnati-operator"
    operator_catalog: "registry.redhat.io/redhat/redhat-operator-index:v{{ ocp_min_version.split('.')[0:2] | join('.') }}"

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
            src: "./templates/imageset-config.yaml.j2"
            dest: "{{ imageset_path }}"
            mode: '0644'

        - name: Execute oc-mirror v2 (retried on failure)
          ansible.builtin.command: >
            {{ playbook_dir }}/oc-mirror --v2
            --authfile {{ auth_file_path }}
            --workspace file://{{ workspace_path }}
            --cache-dir {{ cache_path }}
            {{ oc_mirror_tuning }}
            {{ '--dry-run' if oc_mirror_dry_run | bool else '' }}
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

## Step 7 — Start a tmux session

A mirror run can take hours. Started in a plain SSH session, it stops if your laptop sleeps or the VPN drops. In `tmux`, it keeps running on the host and you can reconnect later.

1. Start a named session from a fresh login, so it has the proxy variables:

   ```bash
   tmux new -s mirror
   env | grep -i _proxy        # check the variables are set inside tmux
   ```
2. Detach and leave it running: press `Ctrl+b`, release, then press `d`.
3. Reconnect later, even from a new SSH session:

   ```bash
   tmux ls                     # list sessions
   tmux attach -t mirror
   ```
4. Scroll back through the output: `Ctrl+b`, then `[`, then the arrow or Page Up keys. Press `q` to stop scrolling.
5. When the run is finished, type `exit` inside the session to close it.

| Keys | Action |
| --- | --- |
| `Ctrl+b` then `d` | Detach (session keeps running) |
| `Ctrl+b` then `[` | Scroll mode; `q` to leave |
| `Ctrl+b` then `c` | New window in the same session |
| `Ctrl+b` then `n` | Next window |

If you change `/etc/profile.d/proxy.sh` while a tmux session is open, the session keeps the old values. Close it with `exit`, log out and in, and start a new one.

## Step 8 — Run the playbook

Run inside the tmux session from Step 7, as your own user, not with `sudo`.

1. Go to the project and confirm both Vault files are there:

   ```bash
   cd ~/oc-mirror-ansible
   ls vars/
   ```
2. Start the run with your registry values from Step 5. `tee` shows the output on screen and saves it to a dated log file:

   ```bash
   ansible-playbook playbook.yml --ask-vault-pass \
     -e "ocp_channel=eus-4.18" \
     -e "ocp_version=4.18.17" \
     -e "target_registry_fqdn=artifactory.internal.repo" \
     -e "target_registry_path=ocp4" \
     2>&1 | tee ~/mirror-$(date +%F-%H%M).log
   ```

   If the service account is limited to that one path (Step 5), add `-e "target_auth_key=artifactory.internal.repo/ocp4"` (your host and path). If Step 4 showed a different operator name, add `-e "osus_operator_package=<name>"`.
3. Enter the Vault password when asked. Then detach (`Ctrl+b`, `d`) and check back later. The first run downloads the release, the operator catalog and the graph data, so expect it to take hours.

What you'll see:

- The pull secret and merge tasks show no detail. On failure they say the output was hidden because of `no_log: true`. That protects the secrets; it is not itself the error.
- `FAILED - RETRYING: ... (3 retries left)` means one mirror attempt failed and the playbook is trying again after 2 minutes. It's only a problem if the last attempt also fails.
- A successful run ends with a recap line showing `failed=0`.

Variables you can override with `-e`:

| Variable | Default | Purpose |
| --- | --- | --- |
| `ocp_channel` | `eus-4.18` | Release channel; one EUS channel per run |
| `ocp_version` | `4.18.17` | The one release to mirror when no range is set |
| `ocp_min_version` | `ocp_version` | Start of a range; also picks the operator catalog version |
| `ocp_max_version` | `ocp_version` | End of a range |
| `shortest_path` | `false` | `true` mirrors only the releases on the update path between min and max |
| `oc_mirror_dry_run` | `false` | `true` lists what would be mirrored and copies nothing |
| `workspace_path` | `/data/oc-mirror/workspace` | Use one per run for upgrade paths, e.g. `workspace-run-a` |
| `target_registry_fqdn` | `artifactory.internal.repo` | Registry host, with `:port` if it uses one |
| `target_registry_path` | `ocp4` | Repository path under the host, e.g. `docker-local/ocp4` |
| `target_auth_key` | the host | Login scope; set to host + path if the account is limited to that repository |
| `mirror_update_service` | `true` | Build the update graph and mirror the OSUS operator |
| `osus_operator_package` | `cincinnati-operator` | OSUS operator package name in the catalog (Step 4) |
| `base_path` | `/data/oc-mirror` | Parent of the workspace, cache and private folders |
| `oc_mirror_tuning` | 4 images, 4 layers, 30 min per image, 5 retries | oc-mirror's own download settings |
| `oc_mirror_retries` | `3` | Extra full attempts after a failed run |
| `oc_mirror_retry_delay` | `120` | Seconds to wait between attempts |

## Step 9 — Verify the mirror

A `failed=0` recap is the first check. These confirm the result:

1. The auth file is gone. This must print nothing:

   ```bash
   ls -A /data/oc-mirror/private/
   ```
2. The release, the update graph and the operator catalog are in the registry. Use your host and path from Step 5:

   ```bash
   REG_HOST=artifactory.internal.repo; REG_PATH=ocp4
   podman login $REG_HOST
   skopeo list-tags docker://$REG_HOST/$REG_PATH/openshift/release-images | jq -r '.Tags[]'
   skopeo list-tags docker://$REG_HOST/$REG_PATH/openshift/graph-image | jq -r '.Tags[]'
   skopeo list-tags docker://$REG_HOST/$REG_PATH/redhat/redhat-operator-index | jq -r '.Tags[]'
   podman logout $REG_HOST
   ```

   Expect `4.18.17-x86_64`, `latest` and `v4.18` respectively.
3. oc-mirror wrote the files a cluster admin applies in Step 10:

   ```bash
   ls /data/oc-mirror/workspace/working-dir/cluster-resources/
   ```

   | File | What it is |
   | --- | --- |
   | `idms-oc-mirror.yaml`, `itms-oc-mirror.yaml` | Tell the cluster to pull from your registry instead of Red Hat's |
   | `cs-redhat-operator-index-*.yaml` | CatalogSource for the mirrored operator catalog |
   | `signature-configmap.yaml` | Release signatures, so the cluster can verify 4.18.17 |
   | `updateService.yaml` | The `UpdateService` resource, already pointing at your mirrored release and graph image |
4. Copy the whole `cluster-resources` folder and the log file from Step 8 to wherever your team hands over to the cluster admin.

## Step 10 — Set up the OpenShift Update Service on the cluster

This step changes the disconnected cluster, so a cluster admin does it in an approved change window. When it's done, the cluster reads its update graph from your mirror and `oc adm upgrade` lists 4.18.17.

You need the `cluster-resources` folder from Step 9, the registry's CA certificate (`ca.crt`) and `cluster-admin` access.

1. **Let the cluster pull from your registry.** The cluster's global pull secret must include the registry's login. If it already pulls images from this registry, skip this. Otherwise ask your lead; the global pull secret is shared by every node, so a change to it is its own reviewed change.
2. **Trust the registry's CA.** One ConfigMap serves both the image pulls (key = registry host; use `host..port` if there's a port) and the Update Service (key `updateservice-registry`). Check whether one is already set first:

   ```bash
   oc get image.config.openshift.io cluster -o jsonpath='{.spec.additionalTrustedCA.name}{"\n"}'
   ```

   If that prints a name, add the two keys to that ConfigMap instead of creating a new one. If it prints nothing:

   ```bash
   oc create configmap registry-ca -n openshift-config \
     --from-file=updateservice-registry=ca.crt \
     --from-file=artifactory.internal.repo=ca.crt
   oc patch image.config.openshift.io cluster --type merge \
     -p '{"spec":{"additionalTrustedCA":{"name":"registry-ca"}}}'
   ```
3. **Apply the mirror resources** generated by oc-mirror:

   ```bash
   cd cluster-resources
   oc apply -f idms-oc-mirror.yaml -f itms-oc-mirror.yaml
   oc apply -f signature-configmap.yaml
   oc apply -f cs-redhat-operator-index-*.yaml
   oc get catalogsource -n openshift-marketplace
   ```

   Note the CatalogSource name from the last command (it starts with `cs-redhat-operator-index`). Wait until `oc get packagemanifests | grep -iE 'cincinnati|update-service'` shows the operator.
4. **Install the Update Service operator.** Replace `<catalogsource-name>`, and the package name if Step 4 found a different one:

   ```bash
   oc create namespace openshift-update-service
   cat <<'EOF' | oc apply -f -
   apiVersion: operators.coreos.com/v1
   kind: OperatorGroup
   metadata:
     name: update-service-operator-group
     namespace: openshift-update-service
   spec:
     targetNamespaces:
       - openshift-update-service
   ---
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: update-service-subscription
     namespace: openshift-update-service
   spec:
     channel: v1
     installPlanApproval: Automatic
     name: cincinnati-operator
     source: <catalogsource-name>
     sourceNamespace: openshift-marketplace
   EOF
   oc get csv -n openshift-update-service -w
   ```

   Wait until the operator's CSV shows `Succeeded`, then press `Ctrl+c`.
5. **Create the Update Service.** oc-mirror already filled in the release and graph image paths:

   ```bash
   oc apply -f updateService.yaml
   oc get updateservice -n openshift-update-service
   oc get pods -n openshift-update-service
   ```

   Wait until the pods are `Running`.
6. **Point the cluster at it** and set the EUS channel:

   ```bash
   NS=openshift-update-service; NAME=update-service-oc-mirror
   GRAPH_URI="$(oc -n $NS get updateservice $NAME \
     -o jsonpath='{.status.policyEngineURI}/api/upgrades_info/v1/graph')"
   echo "$GRAPH_URI"
   oc patch clusterversion version --type merge \
     -p "{\"spec\":{\"upstream\":\"${GRAPH_URI}\"}}"
   oc adm upgrade channel eus-4.18
   oc adm upgrade
   ```

   `oc adm upgrade` should list `4.18.17` as an available update, or report the cluster is already on it. Applying the update itself is a separate change.

To confirm the Update Service is serving a graph (Red Hat's own check), ask it for the channel directly. It should return `200` and a JSON graph listing your mirrored versions:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -H 'Accept: application/json' "${GRAPH_URI}?channel=eus-4.18"
```

The graph only lists releases that are in your registry, so `oc adm upgrade` offers only what you mirrored.

## Mirroring an upgrade path: 4.18.17 → 4.20 → 4.22

To take the cluster from 4.18.17 to 4.20.x and then 4.22.x on EUS channels, mirror **two ranges in two runs** into the same registry, each limited to the update path. One channel with `minVersion: 4.18.17` and `maxVersion: 4.22.x` does not work.

**Why this is urgent:** 4.18's maintenance support ended on 25 August 2026. The cluster is now on 4.18's EUS term, which ends on **25 February 2027**. Both jumps should be done before then.

| Version | GA | Maintenance support ends | EUS ends | Latest z-stream (as of 1 Oct 2026) |
| --- | --- | --- | --- | --- |
| 4.18 | 25 Feb 2025 | 25 Aug 2026 | 25 Feb 2027 | 4.18.56 |
| 4.20 | 21 Oct 2025 | 21 Apr 2027 | — | 4.20.40 |
| 4.22 | 9 Jun 2026 | 31 Dec 2027 | — | 4.22.16 |

Source: [endoflife.date](https://github.com/endoflife-date/endoflife.date/blob/master/products/red-hat-openshift.md); confirm against Red Hat's [life cycle policy](https://access.redhat.com/support/policy/updates/openshift). 4.18.17 is 39 z-streams behind 4.18.56, so the update path may first go to a later 4.18 release. The dry-run in item 3 shows this, and `shortestPath` mirrors it.

### Why one channel can't do it

1. **oc-mirror only mirrors versions that are in the channel you name.** It walks the update graph of that one channel from min to max. An EUS channel reaches back to the previous EUS release: Red Hat's own oc-mirror v2 example uses `eus-4.14` with `minVersion: 4.12.28`. So `eus-4.20` can start at 4.18.17, but `eus-4.22` starts at 4.20 and doesn't contain 4.18. Check this with the command in item 1 below.
2. **The path goes through 4.19 and 4.21.** An EUS-to-EUS update still takes the control plane through the odd version in between (Red Hat: "updating from 4.20 to 4.22 goes through 4.21"). Only worker nodes can skip it, by pausing their machine config pool. So the mirror needs the 4.19.z and 4.21.z hop releases too.
3. **Two EUS channels in one ImageSet triggers a known oc-mirror v2 bug** (cross-channel upgrade calculation errors). Red Hat's workaround is to split the ImageSet, so this takes two separate runs.

### The two runs

| Run | Channel | minVersion | maxVersion | Covers |
| --- | --- | --- | --- | --- |
| A | `eus-4.20` | `4.18.17` | `4.20.<z>` | 4.18.17 → 4.19 hop(s) → 4.20 target |
| B | `eus-4.22` | `4.20.<z>` | `4.22.<z>` | 4.20 target → 4.21 hop(s) → 4.22 target |

![Upgrade path mirroring: two runs, two EUS channels](images/upgrade-path-runs.png)

The 4.20 target ends Run A and starts Run B, so the two mirrors join without a gap.

- **`shortestPath: true`** (`-e shortest_path=true`): oc-mirror mirrors only the releases on the update path, including 4.18.17 itself. Without it, it mirrors every z-stream release in the range, which could be dozens of releases.
- **Pin exact maximum versions, not "latest".** A pinned target gives the same result every run, and Run B's minimum must equal Run A's maximum.
- **One workspace per run, one shared cache.** Each run writes its own cluster resources, and shared image layers download once.
- **Same pull secret for both runs.** The Red Hat pull secret from Step 4 covers `quay.io` (release images) and `registry.redhat.io` (operator catalog and the Update Service operator); no separate Quay account is needed. `registry.access.redhat.com` (the graph's base image) is public.

### Steps

1. **Pick the versions.** List what each channel contains:

   ```bash
   for ch in eus-4.20 eus-4.22; do echo "== $ch"
     curl -s -H 'Accept: application/json' \
       "https://api.openshift.com/api/upgrades_info/v1/graph?channel=$ch&arch=amd64" \
       | jq -r '.nodes[].version' | sort -V | tr '\n' ' '; echo; done
   ```

   Confirm `4.18.17` is in `eus-4.20`. Pick a 4.20 target that is in **both** lists (it's Run A's max and Run B's min) and a 4.22 target from `eus-4.22`. Agree both with your lead and write them down.

   If `eus-4.22` comes back empty, Red Hat hasn't opened it yet. A new EUS channel usually appears 45–90 days after the version's GA. For 4.22, GA was 9 June 2026, past that window, so the eus-4.22 channel should be open by now. Mirror Run A now and Run B later.
2. **Dry-run both ranges.** Nothing is copied; oc-mirror writes the list of what it would copy. Replace the `<z>` values and your registry values:

   ```bash
   cd ~/oc-mirror-ansible
   REG="-e target_registry_fqdn=artifactory.internal.repo -e target_registry_path=ocp4"
   
   ansible-playbook playbook.yml --ask-vault-pass $REG \
     -e ocp_channel=eus-4.20 -e ocp_min_version=4.18.17 -e ocp_max_version=4.20.<z> \
     -e shortest_path=true -e oc_mirror_dry_run=true \
     -e workspace_path=/data/oc-mirror/workspace-run-a
   
   ansible-playbook playbook.yml --ask-vault-pass $REG \
     -e ocp_channel=eus-4.22 -e ocp_min_version=4.20.<z> -e ocp_max_version=4.22.<z> \
     -e shortest_path=true -e oc_mirror_dry_run=true \
     -e workspace_path=/data/oc-mirror/workspace-run-b
   ```
3. **Read the dry-run results.** For each run, list the releases it will mirror and check nothing is missing:

   ```bash
   for r in a b; do d=/data/oc-mirror/workspace-run-$r/working-dir/dry-run
     echo "== run $r"; grep -oE 'release-images:[0-9][^ =]*' $d/mapping.txt | sort -uV
     wc -l < $d/mapping.txt; cat $d/missing.txt 2>/dev/null; done
   ```

   Run A should list 4.18.17, at least one **4.19** release and your 4.20 target; Run B your 4.20 target, at least one **4.21** release and your 4.22 target. It may also list a later 4.18.z or 4.20.z if the graph needs one first.

   **Stop if a run lists only its min and max with no odd-version release in between.** That means the update graph has no path between them: oc-mirror then mirrors just the two ends, and the cluster couldn't update. Pick different versions, for example a later 4.20 target, and dry-run again.
4. **Size the disk.** The 100 GB in "Before you start" is for a single release. Several releases need far more. Use the release count and image count from item 3 to agree the `/data` size with your lead, and grow it before the real runs.
5. **Run for real**, in tmux (Step 7), Run A first. Use the same commands as item 2 without `-e oc_mirror_dry_run=true`. Check `df -h /data` between runs.
6. **Prepare both `cluster-resources` folders for handover** (`workspace-run-a` and `workspace-run-b`):
   - **Signatures:** each run writes a ConfigMap named `mirrored-release-signatures`, as both `signature-configmap.yaml` and `signature-configmap.json`, holding only that run's release signatures. Applying Run B's would replace Run A's. Rename Run A's and delete its JSON copy, so applying the folder can't bring the old name back. The cluster finds signature ConfigMaps by their label, not their name:

     ```bash
     cd /data/oc-mirror/workspace-run-a/working-dir/cluster-resources
     sed -i 's/name: mirrored-release-signatures$/name: mirrored-release-signatures-run-a/' signature-configmap.yaml
     rm -f signature-configmap.json
     grep 'name: mirrored-release-signatures' signature-configmap.yaml
     ```
   - **IDMS and ITMS:** both runs use the same resource names (for example `idms-release-0`), so Run B's replace Run A's when applied. They map whole repositories, so Run B's should cover everything Run A's did. Check before handover; this should print nothing:

     ```bash
     cd /data/oc-mirror
     diff <(grep -h 'source:' workspace-run-a/working-dir/cluster-resources/i*ms-*.yaml | sort -u) \
          <(grep -h 'source:' workspace-run-b/working-dir/cluster-resources/i*ms-*.yaml | sort -u) | grep '^<'
     ```

     If it prints lines, apply Run A's IDMS/ITMS under new names too, and tell your lead.
   - **Update Service:** use `updateService.yaml` from Run B. Run B also refreshes `openshift/graph-image`.
   - **Operator catalogs:** Run A mirrors the Update Service operator from the v4.18 catalog and Run B from v4.20. Mirroring the operator once is enough, but each cluster version will want its own catalog (v4.20, v4.22) for any operators you run. That's a separate mirror.
7. **Update the cluster, one EUS jump at a time** (cluster admin, own change). Red Hat's Control Plane Only procedure, for each jump (4.18 → 4.20, then 4.20 → 4.22):
   1. Update the `oc` client to the target version, and installed operators to versions that support it.
   2. Check all machine config pools show `UPDATED` and none `UPDATING`: `oc get mcp`.
   3. Switch to the **target** EUS channel while still on the old version: `oc adm upgrade channel eus-4.20` (on 4.18), later `eus-4.22` (on 4.20).
   4. Pause the worker pool: `oc patch mcp/worker --type merge --patch '{"spec":{"paused":true}}'`.
   5. Run `oc adm upgrade --to-latest` to the odd hop (4.19 or 4.21), check with `oc adm upgrade`, then run `oc adm upgrade --to-latest` again to the EUS target.
   6. Unpause the worker pool so workers update. Workers must catch up before the control plane jumps again: control plane and workers can be at most two minor versions apart.

This uses the standard update method. The 4.22 release notes list a known issue for the separate *image-based* upgrade method: EUS-to-EUS image-based upgrades aren't supported on clusters with cert-manager installed ([OCPBUGS-86967](https://issues.redhat.com/browse/OCPBUGS-86967)). If your team uses image-based upgrades, check this first.

### If eus-4.22 isn't open yet: stable channels for the second jump

Your team has agreed that the second jump may use stable channels if `eus-4.22` isn't available. Run A stays the same; Run B splits into two runs, one per stable channel. Each run still names one channel only, so the two-EUS-channel bug doesn't apply.

| Run | Channel | minVersion | maxVersion | Covers |
| --- | --- | --- | --- | --- |
| A | `eus-4.20` | `4.18.17` | `4.20.<z>` | unchanged |
| B1 | `stable-4.21` | `4.20.<z>` | `4.21.<y>` | 4.20 target → a 4.21 release |
| B2 | `stable-4.22` | `4.21.<y>` | `4.22.<z>` | that 4.21 release → 4.22 target |

1. **Pick the versions.** Run the listing command from item 1 above with `for ch in stable-4.21 stable-4.22`. Your 4.20 target must be in `stable-4.21`, and the 4.21 release you pick must be in **both** `stable-4.21` and `stable-4.22`. The cluster gets no update recommendations on a channel that doesn't include its current version, so this matters on the cluster as well as for the mirror.
2. **Dry-run B1 and B2**, each with its own workspace. B1 skips the Update Service content; B2 refreshes the graph last:

   ```bash
   ansible-playbook playbook.yml --ask-vault-pass $REG \
     -e ocp_channel=stable-4.21 -e ocp_min_version=4.20.<z> -e ocp_max_version=4.21.<y> \
     -e shortest_path=true -e mirror_update_service=false -e oc_mirror_dry_run=true \
     -e workspace_path=/data/oc-mirror/workspace-run-b1
   
   ansible-playbook playbook.yml --ask-vault-pass $REG \
     -e ocp_channel=stable-4.22 -e ocp_min_version=4.21.<y> -e ocp_max_version=4.22.<z> \
     -e shortest_path=true -e oc_mirror_dry_run=true \
     -e workspace_path=/data/oc-mirror/workspace-run-b2
   ```

   Check the results as in item 3 (use `for r in a b1 b2`). Each run should list its min and max, plus any releases in between that the graph needs.
3. **Run for real** in order A, B1, B2, without `-e oc_mirror_dry_run=true`.
4. **Handover** follows item 6 with three folders. Rename the signature ConfigMap in **every folder except the last one** (Run A to `mirrored-release-signatures-run-a`, Run B1 to `-run-b1`), delete their JSON copies, and run the IDMS/ITMS `diff` for A against B2 and B1 against B2. Use `updateService.yaml` from B2.
5. **On the cluster**, the 4.20 → 4.22 jump becomes two ordinary updates:
   1. On 4.20: `oc adm upgrade channel stable-4.21`, check `oc adm upgrade` lists your 4.21 release, then update to it.
   2. On 4.21: `oc adm upgrade channel stable-4.22`, then update to your 4.22 target.
   3. Once `eus-4.22` opens, switch to it (`oc adm upgrade channel eus-4.22`) so future updates follow EUS.

   Red Hat documents the Control Plane Only update (workers paused across both hops) with the EUS channel. With stable channels, plan for workers to update at each hop unless your lead approves pausing them; they must never fall more than two minor versions behind the control plane.

## Troubleshooting

First find which task failed: it's the task name just above `fatal:` in the log. Then find the row below.

| Failing task or symptom | Likely cause | Fix |
| --- | --- | --- |
| `Load Red Hat pull secret` fails to decrypt | Wrong Vault password, or the two Vault files use different passwords | Use the password from the password manager; re-create the file with the right one |
| `Load Red Hat pull secret` fails to parse JSON | Paste was cut or has extra text | `ansible-vault edit vars/redhat-pull-secret.json` and paste again (Step 4) |
| `Check the pull secret` fails | Secret missing `quay.io` or `registry.redhat.io` | Copy it again from the console (Step 4) |
| oc-mirror says `unauthorized` on a Red Hat registry | Secret reset in the console, or the account lost its subscription | Copy a fresh secret (Step 4); ask your lead about the account |
| oc-mirror says `unauthorized` or `denied` pushing to your registry | Login scope doesn't match the account's rights | Set `target_auth_key` to host + path (Step 5, Step 8) |
| Push fails with `name invalid` or path errors | Repository doesn't allow nested paths, or `target_registry_path` is wrong | Check the path with the admin (Step 5) |
| oc-mirror fails building the graph image | `api.openshift.com` or `registry.access.redhat.com` blocked by the proxy | Ask the network team to allow both (Step 1) |
| oc-mirror says the operator package isn't found | Package name differs in the catalog | Check the name (Step 4) and pass `-e osus_operator_package=<name>` |
| oc-mirror errors about cross-channel upgrades | Two EUS channels in one ImageSet | Mirror one channel per run |
| `Execute oc-mirror v2` fails on `localhost:55000` | `localhost` or `127.0.0.1` missing from `NO_PROXY` | Add both (Step 1) |
| `Execute oc-mirror v2` times out pulling from `quay.io` or `registry.redhat.io` | Proxy too slow or dropping long downloads | Tune, see below |
| `Execute oc-mirror v2` times out pushing to Artifactory | Registry host missing from `NO_PROXY`, or upload limit on the registry's load balancer | Check `NO_PROXY`; ask the Artifactory admin (Step 5) |
| `x509: certificate signed by unknown authority` | Proxy or registry CA not trusted | Add the CA (Step 1) |
| `unknown flag` from oc-mirror | Your build lacks one of the tuning flags | Remove that flag from `oc_mirror_tuning` |
| Works by hand, fails in cron, CI or AAP | Proxy variables not set there | Set them in the job (Step 1) |
| `No space left on device` | `/data` full | Free space or grow the volume; the cache in `/data/oc-mirror/cache` can be large |
| Update Service pods not ready (Step 10) | Cluster can't pull the graph image, or the `updateservice-registry` CA key is missing | Check pull secret and CA ConfigMap (Step 10, items 1–2) |
| `oc adm upgrade` shows no updates or a certificate error (Step 10) | Upstream URL wrong, or the cluster doesn't trust its own route's certificate | Re-run item 6; ask your lead about the ingress CA |
| Dry-run lists only the min and max release, no 4.19 or 4.21 in between | No update path between them in that channel's graph | Pick different versions and dry-run again (upgrade path, item 3) |
| eus-4.22 not listed, or channel graph empty | Red Hat still rolling out the new EUS channel (usually 45–90 days after GA) | Mirror Run A now, Run B when the channel appears |
| Push fails on tags like sha256-….sig, or the registry rejects signature artifacts | Newer oc-mirror v2 mirrors sigstore image signatures by default; the registry doesn't accept them | Ask the admin to allow them, or add --remove-signatures to oc\_mirror\_tuning. Check with your lead first if the cluster enforces sigstore image policies, which need these signatures |
| "Instructed to preserve digests" when mirroring the operator catalog | Older oc-mirror and a registry without OCI support (fixed in the 4.22 oc-mirror) | Download the latest oc-mirror (Step 3, item 4) |

### When oc-mirror times out

A rerun is safe. oc-mirror reuses its cache and skips images already copied, so each run picks up where the last stopped. If retries keep failing, change one setting at a time, from the top:

1. Rerun once as is. Brief proxy trouble often clears on its own.
2. Lower parallel downloads: `--parallel-images 2 --parallel-layers 2`.
3. Raise the per-image timeout: `--image-timeout 60m`.
4. Give the proxy more time between attempts: `-e oc_mirror_retry_delay=600`.
5. Ask the network team whether the proxy limits connection time or size for `quay.io` and `registry.redhat.io`, and for an exemption from the mirror host.

To change the tuning for one run without editing the playbook:

```bash
ansible-playbook playbook.yml --ask-vault-pass \
  -e "target_registry_fqdn=artifactory.internal.repo" -e "target_registry_path=ocp4" \
  -e "oc_mirror_tuning='--parallel-images 2 --parallel-layers 2 --image-timeout 60m --retry-times 8'"
```

For more detail, add `-vvv`. The secret tasks stay hidden by `no_log`; to debug them, remove `no_log` only while testing with a throwaway credential.

## Security notes and limitations

- **The Red Hat pull secret is a company credential.** It can pull every Red Hat image the account is entitled to. Keep it only in `vars/redhat-pull-secret.json` (Vault-encrypted), never in the disconnected cluster, a ticket or chat.
- **Credentials are on disk during the run.** oc-mirror reads auth from a file, so the merged secret sits in `/data/oc-mirror/private` (mode `0700`) until the `always:` block deletes it.
- **A killed run leaves the file.** A host crash or `kill -9` skips the cleanup. After any aborted run, check `ls -A /data/oc-mirror/private/` and delete what's there.
- **Never commit the Vault files unencrypted.** If the project goes into Git, check `head -1` of both files shows `$ANSIBLE_VAULT` before committing, and keep the Vault password out of the repo.
- **One release per run.** The ImageSet mirrors only the platform release. Operators and other images need more entries in the template.
- **Proxy credentials.** If the proxy needs a login, `HTTPS_PROXY=http://user:pass@proxy:8080` exposes it to every process on the host. Prefer an IP-allowlisted proxy for the mirror host.

## Sources

Checked against the upstream source code (read directly):

- [openshift/oc-mirror](https://github.com/openshift/oc-mirror): v2 flags (`--parallel-images` and `--parallel-layers` 1–10, `--image-timeout` default 10m, `--retry-times` default 5, `--retry-delay` unset = exponential backoff, `--cache-dir`); graph image `openshift/graph-image` built from `ubi9` plus graph data from `api.openshift.com`; release paths `openshift/release-images` and `openshift/release`; generated `updateService.yaml`, `idms-oc-mirror.yaml`, `itms-oc-mirror.yaml`, `signature-configmap.yaml`; [example ImageSet with `graph: true`](https://github.com/openshift/oc-mirror/blob/main/docs/image-set-examples/image-set-config.yaml)
- [openshift/cincinnati-operator](https://github.com/openshift/cincinnati-operator): `UpdateService` fields (`releases`, `graphDataImage`, `status.policyEngineURI`), bundle channel `v1`, and the [`updateservice-registry` CA key](https://github.com/openshift/cincinnati-operator/blob/master/docs/external-registry-ca.md)

Checked against Red Hat's documentation source, [openshift/openshift-docs](https://github.com/openshift/openshift-docs/tree/enterprise-4.20/modules) branch `enterprise-4.20` (read directly):

- `update-service-install-cli.adoc`: OperatorGroup and Subscription (`cincinnati-operator`, channel `v1`)
- `update-service-configure-cvo.adoc` and `update-service-create-service-cli.adoc`: `policyEngineURI` + `/api/upgrades_info/v1/graph`, the upstream patch, the graph check
- `config-access-for-sec-reg-osus.adoc`: `updateservice-registry` key and `host..port` form
- `oc-mirror-image-set-config-examples.adoc`: EUS example (`eus-4.14`, min `4.12.28`, `shortestPath: true`, `graph: true`)
- `oc-mirror-updating-cluster-manifests-v2.adoc`: apply `cluster-resources` and `signature-configmap`
- `oc-mirror-imageset-config-parameters-v2.adoc`, `oc-mirror-v2-about-dry-run.adoc`, `oc-mirror-proxy-support.adoc`: `shortestPath`, dry-run files, system proxy
- `understanding-update-channels.adoc`, `updating-control-plane-only-update-cli.adoc`, `core-cluster-upgrade-eus-control-plane-only.adoc`: EUS channels, the Control Plane Only steps, the 45–90 day EUS rollout, 4.20 → 4.22 through 4.21

Checked against the [OpenShift 4.22 release notes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/release_notes/ocp-4-22-release-notes), read from their source on the [`enterprise-4.22` branch](https://github.com/openshift/openshift-docs/tree/enterprise-4.22/release_notes):

- 4.22.1 issued 16 June 2026; newest listed, 4.22.16, issued 29 September 2026
- oc-mirror v1, `ImageContentSourcePolicy` and `oc adm release mirror` are deprecated; use oc-mirror v2 (this guide does)
- New in the 4.22 oc-mirror v2: `oc mirror list --v2`, digest-pinned operator catalogs, and fixes for non-OCI registries and sigstore settings
- Known issue: EUS-to-EUS image-based upgrades unsupported with cert-manager (OCPBUGS-86967)
- From the 4.22 docs modules: "Use the latest available version of the oc-mirror plugin v2 regardless of which versions of OpenShift Container Platform you need to mirror"; signature mirroring is on by default and `--remove-signatures` turns it off
- Unchanged from 4.20: the cluster-resources apply procedure, the EUS ImageSet example and the Control Plane Only procedure

Red Hat pages found by search but blocked from the environment this guide was written in. Their content was checked through the documentation source on GitHub instead (above); the knowledge-base articles were seen by title only:

- [Updating a cluster in a disconnected environment (OCP 4.18)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/disconnected_environments/updating-a-cluster-in-a-disconnected-environment)
- [Mirroring for a disconnected installation (OCP 4.18)](https://docs.redhat.com/ko/documentation/openshift_container_platform/4.18/html/disconnected_environments/installing-mirroring-disconnected)
- [oc-mirror v2 fails when two EUS channels are in the ImageSetConfig](https://access.redhat.com/node/7099908)
- [OCP 4.18 release notes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/release_notes)
- [oc mirror does not mirror intermediate versions on EUS channels with shortestPath](https://access.redhat.com/node/7061405): in current oc-mirror v2 this happens when the graph has no path between min and max (the code then mirrors only the two ends), which the dry-run check in the upgrade-path section catches
