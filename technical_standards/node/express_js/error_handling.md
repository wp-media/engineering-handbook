---
notion_page: https://www.notion.so/wpmedia/Node-js-Error-Handling-1b6ed22a22f080169d7add749f3e9f08?pvs=4
title: Node.js - Error handling in Express.js
---

# Error Handling in Express.js

Robust error handling is essential for building reliable, secure Node.js applications. This guide outlines our basic approach to centralizing error handling in our Node.js applications.

The error handler must have the following features:
- All errors must be reported through logs.
- Details of errors from the application should not be disclosed to the users, unless the error is listed as "not sensitive" or "needed".

The following example demonstrates the implementation of a basic error handler that logs all errors and prevents sending details about unexpected errors to the users.

```javascript
// errorHandler.js
import httpErrors from "http-errors";

function isBlessedError(err) {
  return err instanceof httpErrors.HttpError;
}

export function errorHandler() {
  return (err, req, res, next) => {
    if (res.headersSent) {
      return next(err);
    }

    console.error(err); // Log error for debugging

    if (!isBlessedError(err)) {
      const presentableError = new httpErrors.InternalServerError(
        "Internal Server Error",
        {
          cause: err,
        },
      );
      err = presentableError;
    }

    res.status(err.status || 500).json({ message: err.message });
  };
}
```

### Registering the Error Handler

Add the error handler as the last middleware in your Express app:

```javascript
// app.js
import express from 'express';
import { errorHandler } from './errorHandler.js';
import { router } from './router.js';

const app = express();

// Regular middleware
app.use(express.json());

// Routes
app.use('/api', router);

// Error handler (must be last)
app.use(errorHandler);

export default app;
```

### Delegating Errors to the Handler

In route handlers, pass errors to Express's `next()` function instead of handling them directly:

```javascript
// controllers/site.js
export async function getSite(req, res, next) {
  try {
    const siteId = req.params.id;
    
    // Validate ID
    if (!siteId || !/^\d+$/.test(siteId)) {
      const error = new Error('Invalid site ID');
      error.statusCode = 400;
      return next(error);
    }
    
    const site = await siteService.getById(siteId);
    
    if (!site) {
      const error = new Error('Site not found');
      error.statusCode = 404;
      return next(error);
    }
    
    res.json(site);
  } catch (error) {
    // Pass unexpected errors to the error handler
    next(error);
  }
}
```