# Create a New PostgreSQL Database

A general-purpose guide for creating a PostgreSQL database, creating a dedicated user, configuring PgBouncer, testing a TLS connection, and confirming automatic backups.

This guide uses placeholders only. Replace every value enclosed in angle brackets.

---

# 1. Placeholder Reference


| Placeholder            | Description                    | Example                        |
| ---------------------- | ------------------------------ | ------------------------------ |
| `<PRIVATE_DB_HOST>`    | Private PostgreSQL host        | `127.0.0.1`                    |
| `<PRIVATE_DB_PORT>`    | Private PostgreSQL port        | `5432`                         |
| `<PUBLIC_DB_HOST>`     | Public database hostname       | `db.example.com`               |
| `<PUBLIC_DB_PORT>`     | Public PgBouncer port          | `6432`                         |
| `<ADMIN_USER>`         | PostgreSQL administrative user | `postgres`                     |
| `<DATABASE_NAME>`      | New database name              | `project_db`                   |
| `<DATABASE_USER>`      | New database user              | `project_user`                 |
| `<DATABASE_PASSWORD>`  | Strong database password       | Generated securely             |
| `<BACKUP_ROOT>`        | Backup storage directory       | `/var/backups/postgresql`      |
| `<PGBOUNCER_CONFIG>`   | PgBouncer configuration file   | `/etc/pgbouncer/pgbouncer.ini` |
| `<PGBOUNCER_USERLIST>` | PgBouncer authentication file  | `/etc/pgbouncer/userlist.txt`  |

Recommended database and user names:

```text
project_db
project_user
```

Use lowercase letters, numbers, and underscores.

---

# 2. Requirements

You need:

- A running PostgreSQL-compatible server
- Administrator access to PostgreSQL
- `psql`
- `pg_dump`
- `pg_restore`
- OpenSSL
- Optional PgBouncer installation
- Optional TLS certificate
- Root or sudo access to the server

Check the tools:

```bash
psql --version
pg_dump --version
pg_restore --version
openssl version
```

---

# 3. Confirm PostgreSQL Is Running

Check the private database port:

```bash
ss -lntp | grep ':<PRIVATE_DB_PORT>'
```

Test the administrator connection:

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -c "SELECT version(), current_database(), current_user;"
```

Do not continue until this command succeeds.

---

# 4. Set the Database Variables

Set the names:

```bash
NEW_DB="<DATABASE_NAME>"
NEW_USER="<DATABASE_USER>"
```

Generate a strong URL-safe password:

```bash
NEW_PASSWORD="$(openssl rand -hex 24)"
```

Display the values once:

```bash
echo "Database: $NEW_DB"
echo "Username: $NEW_USER"
echo "Password: $NEW_PASSWORD"
```

Store the password securely.

Validate the names:

```bash
if [[ ! "$NEW_DB" =~ ^[a-z][a-z0-9_]{2,62}$ ]]; then
  echo "Invalid database name: $NEW_DB"
  exit 1
fi

if [[ ! "$NEW_USER" =~ ^[a-z][a-z0-9_]{2,62}$ ]]; then
  echo "Invalid database user: $NEW_USER"
  exit 1
fi
```

The variables exist only in the current shell.

---

# 5. Check Whether the User Exists

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v role_name="$NEW_USER" \
  -Atc "SELECT 1 FROM pg_roles WHERE rolname = :'role_name';"
```

An empty response means the role does not exist.

A response of:

```text
1
```

means it already exists.

---

# 6. Create the Database User

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v role_name="$NEW_USER" \
  -v role_password="$NEW_PASSWORD" <<'SQL'
SELECT format(
  'CREATE ROLE %I LOGIN PASSWORD %L',
  :'role_name',
  :'role_password'
)
WHERE NOT EXISTS (
  SELECT 1
  FROM pg_roles
  WHERE rolname = :'role_name'
)
\gexec

