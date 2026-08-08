# NetBox Upgrade Runbook

> Universal, GitHub-ready runbook for upgrading a native Linux installation of NetBox Community.
>
> **Scope:** Traditional NetBox deployments installed from a release archive or Git repository, using systemd and an external or local PostgreSQL database. Docker/Kubernetes deployments require a different procedure.

## 1. Purpose

This runbook provides a repeatable upgrade procedure that avoids environment-specific IP addresses, hostnames, credentials, plugin names, and version numbers.

It assumes that planned downtime is acceptable and that a reliable rollback method exists. VMware snapshots are used as the primary rollback example. If snapshots are unavailable, use a tested PostgreSQL backup/restore process instead.

## 2. Important Rules

1. Review every NetBox release note between the current and target versions before upgrading.
2. Confirm Python, PostgreSQL, Redis, and plugin compatibility with the target release.
3. Use the same installation method that was used originally: release archive or Git.
4. When crossing a **major NetBox version**, first upgrade to the latest minor release of the current major version before moving to the next major version.
5. Stop NetBox write activity before taking rollback snapshots.
6. After `upgrade.sh` runs database migrations, rollback must restore the database state as well as the application state.
7. Never place passwords, API tokens, Redis passwords, private keys, or `SECRET_KEY` values in this runbook or in shell history.

Official references:

- NetBox upgrade guide: https://netboxlabs.com/docs/netbox/installation/upgrading/
- NetBox release notes: https://netboxlabs.com/docs/netbox/release-notes/
- NetBox releases: https://github.com/netbox-community/netbox/releases

---

## 3. Change Variables

Define the values for the change window before running commands.

```bash
export CURRENT_VERSION='<CURRENT_VERSION>'
export TARGET_VERSION='<TARGET_VERSION>'
export NETBOX_LINK='/opt/netbox'
export NETBOX_ROOT='/opt'
export PYTHON_BIN='<SUPPORTED_PYTHON_BIN>'
```

Example value for `PYTHON_BIN`:

```text
/usr/bin/python3.12
```

Do **not** assume that this example is valid for every NetBox release. Confirm the supported Python range in the official compatibility matrix first.

---

## 4. Pre-Upgrade Discovery

### 4.1 Identify the installation method

```bash
ls -ld /opt/netbox /opt/netbox/.git 2>/dev/null || true
readlink -f /opt/netbox 2>/dev/null || true
```

Interpretation:

- `/opt/netbox` is a symlink and `/opt/netbox/.git` does not exist: release archive installation.
- `/opt/netbox/.git` exists: Git installation.

### 4.2 Record the current application state

```bash
/opt/netbox/venv/bin/python --version
/opt/netbox/venv/bin/pip --version
/opt/netbox/venv/bin/pip check

systemctl status netbox --no-pager
systemctl status netbox-rq --no-pager
systemctl status netbox-housekeeping.timer --no-pager 2>/dev/null || true
```

### 4.3 Check PostgreSQL and Redis versions

Use the database server or an authorized PostgreSQL client:

```bash
psql --version
```

Check the PostgreSQL server version:

```sql
SELECT version();
```

Check Redis:

```bash
redis-server --version
```

If Redis authentication is enabled, do not use an unauthenticated `redis-cli ping` as a health test. Use:

```bash
redis-cli --askpass PING
```

Expected result:

```text
PONG
```

### 4.4 Run the current NetBox system check

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check
```

Do not begin the upgrade if the current installation already reports unresolved system errors.

---

## 5. Review Compatibility and Breaking Changes

Before downloading the target release, verify:

| Component | What to verify |
|---|---|
| NetBox | Direct upgrade path is supported |
| Python | Target release supports the installed Python version |
| PostgreSQL | Server version meets the minimum requirement |
| Redis | Server version meets the minimum requirement |
| Plugins | Every enabled plugin supports the target NetBox version |
| LDAP/SSO | Authentication dependencies support the target Django/NetBox version |
| REST API | Check deprecated or removed endpoints and authentication changes |
| GraphQL | Check filtering/schema breaking changes |
| Webhooks | Check payload changes |
| Custom scripts/reports | Check API and model changes |

List enabled plugins:

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py shell -c \
  'from django.conf import settings; print("\n".join(settings.PLUGINS))'
```

