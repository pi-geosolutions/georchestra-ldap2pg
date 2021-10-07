# LDAP2PG -- customized for geOrchestra

_Use excellent code from [ldap2pg](https://github.com/dalibo/ldap2pg/) to configure, from the console, the schemas and roles in your database._

_**Please be careful when using it: run some tests before running it in production. Review the config. Don't use it blindly, we will not be held responsible of what can happen**_

---

Most geOrchestra instances use a PostGIS DB to store geospatial (and also non-geospatial) data, for publication in GeoServer mostly.

This DB needs to be accessible by some editors of your geOrchestra instance.
Instead of configuring postgresql accounts, schemas and privileges manually,
you can

1. Configure your DB to lookup for users [using LDAP](https://www.postgresql.org/docs/9.3/auth-methods.html#AUTH-LDAP)
2. Configure the accounts, schemas and privileges using the console

This tool addresses the point 2:
- a small script (georchestra-custom/src/main.py) synchronizes the schemas
- then ldap2pg synchronizes the users accounts & privileges

All this based on roles, defined in the console.


## Known limitations

For now, it is tested and working on a separate postgresql cluster (don't use the main geOrchestra DB), with a single database, named `georchestra`.

While ldap2pg is capable of working with several databases, the schema synchronization script is not configured for this. And the default ldap2pg.yml config would probably need some revisiting, too.

Any contributions on this are welcome !


## Using it

### Configure the roles in the console

The expected syntax is `PGSQL_SCHEMA_[SCHEMANAME]_[PRIVILEGE]`.

For instance, `PGSQL_SCHEMA_KSK_WRITER` will give the user the writer role in the schema ksk. If it does not exist, the ksk schema will be created. If, at some point, no role remains that target the ksk schema, it will be removed _if empty_ (`DROP SCHEMA ... RESTRICT;`)

Allowed chunks:
- `PGSQL_SCHEMA_` is a fixed chunk, that allows to filter the relevant roles
- `[SCHEMANAME]` can take any reasonnable value (uppercase non-accetnuated letters, and also _ and -)
- `[PRIVILEGE]` can be
  - READER: the user has read access (SELECT mostly) in the schema
  - PUBLISHER: the user has insert and update access in the schema, but cannot create any table
  - WRITER: the user is actually an owner of the schema (as defined by default in ldap2pg), meaning he can create tables and other objects and insert/update data.



### Run the sync

- Build the container (this is all expected to be run in a container, but you probably can figure out something without: just run the georchestra-custom/src/main.py for schema sync, then this modified ldap2pg)

  From *the root of this repo*, run

        docker build -t georchestra/ldap2pg:5.6 -f georchestra-custom/Dockerfile .

- Run it (one shot)
      docker run --rm --tty \
        -e PGDSN=postgres://georchestra@192.168.1.70:5434/georchestra \
        -e PGPASSWORD_FILE=/workspace/pgpasswd \
        -e LDAPURI=ldap://192.168.1.70:3389 \
        -e LDAPBASEDN=dc=georchestra,dc=org \
        -e LDAPBINDDN=cn=admin,dc=georchestra,dc=org \
        -e LDAPPASSWORD_FILE=/workspace/ldappasswd \
        -e DRY="" \
        -e COLOR=1 \
        georchestra/ldap2pg:5.6

    In production, you will want to program it as a cron task or similar.

#### Environment variables
There are quite a lot of available environment variables

**Compulsory**:
- PGDSN: the connection string to the postgresql DB. Look at the example above for a valid syntax
- PGPASSWORD: password for the user documented in the DSN string. You can use PGPASSWORD_FILE to provide it through a docker secret
- LDAPURI: the connection string to the LDAP DB. Look at the example above for a valid syntax
- LDAPBASEDN: the LDAP base DN. Look at the example above for a valid syntax
- LDAPBINDDN: the LDAP admin user. Look at the example above for a valid syntax
- LDAPPASSWORD_FILE: the LDAP admin user's password. You can use LDAPPASSWORD_FILE to provide it through a docker secret

**Optional**:
- LDAP_ROLE_REGEX: the regular expression used to extract the roles from the LDAP DB. See below for the default value
- and quite a few env vars supported by [ldap2pg] (https://ldap2pg.readthedocs.io/en/latest/cli/#environment-variables). To name a few:
  - DRY: if set to `1`, it will run in dry mode (not change anything). If set to `''`, changes are applied (what we want in production)
  - VERBOSITY: set it to `DEBUG` to get a more verbose output
  - COLOR: set it to `1` to get a colored output (docker need the `--tty` option, too)


### Changing the default config

This is all very configurable if you feel like a rebel. There are mostly 2 places to look at:

#### LDAP_ROLE_REGEX environment variable

You can change the default regular expression applied on the console roles. The default value is
`LDAP_ROLE_REGEX="^PGSQL_SCHEMA_([A-Z-_]+)_(READER|WRITER|PUBLISHER)$"`
If you change it, you will probably need to adjust the ldap2pg.yml config file too

#### ldap2pg.yml

The default config is the georchestra-custom/ldap2pg.yml file. In the docker image, it is copied as /etc/ldap2pg.yml.
To override it, the easiest way is to mount a /workspace volume, in which will be your ldap2pg.yml alternative file. See https://ldap2pg.readthedocs.io/en/latest/config/#file-location

If you feel like playing with configuration, let's have a look at https://ldap2pg.readthedocs.io/en/latest/config/.
