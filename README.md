# thehackingmachine — Build a vulnerable machine

Ansible playbooks that build deliberately vulnerable lab VMs for OSCP-style
practice. Each playbook stands up one CVE's vulnerable service on a target
VM, deliberately misconfigured/unpatched, and leaves it running — ready for
you to exploit manually. The playbooks don't fire the exploit themselves;
some include a non-exploit sanity check (e.g. "is the service up and on the
right version?") just to confirm the build worked.

## Requirements

- Ansible (controller machine)
- A target VM reachable over SSH, reverted to a clean snapshot before each
  build (see below)

## Layout

```
ansible.cfg              # inventory path + sane defaults (host key checking off, etc.)
inventory/hosts.ini       # single [machine] group, one host
base/
  bootstrap.yml           # one-time host prep, run against a fresh-os snapshot to produce clean-base
  verify.yml              # read-only check that the VM matches clean-base
cve/
  _guard.yml              # shared clean-base guard included by every CVE playbook
  cve-<id>.yml            # one playbook per CVE lab
  files/cve-<id>/         # static assets (systemd units, etc.) each playbook deploys
templates/
  cve-writeup.md          # template for a write-up accompanying a new CVE lab
```

## Workflow

1. **Provision a target VM** and point `inventory/hosts.ini` at it
   (`ansible_host`, `ansible_user`).

2. **Bootstrap it once**, against a freshly reverted snapshot:

   ```bash
   ansible-playbook base/bootstrap.yml
   # or, before passwordless sudo exists yet:
   ansible-playbook base/bootstrap.yml --ask-become-pass
   ```

   This installs common lab tooling, sets up passwordless sudo, brands the
   login banner/MOTD, disables `ufw`, and turns off unattended upgrades.
   Reboot afterward, confirm SSH still works, then take your **base
   snapshot** (`clean-base`) — per-CVE playbooks assume only this bootstrap
   has run and nothing else. `ansible-playbook base/verify.yml` confirms the VM
   matches it. (`fresh-os` is the earlier, pre-bootstrap
   snapshot you revert to before running `bootstrap.yml` again.)

3. **Build a CVE lab** by running its playbook against that base snapshot:

   ```bash
   ansible-playbook cve/cve-2021-41773.yml
   ```

   Each one installs build deps, builds the vulnerable service from source,
   applies the deliberately vulnerable config, deploys any files from
   `cve/files/<cve-id>/`, and starts/restarts the service. That's where it
   stops — no exploit is run as part of the playbook.

4. **Exploit it yourself**, manually, against the now-running target.

5. **Revert the VM** to the base snapshot before building a different CVE
   lab (or rebuilding the same one). This is enforced: each playbook first
   checks for a marker file, `/etc/lab-state`, on the VM and refuses to run
   if it exists, telling you which CVE is installed. The marker is written
   at the start of a build (so a failed build still counts as dirty) and
   disappears when you restore the snapshot. To build anyway while iterating
   on a playbook, pass `-e force=true`:

   ```bash
   ansible-playbook cve/cve-2021-41773.yml -e force=true
   ```

You can syntax-check any playbook without running it:

```bash
ansible-playbook cve/cve-2021-41773.yml --syntax-check
```

Each playbook's non-exploit sanity checks (service up, right version
reported, etc.) are tagged `verify`, so you can re-run just those against an
already-built target without repeating the full build:

```bash
ansible-playbook cve/cve-2021-41773.yml --tags verify
```

## Available CVE labs

| Playbook | CVE | Service |
|---|---|---|
| `cve/cve-2021-41773.yml` | CVE-2021-41773 | Apache HTTP Server 2.4.49 (path traversal / RCE via mod_cgi) |
| `cve/cve-2021-42013-lfi.yml` | CVE-2021-42013 | Apache HTTP Server 2.4.50 (double-encoded path traversal) |
| `cve/cve-2021-42013-rce.yml` | CVE-2021-42013 | Apache HTTP Server 2.4.50 (RCE via mod_cgi) |
| `cve/cve-2025-32023.yml` | CVE-2025-32023 | Redis 7.4.2 (HyperLogLog heap overflow RCE) |
| `cve/cve-2026-87902.yml` | CVE-2026-87902 | WordPress 7.1.1 (unauthenticated path traversal / LFI) |

## Adding a new CVE lab

Follow the shape of `cve/cve-2025-32023.yml`: install deps → build/install
the vulnerable version from source → apply the deliberately vulnerable
config → deploy a systemd unit from `cve/files/<cve-id>/` → start/restart
the service. Include the clean-base guard as the first step
(`pre_tasks` → `import_tasks: _guard.yml`, with `lab_cve` set to the CVE id). Don't add an exploit-and-assert step — that's done by hand
against the deployed VM.

If you're writing an accompanying public write-up, start from
`templates/cve-writeup.md` — it explains what's safe to publish (build +
verify steps) and what isn't (exploit code/PoCs).

## License

CC BY-NC-SA 4.0 — see [LICENSE](LICENSE).

- You're free to use, share, and adapt this content, including for your own labs or write-ups, as long as you give attribution, don't use it
  commercially, and share any adaptations under this same license.