Review custom Python requirements:

```bash
if [ -f /opt/netbox/local_requirements.txt ]; then
  cat /opt/netbox/local_requirements.txt
fi
```

### Important: `local_requirements.txt`

Keep only packages that are intentionally added to NetBox, such as plugins, LDAP libraries, storage backends, or organization-specific dependencies.

Avoid pinning NetBox core dependencies such as Django, `redis`, `rq`, `django-rq`, or `gunicorn` unless the NetBox documentation explicitly requires it. `upgrade.sh` installs the versions required by the target release.

---

## 6. Verify the Python Package Repository

This is especially important when the server uses Nexus, Artifactory, an internal PyPI mirror, or an air-gapped repository.

### 6.1 Inspect pip configuration

```bash
/opt/netbox/venv/bin/python -m pip config debug
/opt/netbox/venv/bin/python -m pip config list
env | grep '^PIP_' || true
```

For upgrades, prefer a system-wide pip configuration such as:

```text
/etc/pip.conf
```

Example:

```ini
[global]
index-url = https://<PYPI_REPOSITORY_HOST>/repository/<PYPI_REPOSITORY>/simple
timeout = 60
```

If the repository uses a private CA, install the CA in the operating system trust store. Avoid `trusted-host` unless it is truly required by your environment.

### 6.2 Test the repository with the target Python

```bash
"$PYTHON_BIN" -m venv /tmp/netbox-pip-test
/tmp/netbox-pip-test/bin/python -m pip config debug
/tmp/netbox-pip-test/bin/python -m pip index versions Django
rm -rf /tmp/netbox-pip-test
```

Test every plugin that must be installed during the upgrade:

```bash
/tmp/example-venv/bin/python -m pip index versions <PLUGIN_PACKAGE>
```

Use an actual temporary venv if needed; the path above is only a placeholder.

---

## 7. Prepare the Supported Python Runtime

Install a Python version supported by the target NetBox release using your approved operating-system or internal repository.

Do **not** replace the operating system's default `/usr/bin/python3` unless your OS vendor explicitly requires it.

Verify:

```bash
"$PYTHON_BIN" --version
"$PYTHON_BIN" -m venv --help >/dev/null
```

---

## 8. Start the Maintenance Window

### 8.1 Stop NetBox write activity

```bash
sudo systemctl stop netbox
sudo systemctl stop netbox-rq
sudo systemctl stop netbox-housekeeping.timer 2>/dev/null || true
```

Verify:

```bash
systemctl is-active netbox || true
systemctl is-active netbox-rq || true
```

### 8.2 Take rollback snapshots

If VMware snapshots are your rollback mechanism, take coordinated snapshots of all components whose state must be restored together:

- NetBox application VM
- PostgreSQL VM
- Remote persistent media/storage, if not included in the application snapshot
- Other stateful components required by your design

Recommended snapshot naming pattern:

```text
pre-netbox-upgrade-YYYYMMDD-HHMM
```

Do not rely on restoring only the application VM after database migrations have been executed.

---

## 9. Install the Target NetBox Release

Set the current installation path:

```bash
CURRENT_DIR="$(readlink -f /opt/netbox)"
echo "$CURRENT_DIR"
```

### Option A - Release Archive Installation

Download the target release:

```bash
cd /tmp
wget -O "netbox-v${TARGET_VERSION}.tar.gz" \
  "https://github.com/netbox-community/netbox/archive/v${TARGET_VERSION}.tar.gz"
```

Extract it:

```bash
sudo tar -xzf "netbox-v${TARGET_VERSION}.tar.gz" -C /opt
```

Define the new directory:

```bash
NEW_DIR="/opt/netbox-${TARGET_VERSION}"
```

Update the symlink:

```bash
sudo ln -sfn "${NEW_DIR}/" /opt/netbox
```

Copy the existing configuration:

```bash
sudo cp "${CURRENT_DIR}/netbox/netbox/configuration.py" \
  /opt/netbox/netbox/netbox/
```

