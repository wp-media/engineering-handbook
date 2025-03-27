---
notion_page: https://www.notion.so/wpmedia/Node-js-Input-Validation-1b6ed22a22f080fda004fbb6ec9a314d?pvs=4
title: Node.js - Input Validation
---

# Input Validation

Input validation are essential for building reliable, secure Node.js applications. This guide outlines our approach to implementing effective validation in Node.js applications.

Effective input validation is crucial for security and data integrity. We prefer simple, direct validation using JavaScript rather than validation libraries to maximize control and clarity on the performed operations.

## Basic Validation Approach

Use standard JavaScript for simple validations. If a more complex validation is required, you can use helpers. See `isValidHttpURL` in the example below.

```javascript
import express from "express";
export function isValidHttpURL(maybeUrl) {
  try {
    const url = new URL(maybeUrl);
    return /^https?:$/.test(url.protocol);
  } catch (e) {
    if (e.code === "ERR_INVALID_URL") {
      return false;
    }
    throw e;
  }
}
```

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
    });
    res.status(201).json({ job_id });
  });

  return app;
}
```

### Validation Middleware in Express.js

For routes with similar validation requirements, you can create validation middleware that can be reused:

```javascript
// middleware/authenticateMasterKey.js
import httpErrors from "http-errors";

export function authenticateMasterKey(serviceLocator, is_combined = false) {
  const { config } = serviceLocator;
  const { masterAPIKey } = config;

  if (!masterAPIKey) {
    throw new httpErrors.InternalServerError(
      "Master API Key is not configured",
    );
  }

  return function (req, res, next) {
    const token = req.headers.authorization;

    if (token !== masterAPIKey) {
      if (is_combined) {
        return next();
      }
      throw new httpErrors.Unauthorized("Invalid api key");
    }

    req.masterAPIKeyValid = true;
    next();
  };
}

```

Using validation middleware in routes, when creating the app:

```javascript
  app.use(
    "/api/v1/scan",
    authenticateMasterKey(serviceLocator),
    scanQueuesHandler(serviceLocator),
  );
```
