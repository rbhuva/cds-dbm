# cds-dbm

_cds-dbm_ is a node package that adds **automated delta deployment** and (as a planned, but not yet integrated feature) **full database migration support** to the Node.js Service SDK (<a href="https://www.npmjs.com/package/@sap/cds">@sap/cds</a>) of the <a href="https://cap.cloud.sap/docs/about/">**SAP Cloud Application Programming Model**</a>.

The library offers two ways of handling database deployments:<br>
You can either use automated delta deployments of the current cds data model that are in line with the default development workflow in cap projects. For more complex applications and scenarios, there will also be integrated support of a full fledged database migration concept.
<br> For both scenarios _cds-dbm_ is relying on the popular Java framework <a href="https://www.liquibase.org/">liquibase</a> to handle (most) of the database activities.

Currently _cds-dbm_ offers support for the following databases:

- PostgreSQL (<a href="https://github.com/sapmentors/cds-pg">cds-pg</a>)

Support for other databases is planned whenever a corresponding cds adapter library is available.

<details><summary>Why does <i>cds-dbm</i> (currently) not support SAP HANA?</summary>
<p>

As SAP HANA is a first class citizen in CAP, SAP offers its own deployment solution (<a href="https://www.npmjs.com/package/@sap/hdi-deploy">@sap/hdi-deploy</a>). With CAP it is possible to directly compile a data model into SAP HANA fragments (.hdbtable, etc.), which can then be deployed by the hdi-deploy module taking care of all the important stuff (delta handling, hdi-management on XSA or SAP Cloud Platform, etc.).
<br>
Nevertheless it may be suitable to use the <a href="https://github.com/liquibase/liquibase-hanadb">liquibase-hanadb</a> adapter to add an alternative deployment solution. If so, support might be added in the future.

</p>
</details>

## Current status

This is an alpha version not ready to be used in production environments.
The rough plan ahead:

**Project setup**

- [x] inital project setup including TypeScript and liquibase
- [x] release as npm package
- [ ] fully leverage github build pipeline (github actions, ts > js)

**Basic features**

- [x] add automated deployment model
- [x] add support for auto-undeployment (implicit drop)
- [x] add support for updeployment files (explicit drop)
- [x] add data import of csv files
- [ ] add `build` task for SCP CF deployment

**Advanced features**

- [ ] add support for multitenancy
- [ ] add advanced deployment model including migrations

**Database support**

- [x] add PostgresSQL adapter (<a href="https://github.com/sapmentors/cds-pg">cds-pg</a>)
- [ ] verify and maybe add support for SQLite

Contributions are welcome. Details on how to contribute will be added soon.

## Prerequisites

Since the project uses liquibase internally, a Java Runtime Environment (JRE) in at least version 8 is required on your system.

## Usage in your CAP project

