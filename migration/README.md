# Test Directus v8->v9 migration

https://docs.directus.io/configuration/upgrades-migrations

https://github.com/directus-community/migration-tool

https://docs.directus.io/configuration/config-options

## Setup playground v9 env

```bash
docker-compose up database # give MySQL time to seed
docker-compose up
```

* Directus: http://localhost:8080 (see `docker-compose.yml` for credentials)
* Adminer: http://localhost:8081

## Migration Tool

```
git clone git@github.com:directus-community/migration-tool.git
cd migration-tool/
npm install

cp dotenv migration-tool/.env
nano migration-tool/.env
cd migration-tool/
node index.js
```

**Does not work as of 2022-03-13 (migration tool `2fdae2ed`)!**

## Manual migration

1. `docker-compose up database adminer`
2. Import backup
3. Drop all directus tables:
```sql
SELECT CONCAT('DROP TABLE ', table_name, ';')
FROM information_schema.tables
WHERE TABLE_SCHEMA = 'directus'
AND table_name LIKE 'directus_%'
```
4. `docker-compose up`
