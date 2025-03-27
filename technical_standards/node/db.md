---
notion_page: https://www.notion.so/wpmedia/Node-js-Database-Management-1b6ed22a22f0806a8018daead3f09513?pvs=4
title: Node.js - Database Management
---

# Database Management

Effective database management is critical for building reliable, performant Node.js applications. This guide outlines our recommended approach to database connections, data access layers, and migrations in Node.js applications.

## Database Connection

We encourage a simple, direct approach to database connectivity, starting with plain SQL queries and adding abstractions only when necessary. We believe that having a good understanding of the database behavior is key to building a reliable and scalable application. Abstractions through tools like ORM can be beneficial for complex apps and developers with a good understanding of the underlying logic, but they can be detrimental for simpler projects without advanced needs of DB usage.

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

  async query(text, params) {
    const start = Date.now();
    const res = await this.pool.query(text, params);
    const duration = Date.now() - start;
    console.log("executed query", { text, duration, rows: res.rowCount });
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

The Database connection class and its configuration can be easily put together with the Service Locator pattern.

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

The `dbConfig` configuration mentionned above should be part of `./services/config.js` in this example:

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

Database migrations help manage schema changes over time. We use plain SQL for migrations to maintain full control and clarity.

### Migration Structure

Organize migrations in a dedicated directory:

```
/db
  /migrations
    001-initial-schema.sql
    002-add-user-roles.sql
    003-add-site-metrics.sql
  /dal
  index.js
```

### Writing SQL Migrations

Use plain SQL for migrations:

```sql
-- migrations/001-initial-schema.sql

-- Create sites table
CREATE TABLE IF NOT EXISTS sites (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  url VARCHAR(2000) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP
);

-- Create users table
CREATE TABLE IF NOT EXISTS users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  name VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP
);

-- Create user_sites table for many-to-many relationship
CREATE TABLE IF NOT EXISTS user_sites (
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  site_id INTEGER NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
  role VARCHAR(50) NOT NULL DEFAULT 'viewer',
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_id, site_id)
);
```

### Migration Runner

Create a utility to run migrations:

```javascript
// db/migrations/runner.js
import fs from 'fs/promises';
import path from 'path';
import { fileURLToPath } from 'url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export async function runMigrations(db) {
  try {
    // Create migrations table if it doesn't exist
    await db.query(`
      CREATE TABLE IF NOT EXISTS migrations (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255) NOT NULL UNIQUE,
        applied_at TIMESTAMP NOT NULL DEFAULT NOW()
      )
    `);
    
    // Get applied migrations
    const { rows: appliedMigrations } = await db.query(
      'SELECT name FROM migrations ORDER BY id'
    );
    const appliedMigrationNames = appliedMigrations.map(m => m.name);
    
    // Get all migration files
    const migrationFiles = await fs.readdir(__dirname);
    const sqlFiles = migrationFiles
      .filter(file => file.endsWith('.sql'))
      .sort(); // Ensure ordered execution
    
    // Run pending migrations
    for (const file of sqlFiles) {
      if (!appliedMigrationNames.includes(file)) {
        console.log(`Applying migration: ${file}`);
        
        // Get migration SQL
        const filePath = path.join(__dirname, file);
        const sql = await fs.readFile(filePath, 'utf8');
        
        // Start a transaction
        const client = await db.getClient();
        try {
          await client.query('BEGIN');
          
          // Run the migration
          await client.query(sql);
          
          // Record the migration
          await client.query(
            'INSERT INTO migrations (name) VALUES ($1)',
            [file]
          );
          
          await client.query('COMMIT');
          console.log(`Migration applied: ${file}`);
        } catch (error) {
          await client.query('ROLLBACK');
          console.error(`Migration failed: ${file}`, error);
          throw error;
        } finally {
          client.release();
        }
      }
    }
    
    console.log('All migrations applied successfully');
  } catch (error) {
    console.error('Migration runner error:', error);
    throw error;
  }
}
```

### Running Migrations During Startup

Run migrations during application startup:

```javascript
// app.js
import express from 'express';
import { ServiceLocator } from './serviceLocator.js';
import { runMigrations } from './db/migrations/runner.js';

async function startServer() {
  try {
    // Run database migrations
    const db = ServiceLocator.getDatabase();
    await runMigrations(db);
    
    // Create Express app
    const app = express();
    
    // ... configure app ...
    
    // Start server
    const PORT = process.env.PORT || 3000;
    app.listen(PORT, () => {
      console.log(`Server running on port ${PORT}`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
}

startServer();
```

## Transaction Management

For operations that require multiple database changes, use transactions:

```javascript
// Example of a transaction in a DAL method
async createUserWithSites(userData, siteIds) {
  const client = await db.getClient();
  
  try {
    await client.query('BEGIN');
    
    // Create user
    const userResult = await client.query(
      'INSERT INTO users (name, email, password_hash) VALUES ($1, $2, $3) RETURNING id',
      [userData.name, userData.email, userData.passwordHash]
    );
    const userId = userResult.rows[0].id;
    
    // Associate user with sites
    for (const siteId of siteIds) {
      await client.query(
        'INSERT INTO user_sites (user_id, site_id, role) VALUES ($1, $2, $3)',
        [userId, siteId, 'editor']
      );
    }
    
    await client.query('COMMIT');
    
    // Get the complete user
    const result = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
    return result.rows[0];
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```
