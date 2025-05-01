---
notion_page: https://www.notion.so/wpmedia/Node-js-Database-Management-1b6ed22a22f0806a8018daead3f09513?pvs=4
title: Node.js - Database Management
---

# Database Management

Effective database management is critical for building reliable, performant and understandable Node.js applications. This guide outlines our recommended approach to database connections, data access layers, and migrations in Node.js applications.

## Database Connection

We encourage a simple, direct approach to database connectivity, starting with plain SQL queries and adding abstractions only when necessary. We believe that having a good understanding of the database behavior is key to building a reliable and scalable application. Abstractions through tools like ORM can be beneficial for complex apps and developers with a good understanding of the underlying logic, but they can be detrimental for simpler projects without advanced needs of database usage.

### Using Plain SQL with node-postgres

For plain SQL with PostgreSQL databases, we recommend using the [node-postgres](https://node-postgres.com/) library:

```bash
npm install pg
```

### Database Connection Class

Set up your database connection in a dedicated class:

```javascript
//Database.js
import pg from "pg";

export default class Database {
  constructor(dbConfig) {
    this.pool = new pg.Pool(dbConfig);
  }

  /**
   * Close the connection pool.
   *
   * Since we are using connection pooling, we can risk leaking connections. If
   * that happens it would leave the container running in a broken state.
   *
   * @param {number} timeout How many ms to wait until releasing clients
   */
  async close(timeout = 0) {
    let timeoutHandle;
    if (typeof timeout === "number" && timeout > 0) {
      timeoutHandle = setTimeout(() => {
        console.error(
          `Timeout after ${timeout} ms. Forcefully releasing all clients...`,
        );
        this.pool._clients.forEach((client) => client.release());
      }, timeout);
    }
    await this.pool.end();
    if (timeoutHandle) {
      clearTimeout(timeoutHandle);
    }
  }

  async getClient() {
    return this.pool.connect();
  }
  /**
  * Commented lines in this method demonstrate how debugging 
  * & reporting can easily be added thanks to the centralized
  * query & DB management
  */
  async query(text, params) {
    // const start = Date.now();
    const res = await this.pool.query(text, params);
    // const duration = Date.now() - start;
    // console.log("executed query", { text, duration, rows: res.rowCount });
    return res;
  }
}
```

The `dbConfig` should get its values from environment variables to facilitate deployments on various contexts:

```javascript
      user: getEnv("POSTGRES_USER"),
      password: getEnv("POSTGRES_PASSWORD"),
      host: getEnv("POSTGRES_HOST"),
      port: getEnv("POSTGRES_PORT"),
      database: getEnv("POSTGRES_DB"),
```

The Database connection class and its configuration can be easily put together with the [Service Locator pattern](service_locator.md).

### Integrating with Service Locator

Register the database in your service locator during application startup:

```javascript
//service-locator.js
import ServiceLocator from "dislocator";
import registerDatabaseService from "./services/database.js";
import registerConfigService from "./services/config.js";
import registerDalService from "./services/dal.js";
export default function createServiceLocator() {
  const serviceLocator = new ServiceLocator();
  serviceLocator
    .use(registerConfigService)
    .use(registerDalService)
    .register("db", registerDatabaseService(serviceLocator));
  return serviceLocator;
}
```

```javascript
//services/database.js
import Database from "../Database.js";

export default function registerDatabaseService({ config }) {
  return new Database(config.db);
}
```

The `dbConfig` configuration mentioned above should be part of `./services/config.js` in this example:

```javascript
//services/config.js
export default function registerConfigService(serviceLocator) {
  serviceLocator.register("config", {
    db: {
      user: getEnv("POSTGRES_USER"),
      password: getEnv("POSTGRES_PASSWORD"),
      host: getEnv("POSTGRES_HOST"),
      port: getEnv("POSTGRES_PORT"),
      database: getEnv("POSTGRES_DB"),
    },
  });
```

## DAL Abstraction

The Data Access Layer (DAL) provides a clean abstraction over database operations, keeping SQL queries separate from business logic.

### DAL Structure

Organize your DAL by domain entities:

```
/handlers
/services
/dal
  AccountDAL.js
  DAL.js
  ReportDAL.js
  ScanQueueDAL.js
app.js
```

A DataAccessLayer class defined in `DAL.js` allows to list and instantiate all the existing DAL modules as follows:
- The service locator uses the DAL service
- The DAL service register the DataAccessLayer class
- The DataAccessLayer class provides each DAL module.

```javascript
//services/dal.js
import DataAccessLayer from "../dal/DAL.js";
export default function registerDalService(serviceLocator) {
  serviceLocator.register(
    "dal",
    () => new DataAccessLayer(serviceLocator.get("db")),
  );
}
```

```javascript
//dal/DAL.js
import { ProvisionerTokensDAL } from "./ProvisionerTokenDAL.js";
import { ReportDAL } from "./ReportDAL.js";
import { ScanQueueDAL } from "./ScanQueueDAL.js";
import { AccountDAL } from "./AccountDAL.js";

export default class DataAccessLayer {
  constructor(db) {
    this.db = db;

    this.provisionerTokens = new ProvisionerTokensDAL(db);
    this.scanQueue = new ScanQueueDAL(db);
    this.account = new AccountDAL(db);
    this.report = new ReportDAL(db);
  }
}
```

### Implementing DAL Modules

Create entity-specific DAL modules:

```javascript
// dal/AccountDAL.js
export class AccountDAL {
  constructor(db) {
    this.db = db;
  }

  async createAccount(payload) {
    const result = await this.db.query(
      ` INSERT INTO accounts (email)
        VALUES ($1)
        RETURNING uuid;`,
      [payload.email],
    );

    if (result.rowCount !== 1) {
      throw new Error("Unexpected error during account creation");
    }

    return result.rows[0].uuid;
  }
}

```

### Using the DAL

Create service modules that use the DAL:

```javascript
// handlers/account.js
import express from "express";
import httpErrors from "http-errors";
import validation from "one-validation";

export default function accountsHandler(serviceLocator) {
  const app = express.Router();
  const dal = serviceLocator.dal;

  app.post("/", async (req, res) => {
    if (
      !req.body ||
      typeof req.body !== "object" ||
      !req.body.email ||
      typeof req.body.email !== "string" ||
      !validation.email.test(req.body.email)
    ) {
      throw new httpErrors.BadRequest("Invalid request body");
    }

    const result = await dal.account.createAccount({
      email: req.body.email,
    });

    return res.status(201).json({ uuid: result });
  });

```

## Database Migrations

Database migrations help manage schema changes over time. We recommend plain SQL for migrations to maintain full control and clarity over the operations we are doing. Migrations represent the successive changes done to the data model over time. We also encourage maintaining the current data model in a dedicated file to quickly understand the current state.

### Migration Structure

Organize migrations in a dedicated directory:

```
/database
  /migrations
    001-initial-schema.sql
    002-add-user-roles.sql
    003-add-site-metrics.sql
  schema.sql
```

### Writing SQL Migrations

Use plain SQL for migrations:

```sql
CREATE TYPE scan_queue_status AS ENUM ('pending', 'completed', 'running', 'failed');
CREATE TYPE scan_queue_type AS ENUM ('main', 'single');

CREATE TABLE scan_queues (
    id SERIAL PRIMARY KEY,
    uuid UUID NOT NULL DEFAULT gen_random_uuid() UNIQUE,
    job_id UUID NOT NULL DEFAULT gen_random_uuid(),
    account_id UUID NULL,
    site_id UUID NULL,
    page_id UUID NULL,
    url TEXT NOT NULL,
    type scan_queue_type NOT NULL DEFAULT 'single',
    status scan_queue_status NOT NULL DEFAULT 'pending',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_status_created_at ON scan_queues (status, created_at);
CREATE INDEX IF NOT EXISTS idx_updated_at ON scan_queues (updated_at);
CREATE INDEX IF NOT EXISTS idx_job_id ON scan_queues (job_id);
CREATE INDEX IF NOT EXISTS idx_url ON scan_queues (url);

```

### Managing migrations

Migrations should not be applied multiple times, therefore it is key to track which migration has already been applied to a given database. For this, we recommend using a dedicated table.

```sql
  CREATE TABLE migrations (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    filename TEXT NOT NULL,
    file_content TEXT NOT NULL
  );
```

Then, a command to look for migration files to apply and run them can be created as follows:

```javascript
import path from "node:path";
import fs from "node:fs/promises";
import {
  dbConfig,
  isDevelopment,
  isProduction,
} from "./config-from-environment.js";
import pg from "pg";

const DATABASE_DIR = path.resolve(import.meta.dirname, "../../database");
const DATABASE_MIGRATIONS_DIR = path.resolve(DATABASE_DIR, "migrations");

const MIGRATIONS_TABLE_SCHEMA = `
  CREATE TABLE migrations (
    id SERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    filename TEXT NOT NULL,
    file_content TEXT NOT NULL
  );
`;

async function findMigrationFiles(dirname) {
  let migrationFiles = await fs.readdir(dirname);
  migrationFiles = migrationFiles.sort();
  return migrationFiles;
}

async function findAppliedMigrations(client) {
  const res = await client.query("SELECT * FROM migrations;");

  let appliedMigrationsSet = new Set();

  for (const migration of res.rows) {
    appliedMigrationsSet.add(migration.filename);
  }

  return appliedMigrationsSet;
}

async function checkTableExists(client, tableName) {
  try {
    const res = await client.query(
      `SELECT EXISTS (
              SELECT FROM information_schema.tables
              WHERE table_name = $1
          ) AS table_exists;`,
      [tableName],
    );

    return res.rows[0].table_exists;
  } catch (err) {
    console.error("Error checking table existence:", err);
    return false;
  }
}

const ADVISORY_LOCK_ID = "7010016611124574584";

export default async function commandMigrate() {
  const client = new pg.Client(dbConfig);
  await client.connect();

  try {
    if (isProduction) {
      console.log("Waiting for advisory lock...");
      await client.query("SELECT pg_advisory_lock($1)", [ADVISORY_LOCK_ID]);
    }

    const hasMigrationsTable = await checkTableExists(client, "migrations");

    if (!hasMigrationsTable) {
      console.error("Found no migrations table. Creating it...");
      await client.query(MIGRATIONS_TABLE_SCHEMA);
    }

    const migrationFiles = await findMigrationFiles(DATABASE_MIGRATIONS_DIR);
    const appliedMigrationsSet = await findAppliedMigrations(client);
    let migrationsToApply = migrationFiles.filter(
      (file) => !appliedMigrationsSet.has(file),
    );

    if (migrationsToApply.length === 0) {
      console.log("No migrations to apply.");
      return;
    }

    try {
      await client.query("BEGIN");

      for (const migrationFile of migrationsToApply) {
        let absPath = path.resolve(DATABASE_MIGRATIONS_DIR, migrationFile);
        console.log(`Applying ${migrationFile} (${absPath})`);

        let fileContent = await fs.readFile(absPath, "utf-8");

        await client.query(fileContent);
        await client.query({
          text: "INSERT INTO migrations (filename, file_content) VALUES ($1, $2)",
          values: [migrationFile, fileContent],
        });
      }

      await client.query("COMMIT");
    } catch (e) {
      await client.query("ROLLBACK");
      throw e;
    }

    console.log("Successfully applied all migrations.");

    if (isDevelopment) {
      const { syncSchema } = await import("./command-sync-schema.js");
      await syncSchema();
    }
  } finally {
    await client.end();
  }

  console.log("Done");
}
```

The migration command above can then be used through a CLI in the app to manually trigger the migrations. 

### Examples

The approach presented in this page is used and illustrated in [this private repository](https://gitlab.group.one/rankmath/seo-platform/-/commit/0c02b78cd5ec496929373bd7022113e5e4526581). There are further elements around the CLI to facilitate management of the database, such as automatically generating the schema.sql file thanks to the `command-sync-schema.js`.

### Frictions between git and schema.sql

When working on database changes in branches off of the main development branch, merging changes can result in a wrong `schema.sql` file. When merging or reviewing branches, extra attention must be paid to this file, to ensure the result is as expected. In case the file does not correspond to what is expected, a quick remediation is to destroy the database, start it again from scratch and run the migrations.

```bash
db destroy
db start
db migrate
```
