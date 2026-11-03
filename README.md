# nova-libvirt-swtpm

Kolla's 2025.1 `nova-libvirt` image with one fix: `/var/lib/swtpm-localca` is
owned by `tss`, so instances with a software vTPM (`hw:tpm_version`) can boot.

```
ghcr.io/atlasdigitalengineering/nova-libvirt-swtpm:<version>
```

## The bug

With `swtpm_enabled = True` in nova.conf, every TPM instance fails to build on
every compute host with `Exhausted all hosts available for retrying`. The
compute log shows:

```
libvirt.libvirtError: internal error: Could not run '/usr/bin/swtpm_setup'. exitstatus: 1
```

and the swtpm log inside the `nova_libvirt` container
(`/var/log/swtpm/libvirt/qemu/<instance>-swtpm.log`) says why:

```
Starting vTPM manufacturing as tss:tss
Need read/write rights on statedir /var/lib/swtpm-localca for user tss.
swtpm_localca exit with status 1: An error occurred.
```

Barbican is not the problem: the vTPM secret is created successfully before
this point.

## Cause

Two defaults that do not agree:

| Component | Behaviour |
|---|---|
| Ubuntu Noble `swtpm` package | creates `/var/lib/swtpm-localca` owned by `swtpm` (Ubuntu moved swtpm off `tss`, [LP#1948880](https://bugs.launchpad.net/bugs/1948880)) |
| libvirt in Kolla 20.x | runs swtpm as libvirt's default user, `tss`, because Kolla's `qemu.conf` sets no `swtpm_user` |

The config-only fix (`swtpm_user = "swtpm"` in qemu.conf, matched by
`swtpm_user`/`swtpm_group` in nova.conf) is **not available in kolla-ansible
20.x**: its nova-cell role renders `qemu.conf` from the built-in template with
no custom override path. Later releases add a `qemu.conf.j2` override; this
image exists only until the deployment can use it.

## The fix

[`images/nova-libvirt/Dockerfile`](images/nova-libvirt/Dockerfile) builds on the
upstream image and chowns the directory to `tss`, then proves `tss` can write to
it. A base image that changes the user or the package layout fails the
**build**, not a vTPM boot on a hypervisor.

Base image facts, read from the image's own config on a blade rather than
assumed:

| Fact | Value |
|---|---|
| Base | `quay.io/openstack.kolla/nova-libvirt:2025.1-ubuntu-noble` |
| Digest | `sha256:1e4c5a28d3a313253b02f76dde4782e2ccea602292a00e3f525cf5cd0fbfee3a` |
| `kolla_version` label | `20.6.0` (built 2026-09-09) |
| User | unset, so root |
| Entrypoint / Cmd | `dumb-init --single-child --` / `kolla_start` |

## Deploying

1. Leave nova.conf at `swtpm_enabled = True` **only**. Do not set `swtpm_user`
   or `swtpm_group`: they must match the user libvirt actually runs swtpm as,
   which with this image is the default, `tss`.
2. Pin the image in the environment's `globals.d/02-compute.yml`, **not**
   `globals.yml` (globals.d files are passed after globals.yml, and the last
   `-e` wins):

   ```yaml
   nova_libvirt_image_full: "ghcr.io/atlasdigitalengineering/nova-libvirt-swtpm:<version>"
   ```

3. `kolla-ansible deploy -i <inventory> -t nova`. `reconfigure` does not
   recreate containers from a new image.
4. Verify on each compute host:

   ```bash
   docker exec nova_libvirt ls -ld /var/lib/swtpm-localca   # owned by tss
   ```

   then boot a test instance on a TPM flavor and check `/dev/tpm0` exists in
   the guest. Its swtpm log should still say `tss:tss` but get past
   `swtpm_localca`.

## Releasing

Push a `vX.Y.Z` tag. The [Image workflow](.github/workflows/image.yml) builds on
the pinned base, pushes `ghcr.io/atlasdigitalengineering/nova-libvirt-swtpm:X.Y.Z`
and creates a GitHub release with the `globals.d` line. Pull requests build
without pushing.

To pick up a newer base image without changing anything here, run the workflow
manually (`workflow_dispatch`) against the existing tag.

### The first publish creates a private package

A package first pushed with `GITHUB_TOKEN` is private even though the repository
is public, and the blades pull anonymously. After the first release, set the
package to public: the package page on GitHub, **Package settings**, **Change
visibility**. The workflow warns if it is not anonymously pullable.

## Retiring this image

Delete the `nova_libvirt_image_full` pin once the deployment is on a
kolla-ansible release whose nova-cell role reads a custom `qemu.conf.j2`. Then
set the user in config instead:

- `config/nova/qemu.conf.j2`: Kolla's template plus `swtpm_user = "swtpm"` and
  `swtpm_group = "swtpm"`
- `config/nova.conf`, `[libvirt]`: `swtpm_user = swtpm`, `swtpm_group = swtpm`

## License

GPL-3.0. See [LICENSE](LICENSE).
