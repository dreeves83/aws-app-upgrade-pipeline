# aws-app-upgrade-pipeline

Jenkins pipeline that automates web application upgrades on AWS-hosted Windows servers, with pre-upgrade backups and config preservation.

## Background

The application runs on IIS as a **base tenant** plus one folder and app pool per **customer tenant**, and some environments host multiple customers (multi-tenant). The original pipeline could only update the base tenant. Upgrading a customer tenant was a manual process: back up the folder and database, save the customer's config files, rebuild the folder from the updated base tenant, and restore the configs.

This pipeline extends the original into a self-service upgrade for both, with backups and config preservation built in, while keeping the original base-tenant-only behavior exactly as it was.

> The code in this repo has been sanitized and generalized from production work. Names, paths, and internal references have been replaced, so it is meant to show the approach rather than run as a drop-in tool.

## How it works

The pipeline targets the stack `<environment>-<instance>`, finds its app and processing servers through CloudFormation, and runs every on-server step through **AWS SSM** (PowerShell on the Windows hosts).

### Modes

| `CLIENT_NAME` | Behavior |
|---|---|
| Blank | **Base tenant only.** Runs exactly like the original pipeline. All customer stages are skipped. |
| Set | **Customer upgrade.** Validates, backs up, updates the base tenant, then rebuilds the customer's tenant from it. |

### Stages

**1. Client Name Validation**
- Checks `CLIENT_NAME` against the **live IIS app pool list** on the app server, not a hardcoded list, which matters in multi-tenant environments.
- A mistyped name fails the run immediately with a clear error listing the valid tenants.
- Stops the customer's app pool once validated.

**2. Backing up Client Data**
- **Config capture:** reads the customer's four config files (`CustomerAppSettings.config` and `CustomerConnectionStrings.config` under both `Services\` and `Admin\`) into pipeline memory *before anything on disk changes*. None of this content is printed to the console.
- **Database lookup:** parses the DB host and name from the `database` entry in the connection strings file. The database name can't be derived from the customer name, so parsing the config is the only reliable source.
- **Folder backup:** zips the customer's folder on the app server.
- **Database backup:** runs `pg_dump` on the processing server, pulling the admin password from AWS Secrets Manager at runtime.
- Both backups are uploaded to S3, then verified with `head-object` before the local copies are deleted (see [Backups](#backups)).

**3. Install latest App update**
- Stops the base tenant app pool, clears its `bin` folder, and runs the existing install script to update the base tenant.

**4. Rebuild Client Folder and Configuration**
- Wipes the customer's folder and copies in the freshly updated base tenant.
- Writes the four captured config files back **byte-for-byte**.
- Restarts the customer and base tenant app pools.

### Options

- **`SKIP_BACKUP`** skips the folder zip, database backup, and S3 upload to save about 10 minutes during testing. Config capture still runs, because the rebuild depends on it.
- **`SKIP_BASE_TENANT_UPDATE`** skips the base tenant install entirely. It's built for multi-tenant environments where the base tenant is already on the target version and only customer tenants need to catch up.

## Backups

- Files are named `<customer>_<DisplayVersion>_<InternalVersion>_<date>`, with the version taken from the tenant's `version.json` at backup time, so every backup records exactly what version it holds.
- The S3 bucket is chosen by environment and region, with files stored under `data-backup/<customer>/<DisplayVersion>_<InternalVersion>/`.
- The exact local file is uploaded rather than a regenerated one, so the size check compares like with like.
- If an upload can't be verified, the local copy is **kept** and the run logs a warning instead of deleting it.
- S3 destination paths are printed in the console for every run.

## Design decisions and lessons learned

- **Prototype first.** Each piece of on-server logic was built and tested as a small standalone PowerShell script against a QA stack before being wired into the pipeline as SSM commands.
- **Backward compatible.** A blank `CLIENT_NAME` keeps the original behavior, so existing base-tenant workflows didn't change.
- **Capture before touching.** Configs are held in memory before any destructive step, so the rebuild never depends on files that are about to be deleted.
- **Base64 transport.** Config contents are base64-encoded to pass cleanly through Groovy → JSON → SSM → PowerShell without quoting or XML-escaping problems. The encoder is a small custom Groovy function because of Jenkins script-security sandbox restrictions.
- **`@NonCPS` for regex.** Regex parsing lives in `@NonCPS` helper methods. Run inline, Jenkins tries to serialize the regex `Matcher` at a pipeline checkpoint and crashes the build, which happened twice during development before this fix.
- **No secrets in logs.** Config contents and the database password never reach the console. The password exists only as a short-lived environment variable during `pg_dump`.
- **PostgreSQL client detection.** The processing server is checked for client tools versions 14–17 rather than one hardcoded path.

## Testing

All scenarios were verified against a QA stack:

- Base tenant only (original behavior)
- Full end-to-end customer upgrade, including the base tenant
- Customer-only upgrade with `SKIP_BASE_TENANT_UPDATE`
- Mistyped `CLIENT_NAME` fails with clear error messaging
- S3 upload validation, and cleanup of local backups after validation
- `SKIP_BACKUP` skips the backup work with no configuration lost

## Requirements

- **App server:** IIS, the SSM agent, the AWS CLI, and the application's install script (`E:\scripts\Install-Application.ps1`).
- **Processing server:** the SSM agent, the AWS CLI, and PostgreSQL client tools 14, 15, 16, or 17.
- An RDS instance using a Secrets Manager–managed master password.

## Notes

- `internal-jenkins-lib` stands in for an internal Jenkins shared library. Its `env_parameters` step resolves the deployment IAM role and AWS account for the selected environment and region, used with `withAWS`.
- `APP_BRANCH` selects the application build to install. `REPO_BRANCH` is used by the Jenkins job's SCM configuration to run the pipeline from `main` or a feature branch.

## Tech

Jenkins (declarative pipeline, Groovy) · PowerShell · AWS SSM, S3, Secrets Manager, RDS (PostgreSQL), CloudFormation · IIS