SELECT format(
  'ALTER ROLE %I WITH LOGIN PASSWORD %L',
  :'role_name',
  :'role_password'
)
\gexec
SQL
```

Verify the user:

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v role_name="$NEW_USER" \
  -c "
    SELECT
      rolname,
      rolcanlogin,
      rolcreatedb,
      rolcreaterole,
      rolsuper
    FROM pg_roles
    WHERE rolname = :'role_name';
  "
```

Recommended settings:

```text
rolcanlogin   = true
rolcreatedb   = false
rolcreaterole = false
rolsuper      = false
```

---

# 7. Create the Database

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v database_name="$NEW_DB" \
  -v role_name="$NEW_USER" <<'SQL'
SELECT format(
  'CREATE DATABASE %I OWNER %I ENCODING %L TEMPLATE template0',
  :'database_name',
  :'role_name',
  'UTF8'
)
WHERE NOT EXISTS (
  SELECT 1
  FROM pg_database
  WHERE datname = :'database_name'
)
\gexec
SQL
```

Verify:

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v database_name="$NEW_DB" \
  -c "
    SELECT
      datname AS database_name,
      pg_get_userbyid(datdba) AS owner,
      pg_encoding_to_char(encoding) AS encoding,
      datallowconn AS connections_allowed
    FROM pg_database
    WHERE datname = :'database_name';
  "
```

---

# 8. Apply Basic Access Controls

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d "$NEW_DB" \
  -v database_name="$NEW_DB" \
  -v role_name="$NEW_USER" <<'SQL'
SELECT format(
  'REVOKE ALL ON DATABASE %I FROM PUBLIC',
  :'database_name'
)
\gexec

SELECT format(
  'GRANT CONNECT, TEMPORARY ON DATABASE %I TO %I',
  :'database_name',
  :'role_name'
)
\gexec

REVOKE CREATE ON SCHEMA public FROM PUBLIC;

SELECT format(
  'GRANT USAGE, CREATE ON SCHEMA public TO %I',
  :'role_name'
)
\gexec
SQL
```

---

# 9. Test the New Database Directly

```bash
PGPASSWORD="$NEW_PASSWORD" psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U "$NEW_USER" \
  -d "$NEW_DB" \
  -c "SELECT current_database(), current_user, now();"
```

Create a temporary test table:

```bash
PGPASSWORD="$NEW_PASSWORD" psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U "$NEW_USER" \
  -d "$NEW_DB" <<'SQL'
CREATE TABLE connection_test (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  message TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO connection_test (message)
VALUES ('Database connection successful');

SELECT * FROM connection_test;
SQL
```

Remove it:

```bash
PGPASSWORD="$NEW_PASSWORD" psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U "$NEW_USER" \
  -d "$NEW_DB" \
  -c "DROP TABLE connection_test;"
```

---

# 10. Optional PgBouncer Database Mapping

Skip this section when PgBouncer is not used.

Back up the configuration:

```bash
cp <PGBOUNCER_CONFIG> \
  "<PGBOUNCER_CONFIG>.backup.$(date +%F_%H-%M-%S)"
```

Inspect the database section:

```bash
sed -n '/^\[databases\]/,/^\[/p' <PGBOUNCER_CONFIG>
```

Create the mapping:

```bash
PGBOUNCER_DB_LINE="${NEW_DB} = host=<PRIVATE_DB_HOST> port=<PRIVATE_DB_PORT> dbname=${NEW_DB}"
```

Add it only when it does not already exist:

```bash
if ! grep -qE "^${NEW_DB}[[:space:]]*=" <PGBOUNCER_CONFIG>; then
  sed -i \
    "/^\[databases\]$/a ${PGBOUNCER_DB_LINE}" \
    <PGBOUNCER_CONFIG>
fi
```

Verify:

```bash
sed -n '/^\[databases\]/,/^\[/p' <PGBOUNCER_CONFIG>
```

---

# 11. Add the User to PgBouncer

This example assumes:

```ini
auth_type = scram-sha-256
auth_file = <PGBOUNCER_USERLIST>
```

Retrieve the SCRAM verifier:

```bash
SCRAM_SECRET="$(
  psql \
    -h <PRIVATE_DB_HOST> \
    -p <PRIVATE_DB_PORT> \
    -U <ADMIN_USER> \
    -d postgres \
    -v role_name="$NEW_USER" \
    -Atc "
      SELECT rolpassword
      FROM pg_authid
      WHERE rolname = :'role_name';
    "
)"
```

Validate:

```bash
case "$SCRAM_SECRET" in
  SCRAM-SHA-256\$*)
    echo "SCRAM verifier retrieved successfully."
    ;;
  *)
    echo "ERROR: Valid SCRAM verifier not found."
    exit 1
    ;;