Copy LDAP configuration if present:

```bash
if [ -f "${CURRENT_DIR}/netbox/netbox/ldap_config.py" ]; then
  sudo cp "${CURRENT_DIR}/netbox/netbox/ldap_config.py" \
    /opt/netbox/netbox/netbox/
fi
```

Copy custom requirements if present:

```bash
if [ -f "${CURRENT_DIR}/local_requirements.txt" ]; then
  sudo cp "${CURRENT_DIR}/local_requirements.txt" /opt/netbox/
fi
```

Copy uploaded media:

```bash
if [ -d "${CURRENT_DIR}/netbox/media" ]; then
  sudo cp -pr "${CURRENT_DIR}/netbox/media/." /opt/netbox/netbox/media/
fi
```

Copy custom scripts and reports if they are stored inside the NetBox project tree:

```bash
if [ -d "${CURRENT_DIR}/netbox/scripts" ]; then
  sudo cp -pr "${CURRENT_DIR}/netbox/scripts/." /opt/netbox/netbox/scripts/
fi

if [ -d "${CURRENT_DIR}/netbox/reports" ]; then
  sudo cp -pr "${CURRENT_DIR}/netbox/reports/." /opt/netbox/netbox/reports/
fi
```

Copy Gunicorn configuration if used:

```bash
if [ -f "${CURRENT_DIR}/gunicorn.py" ]; then
  sudo cp "${CURRENT_DIR}/gunicorn.py" /opt/netbox/
fi
```

### Option B - Git Installation

Use this option only if the current installation is Git-based.

```bash
cd /opt/netbox
sudo git fetch --tags
sudo git checkout "v${TARGET_VERSION}"
```

Do not switch installation methods during the same upgrade unless you have explicitly planned and tested that migration.

---

## 10. Update Plugin Requirements

Edit:

```text
/opt/netbox/local_requirements.txt
```

Pin each plugin or custom dependency to a release that supports the target NetBox version.

Generic example:

```text
plugin-package-one==<COMPATIBLE_VERSION>
plugin-package-two==<COMPATIBLE_VERSION>
```

Do not copy old plugin pins blindly to the new release.

---

## 11. Run the NetBox Upgrade

From the new NetBox directory:

```bash
cd /opt/netbox
sudo PYTHON="$PYTHON_BIN" ./upgrade.sh
```

The official upgrade script rebuilds the Python virtual environment, installs NetBox dependencies and `local_requirements.txt`, applies database migrations, collects static files, and performs other upgrade housekeeping tasks.

### Stop Condition

If `upgrade.sh` fails:

1. Do not start NetBox services.
2. Capture the full error output.
3. Fix only a clearly understood dependency/configuration issue, or perform the rollback procedure.
4. Do not create Django migrations manually unless you intentionally modified the NetBox database schema.

---

## 12. Post-Upgrade Validation Before Starting Services

### 12.1 Python dependency validation

```bash
/opt/netbox/venv/bin/python --version
/opt/netbox/venv/bin/pip check
```

Expected:

```text
No broken requirements found.
```

### 12.2 Django/NetBox system check

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check
```

Expected:

```text
System check identified no issues (0 silenced).
```

A deprecation or `FutureWarning` is not necessarily a failure, but it should be reviewed and removed from the configuration when appropriate.

### 12.3 Pending migrations

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py showmigrations --plan \
  | grep '^\[ \]' \
  || echo 'No pending migrations detected.'
```

### 12.4 Confirm installed plugin packages

```bash
/opt/netbox/venv/bin/pip list | sort
```

Optional targeted check:

```bash
/opt/netbox/venv/bin/pip show <PLUGIN_PACKAGE>
```

---

## 13. Start Services

```bash
sudo systemctl restart netbox
sudo systemctl restart netbox-rq
sudo systemctl start netbox-housekeeping.timer 2>/dev/null || true
```

Verify:

```bash
systemctl --no-pager --full status netbox
systemctl --no-pager --full status netbox-rq
systemctl --no-pager --full status netbox-housekeeping.timer 2>/dev/null || true
```

