# Directus v8->v9 Migration

**All secrets in here are used for local test setups only**

Docs:

* https://docs.directus.io/configuration/upgrades-migrations
* https://github.com/directus-community/migration-tool
* https://docs.directus.io/configuration/config-options
* https://avanti.bula21.ch/x/yQNJBQ

## Set up playground v9 env

* Directus: http://localhost:8080/ (`admin@example.org`, `secret-admin`)
* Adminer: http://localhost:8081/ (autologin)
* Keycloak http://keycloak:8082/ (`admin`, `secret-admin`)

### 1. Database import

```bash
docker-compose up database adminer # give MySQL time to seed
```

Export `directus` database from production. Options:

* Output: gzip
* Format: SQL
* USE, CREATE, INSERT
* All tables

Import locally in adminer. Then, rename the old directus tables:

```sql
-- -- meta-command:
-- SELECT CONCAT('RENAME TABLE ', table_name, ' TO ', table_name ,'__old;')
-- FROM information_schema.tables
-- WHERE TABLE_SCHEMA = 'directus'
-- AND table_name LIKE 'directus_%'

RENAME TABLE directus_activity TO directus_activity__old;
RENAME TABLE directus_collection_presets TO directus_collection_presets__old;
RENAME TABLE directus_collections TO directus_collections__old;
RENAME TABLE directus_fields TO directus_fields__old;
RENAME TABLE directus_files TO directus_files__old;
RENAME TABLE directus_folders TO directus_folders__old;
RENAME TABLE directus_migrations TO directus_migrations__old;
RENAME TABLE directus_permissions TO directus_permissions__old;
RENAME TABLE directus_relations TO directus_relations__old;
RENAME TABLE directus_revisions TO directus_revisions__old;
RENAME TABLE directus_roles TO directus_roles__old;
RENAME TABLE directus_settings TO directus_settings__old;
RENAME TABLE directus_user_sessions TO directus_user_sessions__old;
RENAME TABLE directus_users TO directus_users__old;
RENAME TABLE directus_webhooks TO directus_webhooks__old;
```

### 2. Keycloak

In your `/etc/hosts`, add `127.0.0.1 keycloak`.

```bash
# Ctrl-C
docker-compose up keycloak
```

Configure (Or, just import `test-realm-export.json`):

* Go to http://keycloak:8082/auth/admin/ (`admin`, `secret-admin`):
* Add Realm
  * "log-directus"
* Users -> Add
  * "test", "test@example.org", email verified
  * Credentials -> Set Password "secret-test", temporary = Off
* Create client
  * "log-directus"
  * Access Type: "confidential"
  * Redirect URI: `*`
  * Save
  * Go to "Credentials" and copy to `docker-compose.yml` `AUTH_KEYCLOAK_CLIENT_SECRET`
* Restart docker-compose

### 3. Directus and apps

```bash
# Ctrl-C
docker-compose up
```

You can now also log into directus via Keycloak. No user pre-creation required.
