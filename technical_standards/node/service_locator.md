---
notion_page: https://www.notion.so/wpmedia/Node-js-Service-Locator-Pattern-1b6ed22a22f080de90ecc6c627805dbc?pvs=4
title: Node.js - Service Locator Pattern
---

A well-structured application architecture is essential for building maintainable, scalable Node.js applications. This guide outlines our approach to organizing dependencies and implementing the Service Locator pattern in Express.js applications.

## Service Locator Pattern

The Service Locator pattern provides a centralized registry of services, making dependencies available throughout the application while maintaining loose coupling between components.

### Core Implementation

Create a service locator module to manage application-wide dependencies:

```javascript
// serviceLocator.js
export const ServiceLocator = {};

// Store for registered services
const services = new Map();

// Register a service
ServiceLocator.register = (name, instance) => {
  services.set(name, instance);
  return instance;
};

// Get a registered service
ServiceLocator.get = (name) => {
  if (!services.has(name)) {
    throw new Error(`Service '${name}' not registered`);
  }
  return services.get(name);
};

// Check if a service is registered
ServiceLocator.has = (name) => {
  return services.has(name);
};

// Convenience methods for common services
ServiceLocator.getDatabase = () => ServiceLocator.get('db');
ServiceLocator.getLogger = () => ServiceLocator.get('logger');
ServiceLocator.getConfig = () => ServiceLocator.get('config');
```

### Service Registration

Register services during application startup:

```javascript
// app.js
import express from 'express';
import { ServiceLocator } from './serviceLocator.js';
import registerDatabaseService from './services/database.js';
import registerLoggerService from './services/logger.js';
import registerConfigService from './services/config.js';
import registerSiteService from './services/site.js';
import registerUserService from './services/user.js';

async function initializeServices() {
  // Register fundamental services
  const config = ServiceLocator.register('config', registerConfigService(ServiceLocator));
  const logger = ServiceLocator.register('logger', registerLoggerService(ServiceLocator));
  const db = ServiceLocator.register('db', registerDatabaseService(ServiceLocator));
  
  // Register domain services
  ServiceLocator.register('siteService', registerSiteService(ServiceLocator));
  ServiceLocator.register('userService', registerUserService(ServiceLocator));
  
  return { config, logger, db };
}

async function startServer() {
  try {
    // Initialize services
    const { config, logger } = await initializeServices();
    
    // Create Express app
    const app = express();
    
    // ... configure app ...
    
    // Start server
    const PORT = config.port || 3000;
    app.listen(PORT, () => {
      logger.info(`Server running on port ${PORT}`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
}

startServer();
```

### Service Registration Functions

Each service should have a registration function that receives the service locator and returns the service instance:

```javascript
// services/database.js
import pg from 'pg';
const { Pool } = pg;

export default function registerDatabaseService(serviceLocator) {
  const config = serviceLocator.get('config');
  const logger = serviceLocator.get('logger');
  
  // Create a connection pool
  const pool = new Pool({
    host: config.db.host,
    port: config.db.port,
    database: config.db.name,
    user: config.db.user,
    password: config.db.password,
    max: config.db.poolSize || 20,
  });
  
  // Log connection events
  pool.on('connect', () => {
    logger.info('Connected to PostgreSQL database');
  });
  
  pool.on('error', (err) => {
    logger.error('Database connection error', { error: err });
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
        logger.warn('A client has been checked out for more than 5 seconds!', {
          lastQuery: client.lastQuery
        });
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

### Domain Service Example

Domain services use the registered infrastructure services:

```javascript
// services/site.js
export default function registerSiteService(serviceLocator) {
  const db = serviceLocator.getDatabase();
  const logger = serviceLocator.getLogger();
  
  return {
    async getAll() {
      logger.debug('Fetching all sites');
      const result = await db.query('SELECT * FROM sites ORDER BY name');
      return result.rows;
    },
    
    async getById(id) {
      logger.debug('Fetching site by ID', { id });
      const result = await db.query(
        'SELECT * FROM sites WHERE id = $1',
        [id]
      );
      return result.rows[0] || null;
    },
    
    async create(site) {
      logger.debug('Creating new site', { site });
      const result = await db.query(
        'INSERT INTO sites (name, url, created_at) VALUES ($1, $2, NOW()) RETURNING *',
        [site.name, site.url]
      );
      return result.rows[0];
    },
    
    async update(id, site) {
      logger.debug('Updating site', { id, site });
      const result = await db.query(
        'UPDATE sites SET name = $1, url = $2, updated_at = NOW() WHERE id = $3 RETURNING *',
        [site.name, site.url, id]
      );
      return result.rows[0] || null;
    },
    
    async delete(id) {
      logger.debug('Deleting site', { id });
      const result = await db.query(
        'DELETE FROM sites WHERE id = $1 RETURNING id',
        [id]
      );
      return result.rowCount > 0;
    }
  };
}
```

### Using Services in Controllers

Access services from controllers through the service locator:

```javascript
// controllers/site.js
import { ServiceLocator } from '../serviceLocator.js';
import { NotFoundError, ValidationError } from '../errors/index.js';

export async function getAll(req, res, next) {
  try {
    const siteService = ServiceLocator.get('siteService');
    const sites = await siteService.getAll();
    res.json(sites);
  } catch (error) {
    next(error);
  }
}

export async function getById(req, res, next) {
  try {
    const id = parseInt(req.params.id, 10);
    
    if (isNaN(id)) {
      return next(new ValidationError('Invalid site ID'));
    }
    
    const siteService = ServiceLocator.get('siteService');
    const site = await siteService.getById(id);
    
    if (!site) {
      return next(new NotFoundError('Site'));
    }
    
    res.json(site);
  } catch (error) {
    next(error);
  }
}
```