Simply add this package to your [CAP](https://cap.cloud.sap/docs/) project by running:

```bash
npm install cds-dbm
```

---

## Automated delta deployments

> TODO: Add full description

In the meantime some notes on the delta processing:

1. clone current db schema `cds.migrations.db.schema.default` into `cds.migrations.db.schema.clone`
2. drop all cds based views from clone schema because updates on views do not work in the way liquibase is handling this via `CREATE OR REPLACE VIEW`
   (https://liquibase.jira.com/browse/CORE-2377)
3. deploy the full cds model to the reference schema `cds.migrations.db.schema.reference`
4. let liquibase create a diff between the clone and reference schema (including the recreation of the dropped views)
5. do some adjustments on the changelog (handle undeployment stuff, fix order of things)
6. finally deploy changelog to current schema
7. load data from csv files (if requested)

### Usage with cds-pg (PostgreSQL)

_cds-dbm_ requires some additional configuration added to your package.json:

```JSON
  "cds": {
    //...
    "migrations": {
      "db": {
        "schema": {
          "default": "public",
          "clone": "_cdsdbm_clone",
          "reference": "_cdsdbm_ref"
        },
        "deploy": {
          "tmpFile": "tmp/_autodeploy.json",
          "undeployFile": "db/undeploy.json"
        }
      }
    }
  }
```

### Dropping tables/views

cds-dbm follows a safe approach and does not drop any database tables during deployment. Thus, old tables will still be around even if they are not part of your data model anymore.

You can either remove them manually or rely on _cds-dbm_ to handle this for you.

**Undeployment file**

An undeployment file makes it possible to specifically list views and tables that should be dropped from the database schema during the deployment.

The undeployment file's path needs to be specified in the `package.json` configuration (`cds.migrations.deploy.undeployFile`)

```json
# An example undeploy.json:

{
    "views": [],
    "tables": [
        "csw_beers",
        "csw_anotherTable"
    ]
}
```

**auto-undeployment option**

While an `undeploy.json` file gives you fine grained control, it is also possible to automatically remove tables/views from the database schema. When using the `auto-undeploy` flag during deployment, _cds-dbm_ will take the cds model as the reference and remove all other existing tables/views.

### Commands

The following commands exists for working the _cds-dbm_ in the automated delta deployment mode:

**Currently tall tasks must be called with npx**

```bash
npx cds-dbm <task>
```

#### `deploy`

Performs a delta deployment of the current cds data model to the database. By default no csv files are loaded. If this is required, the load strategy has to be defined (`load-via`).

**Usage**

```bash
cds-dbm deploy
```

**Flags**

- `create-db` (_boolean_) - If set, the deploy task tries to create the database before actually deploying the data model. The deployment will not break, if the database has already been created before.
- `auto-undeploy` (_boolean_) - **WARNING**: Drops all tables not known to your data model from the database. This should **only** be used if your cds includes all tables/views in your db (schema). Otherwise it is highly recommended to use an undeployment file.
- `load-via` (_string_) - Can be either `full` (truncate and insert) or `delta` (check for existing records, then update or insert)
- `dry` (_boolean_) - Does not apply the SQL to the database but logs it to stdout

**Examples**

```bash
cds-dbm deploy
cds-dbm deploy --create-db
cds-dbm deploy --load-via delta
cds-dbm deploy --auto-undeploy
cds-dbm deploy --auto-undeploy --dry
```

#### `load`

Loads data from CSV files into the database.
The following conventions apply (according to the default _@sap/cds_ conventions at https://cap.cloud.sap/docs/guides/databases):

- The files must be located in folders db/csv, db/data/, or db/src/csv.
- They contain data for one entity each. File names must follow the pattern <namespace>-<entity>.csv, for example, my.bookshop-Books.csv.
- They must start with a header line that lists the needed element names.

Different to the default mechanism in _@sap/cds_ (which supports only `full` loads), _cds_dbm_ offers two loading strategies:

- `full` – Truncates a table and then inserts all the data from the CSV file
- `delta` – Checks, if an existing record with the same key exists and performs an update. All other records in the table will be left untouched.

**Usage**

```bash
cds-dbm load --via full
cds-dbm load --via delta
```

**Flags**

- `via` (_string_) - Can be either `full` (truncate and insert) or `delta` (check for existing records, then update or insert)

#### `drop`

Drops all tables and views in your data model from the database. If the `all` parameter is given, then everything in the whole schema will be dropped, not only the cds specific entities.

**Usage**

```bash
cds-dbm drop
```

**Flags**

- `all` (_boolean_) - If set, the whole content of the database/schema is being dropped.

**Examples**

```bash
cds-dbm drop
cds-dbm drop --all
```

#### `diff`

Generates a descriptive text file containing all the differences between the defined cds model and the current status of the database.

**Usage**

```bash
cds-dbm diff
```

**Flags**

- `file` (_string_) - The file path of the diff file. If not set this will default to `<project-root>/diff.txt`

**Examples**

```bash
cds-dbm diff
cds-dbm diff --file db/diff.txt
```

---

## Versioned database development using migrations

_Not yet implemented_

## Sponsors

Thank you to **p36 (https://p36.io/)** for sponsoring this project.