esac
```

Back up the user list:

```bash
cp <PGBOUNCER_USERLIST> \
  "<PGBOUNCER_USERLIST>.backup.$(date +%F_%H-%M-%S)"
```

Detect the PgBouncer service account:

```bash
PB_USER="$(systemctl show pgbouncer.service -p User --value)"
[ -n "$PB_USER" ] || PB_USER="postgres"

PB_GROUP="$(id -gn "$PB_USER")"
```

Create an updated file:

```bash
TEMP_USERLIST="$(mktemp)"
```

Remove an old entry for the same user:

```bash
awk -v user="\"${NEW_USER}\"" \
  '$1 != user { print }' \
  <PGBOUNCER_USERLIST> \
  > "$TEMP_USERLIST"
```

Append the current verifier:

```bash
printf '"%s" "%s"\n' \
  "$NEW_USER" \
  "$SCRAM_SECRET" \
  >> "$TEMP_USERLIST"
```

Install securely:

```bash
install \
  -o root \
  -g "$PB_GROUP" \
  -m 640 \
  "$TEMP_USERLIST" \
  <PGBOUNCER_USERLIST>
```

Delete the temporary file:

```bash
rm -f "$TEMP_USERLIST"
```

List usernames without exposing secrets:

```bash
cut -d'"' -f2 <PGBOUNCER_USERLIST>
```

---

# 12. Reload PgBouncer

```bash
systemctl reload pgbouncer.service || \
  systemctl restart pgbouncer.service
```

Check status:

```bash
systemctl status pgbouncer.service --no-pager -l
```

Check logs:

```bash
journalctl \
  -u pgbouncer.service \
  -n 100 \
  --no-pager
```

Check the public port:

```bash
ss -lntp | grep ':<PUBLIC_DB_PORT>'
```

---

# 13. Test the Public TLS Connection

```bash
PGPASSWORD="$NEW_PASSWORD" psql \
  "host=<PUBLIC_DB_HOST> \
  port=<PUBLIC_DB_PORT> \
  dbname=$NEW_DB \
  user=$NEW_USER \
  sslmode=verify-full \
  sslrootcert=system"
```

When `sslrootcert=system` is unsupported:

```bash
PGPASSWORD="$NEW_PASSWORD" psql \
  "host=<PUBLIC_DB_HOST> \
  port=<PUBLIC_DB_PORT> \
  dbname=$NEW_DB \
  user=$NEW_USER \
  sslmode=verify-full \
  sslrootcert=/etc/ssl/certs/ca-certificates.crt"
```

After connecting:

```sql
SELECT current_database(), current_user, now();
```

Check encryption:

```sql
SELECT
  ssl,
  version,
  cipher
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();
```

Expected:

```text
ssl = true
```

Exit:

```sql
\q
```

---

# 14. Generate the Public Database URL

```bash
echo "postgresql://${NEW_USER}:${NEW_PASSWORD}@<PUBLIC_DB_HOST>:<PUBLIC_DB_PORT>/${NEW_DB}?sslmode=verify-full"
```

Generic format:

```text
postgresql://<DATABASE_USER>:<DATABASE_PASSWORD>@<PUBLIC_DB_HOST>:<PUBLIC_DB_PORT>/<DATABASE_NAME>?sslmode=verify-full
```

Do not include `https://` inside the host.

---

# 15. Save Credentials Securely

```bash
CREDENTIAL_FILE="/root/${NEW_DB}-credentials.env"
```

