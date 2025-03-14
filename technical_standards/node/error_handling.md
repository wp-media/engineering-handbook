---
notion_page: https://www.notion.so/wpmedia/Node-js-Error-Handling-1b6ed22a22f080169d7add749f3e9f08?pvs=4
title: Node.js - Error handling
---

# Error Handling

Robust error handling is essential for building reliable, secure Node.js applications. This guide outlines our approach to centralizing error handling in Express.js applications.

## Centralized Error Handling

We use Express.js's built-in error handling middleware to centralize error processing, providing consistent error responses across the application.

### Error Handler Middleware

Create a centralized error handler by adding a middleware with four parameters at the end of your middleware chain:

```javascript
// errorHandler.js
export function errorHandler(err, req, res, next) {
  // If headers have already been sent, delegate to Express's default error handler
  if (res.headersSent) {
    return next(err);
  }

  // Determine status code (default to 500 if not specified)
  const statusCode = err.statusCode || 500;
  
  // Log the error (with different levels based on status code)
  if (statusCode >= 500) {
    console.error('Server error:', err);
  } else {
    console.warn('Client error:', err.message);
  }
  
  // Prepare the error response
  const errorResponse = {
    error: {
      message: err.message || 'Internal Server Error',
      status: statusCode,
    }
  };
  
  // Include stack trace in development but not in production
  if (process.env.NODE_ENV !== 'production' && err.stack) {
    errorResponse.error.stack = err.stack;
  }
  
  // Send the error response
  res.status(statusCode).json(errorResponse);
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

### Custom Error Classes

Create custom error classes to standardize error handling:

```javascript
// errors/index.js
export class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404);
  }
}

export class ValidationError extends AppError {
  constructor(message = 'Validation failed') {
    super(message, 400);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(message, 403);
  }
}
```

Using custom errors in controllers:

```javascript
// controllers/site.js
import { NotFoundError, ValidationError } from '../errors/index.js';

export async function getSite(req, res, next) {
  try {
    const siteId = req.params.id;
    
    // Validate ID
    if (!siteId || !/^\d+$/.test(siteId)) {
      return next(new ValidationError('Invalid site ID'));
    }
    
    const site = await siteService.getById(siteId);
    
    if (!site) {
      return next(new NotFoundError('Site'));
    }
    
    res.json(site);
  } catch (error) {
    next(error);
  }
}
```

### Advanced Error Handler Features

Extend the error handler with additional capabilities:

```javascript
// errorHandler.js
export function errorHandler(err, req, res, next) {
  if (res.headersSent) {
    return next(err);
  }

  const statusCode = err.statusCode || 500;
  
  // 1. Log the error
  logError(err, req);
  
  // 2. Report to error tracking service (e.g., Sentry)
  reportToErrorService(err, req);
  
  // 3. Report metrics (e.g., Prometheus)
  incrementErrorMetric(statusCode);
  
  // 4. Prepare the response
  const errorResponse = {
    error: {
      message: getErrorMessage(err, statusCode),
      status: statusCode,
      code: err.code || 'UNKNOWN_ERROR',
      // Include request ID for tracking
      requestId: req.id || generateRequestId(),
    }
  };
  
  // Include stack trace in development
  if (process.env.NODE_ENV !== 'production' && err.stack) {
    errorResponse.error.stack = err.stack;
  }
  
  // 5. Send the response
  res.status(statusCode).json(errorResponse);
}

// Helper functions
function logError(err, req) {
  const logData = {
    url: req.originalUrl,
    method: req.method,
    statusCode: err.statusCode || 500,
    message: err.message,
    stack: err.stack,
    user: req.user ? req.user.id : 'anonymous',
  };
  
  if (logData.statusCode >= 500) {
    console.error('Server error:', logData);
  } else {
    console.warn('Client error:', logData);
  }
}

function reportToErrorService(err, req) {
  // Integration with error reporting service would go here
  // Example with Sentry:
  // Sentry.captureException(err);
}

function incrementErrorMetric(statusCode) {
  // Increment error metrics for monitoring
  // Example with Prometheus:
  // httpErrorCounter.inc({ status_code: statusCode });
}

function getErrorMessage(err, statusCode) {
  // In production, hide internal server error details
  if (statusCode >= 500 && process.env.NODE_ENV === 'production') {
    return 'Internal Server Error';
  }
  
  return err.message || 'Unknown Error';
}

function generateRequestId() {
  return `req-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}
```
