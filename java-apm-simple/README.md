# Java APM installation using Ansible

This repository installs or upgrades one New Relic Java agent per Linux server, updates every configured `setenv.sh`, and restarts all application services **once** if anything changed.

## 1. Edit the central file

Edit `vars/services.yml`:
- Replace every example hostname and path.
- Each server has **one** application (`SSB9`, `AppNav`, `BAM`, etc.) and a list of services.
- Each service needs an APM application name and an absolute `setenv.sh` path.
- Standard Banner servers use the common `app` restart scripts.
- Walden servers supply a `restart` override using `tomcat`; script names depend on the application type.
- Multiple servers may use the same application and service names.

The script names for standard Banner are **stop_apps.sh**, **start_app.sh**, **status_apps.sh**.

## 2. Confirm prerequisites

- Ansible controller/AWX execution environment with `ansible-core` and Python.
- Linux SSH access, privilege escalation, Java and `unzip` on target servers.
- Outbound HTTPS access from the managed host to the official New Relic Java download URL (or replace with an approved internal mirror).
- Correct Java version for the downloaded agent.
- A directory containing each service's `setenv.sh`; the role can create a missing file but not a missing parent directory.
- Each application startup script must actually source the configured `setenv.sh` and honor `JAVA_OPTS`.
- `app` or `tomcat` must be able to execute the configured stop/start/status scripts.
- Status script must return nonzero until **all** services on that server are running.
- The agent config file must use a `common:` section with a two-space-indented `license_key:` line; if not, review it manually.

## 3. Credentials

Provide `nr_license_key` securely from AWX. Do **not** store it in GitHub.

Create an AWX custom credential type with an input field `license_key` (secret) and this injector:

```yaml
extra_vars:
  nr_license_key: '{{ license_key }}'
```

Use separate PROD/NPROD job templates or credentials so the correct key is selected.

Use an AWX Machine credential for SSH and privilege escalation.

## 4. AWX

1. Push repository to a private GitHub repo.
2. Create an AWX Project using that repo.
3. Create an inventory with hosts matching `java_apm.servers` keys.
4. Create a Job Template pointing at `site.yml`.
5. Attach inventory, Machine credential and appropriate New Relic credential.
6. Run with a **single NPROD host limit** first.
7. Check New Relic APM for every expected JVM and application name.
8. Rerun: no JAR replacement, no `setenv.sh` change and no restart.

Local syntax check:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --syntax-check
```

## 5. What the playbook does

1. Validate paths and restart scripts.
2. Download latest official agent into a temporary directory and compare its version with installed JAR.
3. Install/upgrade only if needed; preserve an old JAR backup.
4. Preserve existing `newrelic.yml` settings except its license key.
5. Add/update only the managed New Relic block in each `setenv.sh`; preserve other Java settings.
6. If agent or configuration changed: stop all services, wait 5 seconds, start all services and retry status checks.
7. Otherwise skip restart.

## 6. Important limitations before production

- A version comparison requires downloading the latest ZIP on **every run**. It does **not** reinstall the JAR on every run.
- The latest agent may not support an old Java runtime. Check compatibility and use an approved release where necessary.
- This role does not automatically remove or migrate unmanaged `-javaagent` flags; it stops and requests review.
- It does not automatically roll back a failed agent upgrade.
- `status` exit code must reflect **all** services. The role cannot infer that from script output alone.
- If the existing `setenv.sh` sets `JAVA_OPTS` **after** the managed block, it may override the agent flag. Confirm startup order and JVM command lines.
- If `newrelic.yml` uses an unexpected structure or multiple license keys, adjust the role after inspection; do not blindly overwrite it.
- Some startup wrappers may require `CATALINA_OPTS` instead of `JAVA_OPTS`. Confirm for each application.
- Do not run production before testing fresh install, upgrade, missing config, and unchanged rerun on NPROD.