```bash
cat > "$CREDENTIAL_FILE" <<EOF
DB_HOST=<PUBLIC_DB_HOST>
DB_PORT=<PUBLIC_DB_PORT>
DB_NAME=${NEW_DB}
DB_USER=${NEW_USER}
DB_PASSWORD=${NEW_PASSWORD}
DATABASE_URL=postgresql://${NEW_USER}:${NEW_PASSWORD}@<PUBLIC_DB_HOST>:<PUBLIC_DB_PORT>/${NEW_DB}?sslmode=verify-full
EOF
```

Protect it:

```bash
chmod 600 "$CREDENTIAL_FILE"
chown root:root "$CREDENTIAL_FILE"
```

Clear sensitive variables after saving:

```bash
unset NEW_PASSWORD SCRAM_SECRET
```

---

# 16. Automatic Backup Integration

This section assumes a backup script automatically discovers all connectable non-template databases.

Run a backup:

```bash
systemctl start postgresql-app-backup.service
```

Check status:

```bash
systemctl status postgresql-app-backup.service --no-pager -l
```

A successful oneshot service may show:

```text
inactive (dead)
```

after completing. Confirm:

```text
status=0/SUCCESS
```

Check the database-specific folder:

```bash
find "<BACKUP_ROOT>/$NEW_DB" \
  -maxdepth 1 \
  -type f \
  -name '*.dump' \
  -printf '%f\n' \
  | sort
```

Recommended structure:

```text
<BACKUP_ROOT>/
├── database_one/
│   ├── database_one_TIMESTAMP.dump
│   └── database_one_TIMESTAMP.dump
└── database_two/
    ├── database_two_TIMESTAMP.dump
    └── database_two_TIMESTAMP.dump
```

Retention behavior:

```text
Run 1: Keep backup 1
Run 2: Keep backup 1 and backup 2
Run 3: Delete backup 1; keep backup 2 and backup 3
Run 4: Delete backup 2; keep backup 3 and backup 4
```

Check the timer:

```bash
systemctl list-timers --all | grep postgresql-app-backup
```

---

# 17. List All Databases

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -c "
    SELECT
      datname AS database,
      pg_get_userbyid(datdba) AS owner,
      pg_size_pretty(pg_database_size(datname)) AS size
    FROM pg_database
    WHERE datistemplate = false
    ORDER BY datname;
  "
```

---

# 18. Change a User Password

```bash
TARGET_USER="<DATABASE_USER>"
NEW_PASSWORD="$(openssl rand -hex 24)"
```

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v role_name="$TARGET_USER" \
  -v role_password="$NEW_PASSWORD" <<'SQL'
SELECT format(
  'ALTER ROLE %I PASSWORD %L',
  :'role_name',
  :'role_password'
)
\gexec
SQL
```

After changing the password:

1. Retrieve the new SCRAM verifier.
2. Replace the user entry in PgBouncer.
3. Reload PgBouncer.
4. Update application secrets.

---

# 19. Disable a User

```bash
TARGET_USER="<DATABASE_USER>"
```

Disable:

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v role_name="$TARGET_USER" \
  -c "SELECT format('ALTER ROLE %I NOLOGIN', :'role_name') \gexec"
```

Enable:

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v role_name="$TARGET_USER" \
  -c "SELECT format('ALTER ROLE %I LOGIN', :'role_name') \gexec"
```

---

# 20. Delete a Database Safely

Create and validate a backup first.

```bash
DELETE_DB="<DATABASE_NAME>"
DELETE_USER="<DATABASE_USER>"
```

Terminate active sessions:

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v database_name="$DELETE_DB" \
  -c "
    SELECT pg_terminate_backend(pid)
    FROM pg_stat_activity
    WHERE datname = :'database_name'
      AND pid <> pg_backend_pid();
  "
```

Drop the database:

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v database_name="$DELETE_DB" \
  -c "SELECT format('DROP DATABASE %I', :'database_name') \gexec"
```

Drop the user:

```bash
psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d postgres \
  -v role_name="$DELETE_USER" \
  -c "SELECT format('DROP ROLE %I', :'role_name') \gexec"
```

Remove the PgBouncer database mapping:

```bash
sed -i \
  "/^${DELETE_DB}[[:space:]]*=/d" \
  <PGBOUNCER_CONFIG>
```

