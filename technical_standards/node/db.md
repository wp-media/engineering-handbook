---
notion_page: https://www.notion.so/wpmedia/Node-js-Database-Management-1b6ed22a22f0806a8018daead3f09513?pvs=4
title: Node.js - Database Management
---

# Database Management

Effective database management is critical for building reliable, performant Node.js applications. This guide outlines our approach to database connections, data access layers, and migrations in Express.js applications.

## Database Connection

We prioritize a simple, direct approach to database connectivity, starting with plain SQL queries and adding abstractions only when necessary.

### Using Plain SQL with node-postgres

For PostgreSQL databases, we use the [node-postgres](https://node-postgres.com/) library:

```bash
npm install pg
```

### Database Connection Setup

Set up your database connection in a dedicated module:

```javascript
// db/index.js
import pg from 'pg';
const { Pool } = pg;

export default function registerDatabaseInServiceLocator(serviceLocator) {
  // Create a connection pool
  const pool = new Pool({
    host: process.env.DB_HOST || 'localhost',
    port: process.env.DB_PORT || 5432,
    database: process.env.DB_NAME || 'myapp',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || '',
    max: 20, // Maximum number of clients in the pool
    idleTimeoutMillis: 30000,
  });

  // Test the connection
  pool.on('connect', () => {
    console.log('Connected to PostgreSQL database');
  });

  pool.on('error', (err) => {
    console.error('Unexpected error on idle client', err);
    process.exit(-1);
  });

  // Define the database interface
  const db = {
    query: (text, params) => pool.query(text, params),
    getClient: async () => {
      const client = await pool.connect();
      const query = client.query;
      const release = client.release;

      // Set a timeout of 5 seconds, after which we will log this client's last query
      const timeout = setTimeout(() => {
        console.error('A client has been checked out for more than 5 seconds!');
        console.error(`The last executed query on this client was: ${client.lastQuery}`);
      }, 5000);

      // Monkey patch the query method to keep track of the last query executed
      client.query = (...args) => {
        client.lastQuery = args;
        return query.apply(client, args);
      };

      client.release = () => {
        clearTimeout(timeout);
        client.query = query;
        client.release = release;
        return release.apply(client);
      };

      return client;
    },
    end: () => pool.end(),
  };

  return db;
}
```

### Integrating with Service Locator

Register the database in your service locator during application startup:

```javascript
// serviceLocator.js
import registerDatabaseInServiceLocator from './db/index.js';

export const ServiceLocator = {};

// Register services
ServiceLocator.db = registerDatabaseInServiceLocator(ServiceLocator);

// Get database service
ServiceLocator.getDatabase = () => ServiceLocator.db;
```

### Using the Database Connection

Access the database through the service locator:

```javascript
// Example usage in a service
import { ServiceLocator } from '../serviceLocator.js';

export async function getSiteById(id) {
  const db = ServiceLocator.getDatabase();
  
  const query = {
    text: 'SELECT * FROM sites WHERE id = $1',
    values: [id],
  };
  
  const result = await db.query(query);
  return result.rows[0];
}
```

## DAL Abstraction

The Data Access Layer (DAL) provides a clean abstraction over database operations, keeping SQL queries separate from business logic.

### DAL Structure

Organize your DAL by domain entities:

```
/db
  /dal
    site.js
    user.js
    report.js
  index.js
```

### Implementing DAL Modules

Create entity-specific DAL modules:

```javascript
// db/dal/site.js
export default function createSiteDAL(db) {
  return {
    /**
     * Get all sites
     * @returns {Promise<Array>} Array of site objects
     */
    async getAll() {
      const result = await db.query('SELECT * FROM sites ORDER BY name');
      return result.rows;
    },
    
    /**
     * Get a site by ID
     * @param {number} id - Site ID
     * @returns {Promise<Object|null>} Site object or null if not found
     */
    async getById(id) {
      const result = await db.query(
        'SELECT * FROM sites WHERE id = $1',
        [id]
      );
      return result.rows[0] || null;
    },
    
    /**
     * Create a new site
     * @param {Object} site - Site data
     * @returns {Promise<Object>} Created site with ID
     */
    async create(site) {
      const result = await db.query(
        'INSERT INTO sites (name, url, created_at) VALUES ($1, $2, NOW()) RETURNING *',
        [site.name, site.url]
      );
      return result.rows[0];
    },
    
    /**
     * Update a site
     * @param {number} id - Site ID
     * @param {Object} site - Updated site data
     * @returns {Promise<Object|null>} Updated site or null if not found
     */
    async update(id, site) {
      const result = await db.query(
        'UPDATE sites SET name = $1, url = $2, updated_at = NOW() WHERE id = $3 RETURNING *',
        [site.name, site.url, id]
      );
      return result.rows[0] || null;
    },
    
    /**
     * Delete a site
     * @param {number} id - Site ID
     * @returns {Promise<boolean>} True if deleted, false if not found
     */
    async delete(id) {
      const result = await db.query(
        'DELETE FROM sites WHERE id = $1 RETURNING id',
        [id]
      );
      return result.rowCount > 0;
    }
  };
}
```

### Registering DAL Modules

Register DAL modules with the service locator:

```javascript
// serviceLocator.js
import registerDatabaseInServiceLocator from './db/index.js';
import createSiteDAL from './db/dal/site.js';
import createUserDAL from './db/dal/user.js';

export const ServiceLocator = {};

// Register database
ServiceLocator.db = registerDatabaseInServiceLocator(ServiceLocator);

// Register DAL modules
ServiceLocator.siteDAL = createSiteDAL(ServiceLocator.db);
ServiceLocator.userDAL = createUserDAL(ServiceLocator.db);

// Getter methods
ServiceLocator.getDatabase = () => ServiceLocator.db;
ServiceLocator.getSiteDAL = () => ServiceLocator.siteDAL;
ServiceLocator.getUserDAL = () => ServiceLocator.userDAL;
```

### Using the DAL in Services

Create service modules that use the DAL:

```javascript
// services/site.js
import { ServiceLocator } from '../serviceLocator.js';

export function createSiteService() {
  const siteDAL = ServiceLocator.getSiteDAL();
  
  return {
    async getAll() {
      return await siteDAL.getAll();
    },
    
    async getById(id) {
      return await siteDAL.getById(id);
    },
    
    async create(siteData) {
      // Validate or transform data as needed
      return await siteDAL.create(siteData);
    },
    
    async update(id, siteData) {
      // Validate or transform data as needed
      return await siteDAL.update(id, siteData);
    },
    
    async delete(id) {
      return await siteDAL.delete(id);
    }
  };
}

// Register in service locator
ServiceLocator.siteService = createSiteService();
ServiceLocator.getSiteService = () => ServiceLocator.siteService;
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
