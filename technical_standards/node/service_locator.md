---
notion_page: https://www.notion.so/wpmedia/Node-js-Service-Locator-Pattern-1b6ed22a22f080de90ecc6c627805dbc?pvs=4
title: Node.js - Service Locator Pattern
---

A well-structured application architecture is essential for building maintainable, scalable Node.js applications. This guide outlines our approach to organizing dependencies and implementing the Service Locator pattern in our Node.js applications.

## Service Locator Pattern

The Service Locator pattern provides a centralized registry of services, making dependencies available throughout the application while maintaining loose coupling between components.

### Core Implementation

Create a service locator module to manage application-wide dependencies. Register the services when creating the service locator.

```javascript
// service-locator.js
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

This service locator can then be used to create the app:

```javascript
import express from "express";
import { errorHandler } from "./errorHandler.js";
import scanQueuesHandler from "./handlers/scanQueues.js";

export default function appFactory(serviceLocator) {
  const db = serviceLocator.db;

  const app = express();
  app.use(express.json());

  app.use(
    "/api/v1/scan",
    scanQueuesHandler(serviceLocator),
  );

  app.use(errorHandler());

  return app;
}
```

### Using Services in Controllers

Access services from controllers and handlers through the service locator. In the following example, the DAL service is accessed through the service locator for the handler to call the corresponding business logic.

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