Check recent logs:

```bash
journalctl -u netbox -n 100 --no-pager
journalctl -u netbox-rq -n 100 --no-pager
```

---

## 14. Functional Validation

Test at least the following:

- Web UI loads successfully
- Login works
- Admin users can access expected objects
- NetBox reports the target version
- Search works
- Object create/update/delete works on a safe test object
- Background jobs are processed
- RQ workers are healthy
- Redis connectivity works
- REST API authentication works
- GraphQL queries used by automation still work
- Webhooks still deliver expected payloads
- LDAP/SSO works, if enabled
- Every installed plugin loads and its main pages work
- Custom scripts and reports execute successfully
- Media/images load correctly
- Reverse proxy/TLS health is normal

Useful commands:

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check

/opt/netbox/venv/bin/pip check
```

If Redis requires authentication:

```bash
redis-cli --askpass PING
```

---

## 15. Rollback Procedure - VMware Snapshot Method

Use rollback if the upgrade cannot be corrected safely within the maintenance window.

### 15.1 Stop NetBox

```bash
sudo systemctl stop netbox
sudo systemctl stop netbox-rq
sudo systemctl stop netbox-housekeeping.timer 2>/dev/null || true
```

### 15.2 Revert coordinated snapshots

Restore the snapshots that represent the **same pre-upgrade point in time** for:

1. PostgreSQL VM
2. NetBox application VM
3. Any additional stateful storage included in the change

The exact restore order depends on the virtualization/storage design, but services must remain stopped until all required state has been reverted.

### 15.3 Start the restored environment

After all required snapshots have been reverted:

```bash
sudo systemctl start netbox
sudo systemctl start netbox-rq
sudo systemctl start netbox-housekeeping.timer 2>/dev/null || true
```

Validate:

```bash
sudo -u netbox \
  /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py check
```

Confirm that the UI, database data, workers, plugins, and integrations are back to the pre-upgrade state.

### Critical rollback warning

After `upgrade.sh` has applied database migrations, **changing only `/opt/netbox` back to the previous symlink is not a complete rollback**. The database must also be restored to a compatible pre-upgrade state.

---

## 16. Change Completion Checklist

- [ ] Release notes reviewed
- [ ] Upgrade path confirmed
- [ ] Python compatibility confirmed
- [ ] PostgreSQL compatibility confirmed
- [ ] Redis compatibility confirmed
- [ ] Plugin compatibility confirmed
- [ ] pip/PyPI/Nexus repository tested with target Python
- [ ] NetBox/RQ stopped before snapshots
- [ ] Coordinated rollback snapshots created
- [ ] Target release installed using the original installation method
- [ ] `local_requirements.txt` reviewed and updated
- [ ] `upgrade.sh` completed successfully
- [ ] `pip check` passed
- [ ] `manage.py check` passed
- [ ] No unexpected pending migrations
- [ ] NetBox service healthy
- [ ] RQ workers healthy
- [ ] UI and authentication tested
- [ ] REST API/GraphQL/webhooks tested as applicable
- [ ] Plugins tested
- [ ] Custom scripts/reports tested
- [ ] Monitoring/logs reviewed
- [ ] Snapshot retention/cleanup handled according to change policy

---

## 17. Notes for Production Environments

For business-critical environments, prefer a tested database backup/restore procedure in addition to VM snapshots. Snapshot consistency depends on the virtualization, storage, PostgreSQL, and application state at the moment the snapshot is taken.

For clustered, containerized, Kubernetes, highly available, or horizontally scaled NetBox deployments, adapt this runbook to the architecture rather than applying it verbatim.

---

## 18. References

- NetBox Upgrade Guide: https://netboxlabs.com/docs/netbox/installation/upgrading/
- NetBox Release Notes: https://netboxlabs.com/docs/netbox/release-notes/
- NetBox GitHub Releases: https://github.com/netbox-community/netbox/releases
- NetBox Installation Guide: https://netboxlabs.com/docs/netbox/installation/

---

**Document version:** 3.0  
**Language:** English  
**Designed for:** GitHub / Markdown  
**Environment-specific values:** None
