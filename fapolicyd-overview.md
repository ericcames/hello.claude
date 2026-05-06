# FAPolicyD — File Access Policy Daemon

Userspace application allowlisting daemon on RHEL/Fedora. Gates which executables, scripts, and libraries can run, based on a trust database (RPM-installed files by default) plus rule files in `/etc/fapolicyd/rules.d/`. Uses the kernel's fanotify API. Common driver: meeting allowlisting controls in compliance frameworks (PCI, NIST 800-53, DISA STIGs).

## Learning / permissive mode

FAPolicyD does not auto-generate rules, but it has a permissive mode that serves as a learning mode — denials are logged but not enforced, so you can exercise a workload and see what *would* have been blocked.

### Workflow

1. **Switch to permissive.** Either:
   - Edit `/etc/fapolicyd/fapolicyd.conf` → `permissive = 1` and `systemctl restart fapolicyd`
   - Or run a foreground session: `fapolicyd --debug-deny --permissive`
2. **Exercise the workload.** Run the apps, scripts, services that must work in production.
3. **Review what would have been blocked:**
   - `/var/log/fapolicyd-access.log` (if `syslog_format` is enabled)
   - `ausearch -m FANOTIFY_DENY` from the audit log
4. **Add legitimate items to trust DB or rules:**
   - `fapolicyd-cli --file add /path/to/binary` — per-file trust
   - `fapolicyd-cli --list` — inspect trust DB
   - `fapolicyd-cli --update` — refresh the RPM-derived trust DB
   - Or write a custom rule under `/etc/fapolicyd/rules.d/`
5. **Flip back to enforcing.** `permissive = 0` in the config, then `systemctl restart fapolicyd`.

## Automation

Red Hat ships an Ansible role for fleet-wide deploy and configuration:

- `rhel-system-roles.fapolicyd` (RHEL)
- `linux-system-roles.fapolicyd` (upstream / Fedora)

Useful when rolling out allowlisting across many managed nodes from AAP.

## Public references

- [Red Hat — Blocking and allowing applications using fapolicyd (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/assembly_blocking-and-allowing-applications-using-fapolicyd_security-hardening)
- [Upstream project — linux-application-whitelisting/fapolicyd](https://github.com/linux-application-whitelisting/fapolicyd)
- [`fapolicyd.conf(5)` man page](https://man7.org/linux/man-pages/man5/fapolicyd.conf.5.html)
