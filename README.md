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
-- -- meta-query generating the below:
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

You can now also log into Directus via Keycloak. No user pre-creation required.

## Migration

### Users

Run in adminer:

```sql
INSERT INTO directus_users (
  id,
  first_name,
  last_name,
  email,
  provider,
  external_identifier
) SELECT
  UUID(),
  first_name,
  last_name,
  email,
  'keycloak',
  external_id
FROM directus_users__old;
```

### Ownership

> Example (`telekom.owner`):
> 
> ```sql
> -- for verification only, not needed in prod
> ALTER TABLE telekom ADD owner__old INT(10) NULL AFTER owner;
> UPDATE telekom SET owner__old = owner;
> ALTER TABLE telekom MODIFY owner CHAR(36) NULL;
> 
> UPDATE telekom
> LEFT JOIN directus_users__old ON telekom.owner = directus_users__old.id
> LEFT JOIN directus_users ON directus_users.email = directus_users__old.email
> SET telekom.owner = directus_users.id
> ```
> 
> Old -> new ID mapping query:
> 
> ```sql
> SELECT
>   directus_users.email,
>   directus_users__old.id AS id_old,
>   directus_users.id AS id_new
> FROM directus_users__old
> LEFT JOIN directus_users ON directus_users.email = directus_users__old.email
> ORDER BY id_old
> ```

