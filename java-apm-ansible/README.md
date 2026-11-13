# New Relic Java APM — Ansible / AWX

**Status: starter implementation for NPROD validation, not production-approved.** Replace all sample values. See `docs/GO_LIVE_CHECKLIST.md`.

## Behavior
- One server has one application type and multiple Java services.
- One shared agent JAR on each server; each JVM has its own `setenv.sh`.
- Download the official current Java agent ZIP on each run; compare staged and installed JAR versions.
- Install/upgrade only when missing or older; never downgrade.
- Independently manage a New Relic block in every service `setenv.sh`.
- If agent, license configuration, or any `setenv.sh` changed: stop all, wait five seconds, start all, retry status.
- No restart on unchanged runs.

## Repository
- `site.yml`: playbook, serialized (`serial: 1`) to reduce fleet-wide outage risk.
- `vars/services.yml`: single source of truth for host, service, labels, scripts.
- `inventory/hosts.ini`: example inventory; AWX inventory can replace it.
- `roles/java_apm/tasks/`: validation, detection, installation, configuration, restart.
- `roles/java_apm/templates/`: minimal agent config and managed startup snippet.

## AWX setup
1. Create Git repository, push these files, and create an AWX Project synced to it.
2. Create inventory and hosts with names identical to `java_apm.servers` keys.
3. Add Machine credential (SSH/sudo) to run privileged tasks.
4. Create **custom credential type** with secret input `license_key` and extra-vars injection `nr_license_key: '{{ license_key }}'`. Configure separate PROD and NPROD credentials and attach the correct one to each job template. Ensure secret redaction.
5. Create a Job Template: Project, inventory, `site.yml`, execution environment with ansible-core and Python, Machine + New Relic credentials, privilege escalation enabled.
6. Restrict job inventory/limit to one environment; start with a single NPROD host.
7. Run syntax check, then a carefully scheduled live test. Do not use check mode as proof of correctness: download/version and shell startup behavior need real execution.

## Required environment
- Target: Linux, Python for Ansible modules, Java executable, `unzip`, HTTPS to approved artifact source.
- AWX: supported Ansible execution environment, SSH access, sudo rights, custom license credential.
- Scripts: executable by selected service user and return meaningful status codes.

## Known hardening points before production
1. **JVM option quoting**: the template uses quoted `NEW_RELIC_APP_NAME` and `NEW_RELIC_LABELS` exports to preserve spaces and only appends `-javaagent` to `JAVA_OPTS`. Confirm your startup scripts source `setenv.sh` and preserve those environment variables.
2. **Existing New Relic configs**: unmanaged `-javaagent` is rejected; preexisting `newrelic.yml` needs review for duplicate `license_key`, environment overrides, and compatibility. Agent file update preserves existing YAML.
3. **Release staging**: the official `current` ZIP is downloaded on each run to establish the actual latest version. For enterprise use, an approved internal artifact mirror is preferable. Confirm the staged agent supports every JVM on that server.
4. **Status checks**: status script must fail if any expected service is down. This role does not yet verify each service PID has an attached agent; add application-specific PID/health checks before production.
5. **Upgrade safety**: JAR is backed up before replacement; the role does not implement automatic rollback. Plan maintenance and recovery procedures. Existing `newrelic.yml` is preserved and not automatically merged with newly introduced vendor settings.
6. **License**: currently stored in `newrelic.yml` (root-owned, group-readable to service user) and not in Git. Validate existing config shape and key escaping before production.
7. **Failure recovery**: if start/status fails, job fails and operator must restore previous JAR/config and rerun startup. Test this scenario.

## Local validation
```bash
ansible-playbook -i inventory/hosts.ini site.yml --syntax-check
ansible-playbook -i inventory/hosts.ini site.yml --limit banner-ap01
```
Do not place real secrets in shell history. Supply them through AWX credentials.

## Reference
New Relic official Java install: https://docs.newrelic.com/docs/apm/agents/java-agent/installation/java-agent-installation-app-servers/
