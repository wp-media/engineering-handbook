---
notion_page: https://www.notion.so/wpmedia/Node-js-API-1b6ed22a22f080428b85e95ba12a154c?pvs=4
title: Node.js - API
---

# API Structure & Routing

Designing a clear, consistent API structure is essential for building maintainable applications. This guide outlines our recommended approach to organizing routes, controllers, and API versioning.

## Router Organization

We prioritize clarity and simplicity in our routing structure, making it easy to understand the overall API at a glance.

### Direct Controller Wiring

Wire controllers directly in the main application file (or in the app factory) for better visibility:

```javascript
import express from "express";
import { errorHandler } from "./errorHandler.js";
import path from "node:path";
import { authenticateMasterKey } from "./middlewares/authenticateMasterKey.js";
import scanQueuesHandler from "./handlers/scanQueues.js";

const PUBLIC_PATH = path.resolve(import.meta.dirname, "../public");

export default function appFactory(serviceLocator) {
  const db = serviceLocator.db;

  const app = express();
  app.use(express.json());
  app.use(express.static(PUBLIC_PATH));

  app.use(
    "/api/v1/scan",
    authenticateMasterKey(serviceLocator),
    scanQueuesHandler(serviceLocator),
  );

  app.use(errorHandler());

  return app;
}


```

**Why?** This approach:
- Provides a clear overview of all available endpoints
- Makes the API structure immediately visible in one file
- Reduces unnecessary abstraction layers
- Simplifies debugging and tracing request handling

### Handler/Controller Organization

Keep controllers (also called handlers) focused on request handling, delegating business logic to service modules. In the following example, the controller (or handler) performs request validation and response management. The actual business logic is implemented within the scanQueue service.

```javascript
import express from "express";
import httpErrors from "http-errors";
import { isValidHttpURL } from "../utils/isValidHttpURL.js";

export default function scanQueuesHandler(serviceLocator) {
  const app = express.Router();
  const dal = serviceLocator.dal;

  app.post("/", async (req, res) => {
    if (!req.body || typeof req.body !== "object") {
      throw new httpErrors.BadRequest("Invalid request body");
    }
    const { url } = req.body;
    if (!url) {
      throw new httpErrors.BadRequest("URL is required");
    }
    if (typeof url !== "string") {
      throw new httpErrors.BadRequest("URL must be a string");
    }
    if (!isValidHttpURL(url)) {
      throw new httpErrors.BadRequest("Invalid URL format");
    }
    const { job_id } = await dal.scanQueue.createScan({
      url,
      type: "single",
    });
    res.status(201).json({ job_id });
  });

  return app;
}

```

## API Versioning

API versioning is essential for maintaining backward compatibility while allowing evolution. Our recommended approach focuses on routing-level versioning without tying it to the file structure.

Let's say we have endpoints for `/api/v1/site` and `/api/v2/site`. Chances are there is quite some shared logic between those two endpoints, despite the lack of backward compatibility. To facilitate reusing logic and still having a good overview of the endpoint's behavior at a glance, we recommend the following file structure:

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

## Route Parameter Validation

Validate route parameters early in your request handling. This should be part of the handler/controller, to ensure that the inputs sent to the services and the business logic are as expected. In case a validation fails, a proper error message must be returned along with a Bad Request (400) error code including the missing parameter and/or its expected type/format.

```javascript
  app.post("/", async (req, res) => {
    if (!req.body || typeof req.body !== "object") {
      throw new httpErrors.BadRequest("Invalid request body");
    }
    const { url } = req.body;
    if (!url) {
      throw new httpErrors.BadRequest("URL is required");
    }
    if (typeof url !== "string") {
      throw new httpErrors.BadRequest("URL must be a string");
    }
    if (!isValidHttpURL(url)) {
      throw new httpErrors.BadRequest("Invalid URL format");
    }
    ...
}
```