```sql
-- -- meta-query generating the below:
-- SELECT
--   -- TABLE_NAME,
--   -- COLUMN_NAME,
--   CONCAT('ALTER TABLE ', TABLE_NAME,' MODIFY ', COLUMN_NAME,' CHAR(36) NULL;'),
--   CONCAT('UPDATE ', TABLE_NAME,' LEFT JOIN directus_users__old ON ', TABLE_NAME,'.', COLUMN_NAME,' = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET ', TABLE_NAME,'.', COLUMN_NAME,' = directus_users.id;')
-- FROM
--   information_schema.columns
-- WHERE
--   table_schema = 'directus'
--   AND NOT TABLE_NAME LIKE 'directus_%'
--   AND DATA_TYPE = 'int'
--   AND (
--     COLUMN_NAME LIKE '%owner%'
--     OR COLUMN_NAME LIKE '%modified_by%'
--     OR COLUMN_NAME LIKE '%kontaktperson%'
--     OR COLUMN_NAME LIKE '%auftraggeber%'
--     OR COLUMN_NAME LIKE '%verantwortliche_person_betrieb%'
--   )
-- ORDER BY
--   table_name,
--   ordinal_position;


ALTER TABLE abfallentsorgung MODIFY owner CHAR(36) NULL;	UPDATE abfallentsorgung LEFT JOIN directus_users__old ON abfallentsorgung.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET abfallentsorgung.owner = directus_users.id;
ALTER TABLE abnahme MODIFY owner CHAR(36) NULL;	UPDATE abnahme LEFT JOIN directus_users__old ON abnahme.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET abnahme.owner = directus_users.id;
ALTER TABLE abnahme MODIFY modified_by CHAR(36) NULL;	UPDATE abnahme LEFT JOIN directus_users__old ON abnahme.modified_by = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET abnahme.modified_by = directus_users.id;
ALTER TABLE abwasser MODIFY owner CHAR(36) NULL;	UPDATE abwasser LEFT JOIN directus_users__old ON abwasser.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET abwasser.owner = directus_users.id;
ALTER TABLE anlage MODIFY owner CHAR(36) NULL;	UPDATE anlage LEFT JOIN directus_users__old ON anlage.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET anlage.owner = directus_users.id;
ALTER TABLE anlage MODIFY modified_by CHAR(36) NULL;	UPDATE anlage LEFT JOIN directus_users__old ON anlage.modified_by = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET anlage.modified_by = directus_users.id;
ALTER TABLE anlage MODIFY kontaktperson CHAR(36) NULL;	UPDATE anlage LEFT JOIN directus_users__old ON anlage.kontaktperson = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET anlage.kontaktperson = directus_users.id;
ALTER TABLE objekt MODIFY owner CHAR(36) NULL;	UPDATE objekt LEFT JOIN directus_users__old ON objekt.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET objekt.owner = directus_users.id;
ALTER TABLE objekt MODIFY modified_by CHAR(36) NULL;	UPDATE objekt LEFT JOIN directus_users__old ON objekt.modified_by = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET objekt.modified_by = directus_users.id;
ALTER TABLE objekt MODIFY kontaktperson_auftraggeber CHAR(36) NULL;	UPDATE objekt LEFT JOIN directus_users__old ON objekt.kontaktperson_auftraggeber = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET objekt.kontaktperson_auftraggeber = directus_users.id;
ALTER TABLE objekt MODIFY kontaktperson_nutzung CHAR(36) NULL;	UPDATE objekt LEFT JOIN directus_users__old ON objekt.kontaktperson_nutzung = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET objekt.kontaktperson_nutzung = directus_users.id;
ALTER TABLE projekt MODIFY owner CHAR(36) NULL;	UPDATE projekt LEFT JOIN directus_users__old ON projekt.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET projekt.owner = directus_users.id;
ALTER TABLE projekt MODIFY modified_by CHAR(36) NULL;	UPDATE projekt LEFT JOIN directus_users__old ON projekt.modified_by = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET projekt.modified_by = directus_users.id;
ALTER TABLE projekt MODIFY auftraggeber CHAR(36) NULL;	UPDATE projekt LEFT JOIN directus_users__old ON projekt.auftraggeber = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET projekt.auftraggeber = directus_users.id;
ALTER TABLE projekt MODIFY verantwortliche_person_betrieb CHAR(36) NULL;	UPDATE projekt LEFT JOIN directus_users__old ON projekt.verantwortliche_person_betrieb = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET projekt.verantwortliche_person_betrieb = directus_users.id;
ALTER TABLE strom MODIFY owner CHAR(36) NULL;	UPDATE strom LEFT JOIN directus_users__old ON strom.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET strom.owner = directus_users.id;
ALTER TABLE telekom MODIFY owner CHAR(36) NULL;	UPDATE telekom LEFT JOIN directus_users__old ON telekom.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET telekom.owner = directus_users.id;
ALTER TABLE trp_client MODIFY modified_by CHAR(36) NULL;	UPDATE trp_client LEFT JOIN directus_users__old ON trp_client.modified_by = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET trp_client.modified_by = directus_users.id;
ALTER TABLE trp_order MODIFY modified_by CHAR(36) NULL;	UPDATE trp_order LEFT JOIN directus_users__old ON trp_order.modified_by = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET trp_order.modified_by = directus_users.id;
ALTER TABLE trp_order MODIFY owner CHAR(36) NULL;	UPDATE trp_order LEFT JOIN directus_users__old ON trp_order.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET trp_order.owner = directus_users.id;
ALTER TABLE trp_order_construction MODIFY owner CHAR(36) NULL;	UPDATE trp_order_construction LEFT JOIN directus_users__old ON trp_order_construction.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET trp_order_construction.owner = directus_users.id;
ALTER TABLE trp_order_goods MODIFY owner CHAR(36) NULL;	UPDATE trp_order_goods LEFT JOIN directus_users__old ON trp_order_goods.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET trp_order_goods.owner = directus_users.id;
ALTER TABLE trp_order_people MODIFY owner CHAR(36) NULL;	UPDATE trp_order_people LEFT JOIN directus_users__old ON trp_order_people.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET trp_order_people.owner = directus_users.id;
ALTER TABLE trp_tour MODIFY modified_by CHAR(36) NULL;	UPDATE trp_tour LEFT JOIN directus_users__old ON trp_tour.modified_by = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET trp_tour.modified_by = directus_users.id;
ALTER TABLE wasser MODIFY owner CHAR(36) NULL;	UPDATE wasser LEFT JOIN directus_users__old ON wasser.owner = directus_users__old.id LEFT JOIN directus_users ON directus_users.email = directus_users__old.email SET wasser.owner = directus_users.id;
```

### Relations etc. via tool

```bash
git clone git@github.com:directus-community/migration-tool.git tool
cd tool
npm install

cp ../dotenv .env
nano .env
node index.js -l
```

## TODO

- [x] User per SQL erstellen --> Ja
- [x] Können Vor-erstellte user per SSO einloggen? --> Ja
- [x] Testen, ob SSO jetzt ohne Vorerstellen von Benutzern geht --> Ja
- [ ] Migrate users with id 1, 7 (`external_id IS NULL`)
- [ ] Login von Logomat/TraMat
- [ ] Änderungen am Logomat/TraMat testen
- [ ] Rollen automatisch zuweisen bei SSO?
