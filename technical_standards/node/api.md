---
notion_page: https://www.notion.so/wpmedia/Node-js-API-1b6ed22a22f080428b85e95ba12a154c?pvs=4
title: Node.js - API
---

# API Structure & Routing

Designing a clear, consistent API structure is essential for building maintainable Express.js applications. This guide outlines our approach to organizing routes, controllers, and API versioning.

## Router Organization

We prioritize clarity and simplicity in our routing structure, making it easy to understand the overall API at a glance.

### Direct Controller Wiring

Wire controllers directly in the main application file for better visibility:

```javascript
// app.js
import express from 'express';
import * as sitesController from './api/controllers/site.js';
import * as usersController from './api/controllers/user.js';

const app = express();

// Setup middleware
app.use(express.json());

// Setup routes
app.use('/api/v1/sites', express.Router()
  .get('/', sitesController.getAll)
  .get('/:id', sitesController.get)
  .post('/', sitesController.create)
  .put('/:id', sitesController.update)
  .delete('/:id', sitesController.remove)
);

app.use('/api/v1/users', express.Router()
  .get('/', usersController.getAll)
  .get('/:id', usersController.get)
);

export default app;
```

**Why?** This approach:
- Provides a clear overview of all available endpoints
- Makes the API structure immediately visible in one file
- Reduces unnecessary abstraction layers
- Simplifies debugging and tracing request handling

### Nested Routers for Route Grouping

For more complex APIs, use Express's nested router capability to group related routes:

```javascript
// app.js
import express from 'express';
import * as sitesController from './api/controllers/site.js';
import * as reportController from './api/controllers/report.js';

const app = express();
app.use(express.json());

// Main sites router
const sitesRouter = express.Router();

// Sites endpoints
sitesRouter.get('/', sitesController.getAll);
sitesRouter.get('/:id', sitesController.get);
sitesRouter.put('/:id', sitesController.update);

// Nested routes for site reports
sitesRouter.use('/:siteId/reports', express.Router()
  .get('/', reportController.getAllForSite)
  .get('/:reportId', reportController.getForSite)
  .post('/', reportController.createForSite)
);

// Register the main router
app.use('/api/v1/sites', sitesRouter);

export default app;
```

This pattern allows for logical grouping while maintaining a clear structure.

### Controller Organization

Keep controllers focused on request handling, delegating business logic to service modules:

```javascript
// api/controllers/site.js
import { ServiceLocator } from '../../serviceLocator.js';

export async function getAll(req, res, next) {
  try {
    const siteService = ServiceLocator.getSiteService();
    const sites = await siteService.getAll();
    res.json(sites);
  } catch (error) {
    next(error);
  }
}

export async function get(req, res, next) {
  try {
    const siteService = ServiceLocator.getSiteService();
    const site = await siteService.getById(req.params.id);
    
    if (!site) {
      const error = new Error('Site not found');
      error.statusCode = 404;
      return next(error);
    }
    
    res.json(site);
  } catch (error) {
    next(error);
  }
}
```

## API Versioning

API versioning is essential for maintaining backward compatibility while allowing evolution. Our approach focuses on routing-level versioning without tying it to the file structure.

### Versioning Through Routes, Not Files

Avoid organizing files by API version:

```diff
- /api
-   /handlers
-     /v1
-       site.js
-       user.js
-     /v2
-       site.js
-       user.js
+ /api
+   /controllers
+     site.js
+     user.js
```

**Why?** This approach:
- Prevents code duplication
- Makes it easier to maintain shared logic between versions
- Simplifies refactoring when introducing new versions

### Implementing Versioned Routes

Handle versioning at the routing level:

```javascript
// app.js
import express from 'express';
import * as siteController from './api/controllers/site.js';

const app = express();
app.use(express.json());

// V1 API routes
app.use('/api/v1/sites', express.Router()
  .get('/', siteController.getAllV1)
  .get('/:id', siteController.getV1)
);

// V2 API routes (when needed)
app.use('/api/v2/sites', express.Router()
  .get('/', siteController.getAllV2)
  .get('/:id', siteController.getV2)
);

export default app;
```

For controllers that need to handle multiple versions:

```javascript
// api/controllers/site.js

// V1 endpoints
export async function getAllV1(req, res, next) {
  try {
    const sites = await siteService.getAll();
    // V1 response format
    res.json(sites);
  } catch (error) {
    next(error);
  }
}

// V2 endpoints
export async function getAllV2(req, res, next) {
  try {
    const sites = await siteService.getAll();
    // V2 response format with additional fields
    const enhancedSites = sites.map(site => ({
      ...site,
      metrics: site.metrics || {},
      _links: {
        self: `/api/v2/sites/${site.id}`,
        reports: `/api/v2/sites/${site.id}/reports`
      }
    }));
    res.json(enhancedSites);
  } catch (error) {
    next(error);
  }
}
```

### Handling Version Compatibility

In many cases, version differences involve only pre-processing inputs or post-processing outputs:

```javascript
// api/controllers/site.js
import { transformResponseV1, transformResponseV2 } from '../transforms/site.js';

// Shared implementation
async function getSiteById(id) {
  const siteService = ServiceLocator.getSiteService();
  return await siteService.getById(id);
}

// V1 endpoint
export async function getV1(req, res, next) {
  try {
    const site = await getSiteById(req.params.id);
    if (!site) {
      const error = new Error('Site not found');
      error.statusCode = 404;
      return next(error);
    }
    res.json(transformResponseV1(site));
  } catch (error) {
    next(error);
  }
}

// V2 endpoint
export async function getV2(req, res, next) {
  try {
    const site = await getSiteById(req.params.id);
    if (!site) {
      const error = new Error('Site not found');
      error.statusCode = 404;
      return next(error);
    }
    res.json(transformResponseV2(site));
  } catch (error) {
    next(error);
  }
}
```

This pattern allows sharing core logic while adapting inputs and outputs for each API version.

## Route Parameter Validation

Validate route parameters early in your request handling:

```javascript
export async function get(req, res, next) {
  const id = req.params.id;
  
  // Validate ID format
  if (!id || !/^\d+$/.test(id)) {
    const error = new Error('Invalid site ID format');
    error.statusCode = 400;
    return next(error);
  }
  
  try {
    // Continue with request handling
  } catch (error) {
    next(error);
  }
}
```

## Best Practices Summary

- Wire controllers directly in the main app file for better visibility
- Use nested Express routers for logical grouping of related routes
- Implement API versioning at the routing level, not in the file structure
- Share implementation logic between API versions when possible
- Keep controllers focused on request handling, delegating business logic to services
- Validate route parameters early in the request handling flow
- Always use the Express error handling pattern (passing errors to `next()`)

Following these practices will result in a clear, maintainable API structure that's easy to understand and extend.