# stratum-ansible-role-containerlab

Ansible role that bootstraps [Containerlab](https://containerlab.dev/) on a
nested-virtualisation GCE node (Rocky Linux 9).  Part of the **STRATUM platform**
factory layer — Phase 4 bootstrap (D-INFRA-30).

## What it does

1. Installs Docker CE (enables CRB and Docker CE repository via dnf).
2. Starts and enables the Docker service.
3. Downloads and installs the Containerlab CLI RPM from GitHub releases.
4. Creates the topology directory (`containerlab_topology_dir`) and subdirectories
   (`images/`, `configs/`).
5. Pulls vrnetlab OCI tarballs from GCS cache (`gs://<bucket>/images/<kind>/<tag>/*.tar`).
   Soft-skips with `ignore_errors: true` when the GCS path is empty (first run).
6. Loads OCI tarballs into Docker. Soft-skips when no tarballs are present.
7. Pulls startup configs from GCS (`gs://<bucket>/configs/<kind>/<tag>/`).
   Soft-skips when absent.
8. Templates the Containerlab topology YAML to
   `{{ containerlab_topology_dir }}/topology.clab.yml`.
9. Deploys the lab: `containerlab deploy -t topology.clab.yml --reconfigure`.
10. Verifies SSH reachability for each device on the Containerlab management network
    (`172.20.20.x`, port 22, timeout 300 s).

## Requirements

- Rocky Linux 9 target (EL 9 family).
- `gcloud` CLI authenticated on the target host (Application Default Credentials or
  a service-account key via `GOOGLE_APPLICATION_CREDENTIALS`).
- Internet access to download Docker CE packages and the Containerlab RPM (or an
  offline mirror configured before running the role).

## Role variables

All variables are defined in `defaults/main.yml`.

| Variable | Default | Required | Description |
|---|---|---|---|
| `containerlab_version` | `""` | yes | Containerlab release version (e.g. `"0.68.0"`). Used to build the GitHub download URL. |
| `containerlab_cache_bucket` | `""` | yes | GCS bucket name (without `gs://`) holding OCI image tarballs and startup configs. |
| `containerlab_topology_dir` | `/opt/stratum/containerlab` | no | Absolute path on the host for topology files, images, and configs. |
| `containerlab_devices` | see defaults | no | List of device definitions. Each entry: `kind`, `name`, `image_tag`. Optional `startup_config` key activates the startup-config topology field. |
| `containerlab_links` | see defaults | no | List of point-to-point links. Each entry: `endpoints` (two-element list of `"node:interface"` strings). |

## Example playbook

```yaml
- name: Bootstrap Containerlab
  hosts: containerlab_node
  become: true
  roles:
    - role: apellini.containerlab
      vars:
        containerlab_version: "0.68.0"
        containerlab_cache_bucket: stratum-dev-cache
        containerlab_devices:
          - kind: cisco_n9kv
            name: n9kv-1
            image_tag: "9.3.10"
          - kind: cisco_n9kv
            name: n9kv-2
            image_tag: "9.3.10"
        containerlab_links:
          - endpoints: ["n9kv-1:eth1", "n9kv-2:eth1"]
```

## Tags

| Tag | Description |
|---|---|
| `save_configs` | Run `tasks/save_configs.yml` to save running device configs and push them back to GCS. Invoked via `--tags save_configs`. |

## Save configs (end-of-session)

To save device running configs to GCS after a lab session:

```bash
ansible-playbook site.yml --tags save_configs
```

This runs `containerlab save` and then pushes each device's configs directory back
to `gs://<bucket>/configs/<kind>/<tag>/`.

## License

MIT

## Author

[apellini](https://github.com/apellini) — STRATUM platform
