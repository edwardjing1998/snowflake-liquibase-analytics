# Snowflake Analytics Deployment with Maven and Liquibase

This repository deploys version-controlled policy and billing analytical objects
to Snowflake. It is a Maven/Liquibase deployment project, not a Spring Boot
microservice and not a Databricks job.

## Security first

Never place a Snowflake password in this repository. If a password was exposed
in chat, logs, source code, or terminal history, rotate it before continuing.

## Default target

Configure these GitHub values for the learning environment:

| Type | Name | Suggested value |
|---|---|---|
| Variable | `SNOWFLAKE_ACCOUNT` | `usbxmys-cm03603` |
| Variable | `SNOWFLAKE_DATABASE` | `EDU_AI_APP` |
| Variable | `SNOWFLAKE_WAREHOUSE` | `GLOBAL_FINANCE_WAREHOUSE` |
| Variable | `SNOWFLAKE_ROLE` | A role allowed to create objects in the target database |
| Variable | `SNOWFLAKE_CHANGELOG_SCHEMA` | `PUBLIC` |
| Secret | `SNOWFLAKE_USER` | Your Snowflake username |
| Secret | `SNOWFLAKE_PASSWORD` | Your newly rotated password |

Use a dedicated deployment role in a real project. Do not use `ACCOUNTADMIN` in
CI/CD.

## Objects created

Schemas:

```text
EDU_AI_APP.STAGING
EDU_AI_APP.CORE
EDU_AI_APP.MART
EDU_AI_APP.AUDIT
```

Tables and views:

```text
STAGING.STG_POLICY
STAGING.STG_BILL_LINE
CORE.DIM_POLICY
CORE.BRIDGE_POLICY_PRODUCT
CORE.DIM_BILLING_CYCLE
CORE.FACT_BILL
CORE.FACT_BILL_LINE
CORE.FACT_PAYMENT
CORE.FACT_ADJUSTMENT
AUDIT.DATA_LOAD_AUDIT
MART.V_ROAMING_ADJUSTMENT_SUMMARY
```

Liquibase also creates `DATABASECHANGELOG` and `DATABASECHANGELOGLOCK` in the
configured changelog schema.

## Rotate an exposed password

Sign in with a sufficiently privileged identity and use your organization's
approved password-reset procedure. A Snowflake administrator can execute an
appropriate `ALTER USER ... SET PASSWORD` statement. Do not paste the new
password into SQL committed to Git.

## Local prerequisites

- JDK 21
- Maven 3.9+
- Network access to the Snowflake account
- A Snowflake role with the required database/schema/object privileges

## Local configuration

Export values in your current terminal. Do not save the password in Git:

```bash
export SNOWFLAKE_ACCOUNT="usbxmys-cm03603"
export SNOWFLAKE_DATABASE="EDU_AI_APP"
export SNOWFLAKE_WAREHOUSE="GLOBAL_FINANCE_WAREHOUSE"
export SNOWFLAKE_ROLE="YOUR_DEPLOYMENT_ROLE"
export SNOWFLAKE_CHANGELOG_SCHEMA="PUBLIC"
export SNOWFLAKE_USER="YOUR_USERNAME"
export SNOWFLAKE_PASSWORD="YOUR_NEW_PASSWORD"
```

Validate the changelogs:

```bash
mvn --batch-mode liquibase:validate
```

Review pending changes:

```bash
mvn --batch-mode liquibase:status -Dliquibase.verbose=true
```

Generate SQL without changing Snowflake:

```bash
mvn --batch-mode liquibase:updateSQL \
  -Dliquibase.outputFile=target/liquibase-update.sql
```

Apply pending changesets:

```bash
mvn --batch-mode liquibase:update
```

## GitHub setup

1. Create a new GitHub repository and push this project.
2. Open `Settings -> Environments`.
3. Create an environment named `snowflake-staging`.
4. Add the five environment variables listed above.
5. Add `SNOWFLAKE_USER` and the rotated `SNOWFLAKE_PASSWORD` as environment
   secrets.
6. Optionally configure required reviewers on the environment.
7. Push to `main` or select `Actions -> Deploy Snowflake Analytical Objects -> Run workflow`.

## Verify in Snowflake

```sql
SELECT ID, AUTHOR, FILENAME, DATEEXECUTED, ORDEREXECUTED, EXECTYPE
FROM EDU_AI_APP.PUBLIC.DATABASECHANGELOG
ORDER BY ORDEREXECUTED;
```

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE
FROM EDU_AI_APP.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA IN ('STAGING', 'CORE', 'MART', 'AUDIT')
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

```sql
SELECT *
FROM EDU_AI_APP.MART.V_ROAMING_ADJUSTMENT_SUMMARY;
```

## Change-management rules

- Never modify an already deployed changeset.
- Add a new uniquely identified changeset for each change.
- Review the generated SQL artifact before approving a protected deployment.
- Treat rollback that drops analytical tables as destructive; do not run it in
  production without an explicit recovery plan.
- Snowflake standard-table primary and unique constraints are usually
  informational. Validate uniqueness and referential integrity in data jobs and
  quality tests.
- Keep customer PII out of these tables unless it is classified, approved,
  tokenized, and protected by appropriate policies.