Remove the PgBouncer user entry and reload PgBouncer.

---

# 21. Restore a Backup

Find backups:

```bash
find <BACKUP_ROOT> \
  -type f \
  -name '*.dump' \
  -printf '%p\n' \
  | sort
```

Set values:

```bash
RESTORE_DB="<RESTORED_DATABASE_NAME>"
RESTORE_USER="<DATABASE_USER>"
BACKUP_FILE="<BACKUP_ROOT>/<DATABASE_NAME>/<BACKUP_FILE>.dump"
```

Create the empty database:

```bash
createdb \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -O "$RESTORE_USER" \
  "$RESTORE_DB"
```

Restore:

```bash
pg_restore \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <ADMIN_USER> \
  -d "$RESTORE_DB" \
  --no-owner \
  --role="$RESTORE_USER" \
  "$BACKUP_FILE"
```

Test:

```bash
PGPASSWORD="<DATABASE_PASSWORD>" psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U "$RESTORE_USER" \
  -d "$RESTORE_DB" \
  -c "SELECT current_database(), current_user;"
```

---

# 22. GUI Connection Fields

```text
Server name: Any descriptive name
Host: <PUBLIC_DB_HOST>
Port: <PUBLIC_DB_PORT>
Database: <DATABASE_NAME>
User: <DATABASE_USER>
Password: <DATABASE_PASSWORD>
SSL mode: verify-full
```

Do not include `https://` in the host field.

---

# 23. Troubleshooting

## Private connection refused

```bash
ss -lntp | grep ':<PRIVATE_DB_PORT>'
```

Confirm PostgreSQL is running.

## PgBouncer failed

```bash
systemctl status pgbouncer --no-pager -l
journalctl -u pgbouncer -n 100 --no-pager
```

## Public port is not listening

```bash
ss -lntp | grep ':<PUBLIC_DB_PORT>'
```

## PgBouncer reports no such database

```bash
sed -n '/^\[databases\]/,/^\[/p' <PGBOUNCER_CONFIG>
```

## PgBouncer reports no such user

```bash
cut -d'"' -f2 <PGBOUNCER_USERLIST>
```

## Password authentication failed

Test directly against the private PostgreSQL endpoint:

```bash
PGPASSWORD="<DATABASE_PASSWORD>" psql \
  -h <PRIVATE_DB_HOST> \
  -p <PRIVATE_DB_PORT> \
  -U <DATABASE_USER> \
  -d <DATABASE_NAME> \
  -c "SELECT current_user;"
```

When the private connection works but PgBouncer fails, refresh the SCRAM verifier in the PgBouncer user list.

## Root certificate missing

Use one of:

```text
sslrootcert=system
```

```text
sslrootcert=/etc/ssl/certs/ca-certificates.crt
```

---

# 24. Final Checklist

```text
[ ] PostgreSQL is running
[ ] New user exists
[ ] New user is not a superuser
[ ] New database exists
[ ] New user owns the database
[ ] Private connection succeeds
[ ] PgBouncer database mapping exists
[ ] PgBouncer user list contains the user
[ ] PgBouncer was reloaded
[ ] Public TLS connection succeeds
[ ] pg_stat_ssl reports ssl = true
[ ] Credentials are stored securely
[ ] Automatic backup includes the new database
[ ] Only the latest two backups are retained
```

---

# 25. Official References

- PostgreSQL `CREATE ROLE`
  https://www.postgresql.org/docs/current/sql-createrole.html
- PostgreSQL `CREATE DATABASE`
  https://www.postgresql.org/docs/current/sql-createdatabase.html
- PostgreSQL `psql`
  https://www.postgresql.org/docs/current/app-psql.html
- PostgreSQL `pg_dump`
  https://www.postgresql.org/docs/current/app-pgdump.html
- PostgreSQL `pg_restore`
  https://www.postgresql.org/docs/current/app-pgrestore.html
- PgBouncer configuration
  https://www.pgbouncer.org/config
- PgBouncer usage
  https://www.pgbouncer.org/usage
