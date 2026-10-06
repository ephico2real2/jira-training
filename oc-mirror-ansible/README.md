# oc-mirror v2 with Ansible

Mirrors OpenShift release images and operator catalogs into an internal registry for a
disconnected cluster, using oc-mirror v2 and Ansible. The Red Hat pull secret and registry
login come from Ansible Vault; the proxy comes from the mirror host.

| Guide | Covers |
| --- | --- |
| [docs/release-mirroring-guide.md](docs/release-mirroring-guide.md) | Host setup, proxy, Ansible Vault, release mirroring (4.18.17, EUS), OpenShift Update Service, upgrade path 4.18 → 4.20 → 4.22 |
| [docs/catalog-mirroring-guide.md](docs/catalog-mirroring-guide.md) | Operator catalogs (`v$OCP_VERSION`), inventory, package filtering, CatalogSources, catalogs across the upgrade path |

## Files

| File | Purpose |
| --- | --- |
| `playbook.yml` | Release images (+ update graph and Update Service operator) |
| `catalogs-playbook.yml` | Operator catalogs, one group and version per run |
| `templates/imageset-config.yaml.j2` | Release ImageSet |
| `templates/catalogs-imageset.yaml.j2` | Catalog ImageSet |
| `vars/catalogs-redhat.yml`, `vars/catalogs-partner.yml` | **Samples only** (IBM Fusion's sample setup); replace with your operator inventory |
| `vars/credentials.yml.example` | Shape of the registry login; create the real file with `ansible-vault create` |

Not in Git (see `.gitignore`): `vars/credentials.yml`, `vars/redhat-pull-secret.json`, the
`oc-mirror` binary. Create them on the mirror host as the release guide describes.

## Quick start

```bash
# Release images, dry-run first
ansible-playbook playbook.yml --ask-vault-pass -e oc_mirror_dry_run=true

# Red Hat operator catalog for 4.18, dry-run first
ansible-playbook catalogs-playbook.yml --ask-vault-pass \
  -e catalog_ocp_version=4.18 -e @vars/catalogs-redhat.yml -e oc_mirror_dry_run=true
```

Tested with stand-in `oc-mirror` and Vault files (rendering, retries, dry-run, auth-file
cleanup); not yet run against real registries.
