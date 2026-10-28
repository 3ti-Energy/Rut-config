# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An Ansible playbook that configures Teltonika RUT routers (RutOS, which is based on OpenWrt) at 3ti sites. Each inventory host is one site router (`3ti-NNNN`). The supported models are older RUT955s (firmware ≤ 7.15) and RUTM55s. All configuration goes through UCI using the `community.openwrt` collection.

## Commands

```sh
ansible-galaxy install -r requirements.yaml                                     # install the pinned collection
ansible-playbook --inventory hosts.yaml site.yaml --limit '3ti-00XX' --diff --check   # dry run against one site
ansible-playbook --inventory hosts.yaml site.yaml --limit '3ti-00XX' --diff           # real run (prompts before committing)
ansible-playbook site.yaml --syntax-check
```

There are no tests and no lint config. Always use `--limit` because a run without it targets every site. `community.openwrt` is pinned to 1.8.0 because newer versions need `ucode-mod-uci`, which RutOS doesn't ship. Don't upgrade it.

## How a run works (`site.yaml`)

1. **Credentials come from the site hub, not from the repo.** The first block runs on localhost (`delegate_to: localhost`). It calls `GET http://{{ IP_24 }}.100:5003/api/settings` on the site's hub and reads the wifi, root/admin and 3ti-admin passwords by `deviceTypeSettingID` (13, 11 and 12). This means the control machine must be able to reach the hub's LAN, normally over VPN. The router is then reached over SSH as `root` at `{{ IP_24 }}.254`.
2. **UCI changes are staged, not committed.** Task files mostly use `community.openwrt.uci` with `cmd: set|section|absent`, which only stage changes. At the end, the play shows `uci changes`, pauses for a yes/no answer, and then either runs `commit` (followed by the `reload_config` handler) or `revert`. `serial: 1` means this prompt happens once per router, one router at a time.
   - Raw `community.openwrt.command` tasks (e.g. `adduser` in `openwrt_users.yaml`) run **immediately** and skip this approval step. Use them sparingly.
   - The `command` module has no check-mode support, so read-only commands need `check_mode: false`. Otherwise `--check` skips them and later `when:` conditions fail on missing keys.
3. Task files are imported in this order: users → system → network → failover → sms → events.

## Inventory and variable conventions

- `IP_24` is the first three octets of the site LAN (/24). The address plan is fixed: `.254` is the router, `.100` is the hub (static lease from `HUB_MAC`, and `hub.3ti.co.uk` resolves to it locally for charge-point TLS), `.110` is CCTV (`CCTV_MAC`), `.101` is the host machine (`HOST_MAC`), and DHCP hands out `.50`–`.150`.
- These per-host vars are optional: `STATIC_IP` and `GATEWAY` (static WAN), `wired`, `no_sims`, and `default_sim`. Their defaults in `group_vars/all/vars.yaml` are `wired: true`, `no_sims: 1` and `default_sim: 2`.
- `site.yaml` sets `ansible_host` as a play var, which takes precedence over inventory vars. So setting `ansible_host` on a host in `hosts.yaml` has no effect.
- `group_vars/all/vault.yaml` is gitignored and holds the `vault_*` secrets (APN credentials and the SMTP password). Don't read it, print it or commit it. Tasks that handle secrets set `no_log: true`.

## Patterns to follow

- **Idempotent anonymous sections:** use `cmd: section` with a `find:` dict. The task matches an existing section by those keys and updates it, or creates one if there's no match.
- **Failover (`openwrt_failover.yaml`):** `wired`, `no_sims` and `default_sim` drive the mwan3 member metrics (wired WAN first if present, then the default SIM, then the other SIM), which interfaces are enabled, sim_switch, and switch-back to the primary SIM after 15 minutes.
- **Branching by model and firmware:**
  - The rpcd ACLs for the per-site admin user (`admin-<hostname>`) are only set when `ansible_facts.distribution_version is version('00.07.15', '<=')`.
  - Event reporting is defined once in the `openwrt_events` list. It's written to `event_juggler` on newer firmware, or to the legacy `events_reporting` service on RUT955s. The playbook picks one by checking `ansible_facts.openwrt_services`.
  - event_juggler sections are named by index: event `i*2+1` links to action `i*2+2`. Reordering the list renumbers these sections.
  - RUTM55 gets a second radio. It's detected with `ansible_facts.openwrt_board.model`.
- Inventory hostnames are used in UCI values (device name, hostname, SSID `<host>-staff`, admin username). Lowercase them where OpenWrt expects it.
