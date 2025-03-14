---
notion_page: https://www.notion.so/wpmedia/Node-js-Input-Validation-1b6ed22a22f080fda004fbb6ec9a314d?pvs=4
title: Node.js - Input Validation
---

# Input Validation

Input validation are essential for building reliable, secure Node.js applications. This guide outlines our approach to implementing effective validation in Express.js applications.

Effective input validation is crucial for security and data integrity. We prefer simple, direct validation using JavaScript rather than heavy validation libraries.

## Basic Validation Approach

Use standard JavaScript for simple validations:

```javascript
// controllers/site.js
export async function createSite(req, res, next) {
  try {
    // Validate required fields
    if (!req.body || typeof req.body !== 'object') {
      return next(new ValidationError('Request body is required'));
    }
    
    const { url, name } = req.body;
    
    // Validate URL
    if (!url || typeof url !== 'string') {
      return next(new ValidationError('URL is required and must be a string'));
    }
    
    // Validate URL format
    try {
      new URL(url); // Will throw if invalid
    } catch (error) {
      return next(new ValidationError('Invalid URL format'));
    }
    
    // Validate name
    if (!name || typeof name !== 'string') {
      return next(new ValidationError('Name is required and must be a string'));
    }
    
    if (name.length < 3 || name.length > 100) {
      return next(new ValidationError('Name must be between 3 and 100 characters'));
    }
    
    // Process the validated input
    const site = await siteService.create({ url, name });
    res.status(201).json(site);
  } catch (error) {
    next(error);
  }
}
```

## Validation Helpers

Create validation helper functions for common validation patterns:

```javascript
// utils/validation.js
export const validate = {
  required: (value, name) => {
    if (value === undefined || value === null) {
      throw new ValidationError(`${name} is required`);
    }
    return value;
  },
  
  string: (value, name, options = {}) => {
    validate.required(value, name);
    
    if (typeof value !== 'string') {
      throw new ValidationError(`${name} must be a string`);
    }
    
    const { min, max } = options;
    
    if (min !== undefined && value.length < min) {
      throw new ValidationError(`${name} must be at least ${min} characters`);
    }
    
    if (max !== undefined && value.length > max) {
      throw new ValidationError(`${name} must be at most ${max} characters`);
    }
    
    return value;
  },
  
  number: (value, name, options = {}) => {
    validate.required(value, name);
    
    const numValue = Number(value);
    
    if (isNaN(numValue)) {
      throw new ValidationError(`${name} must be a number`);
    }
    
    const { min, max } = options;
    
    if (min !== undefined && numValue < min) {
      throw new ValidationError(`${name} must be at least ${min}`);
    }
    
    if (max !== undefined && numValue > max) {
      throw new ValidationError(`${name} must be at most ${max}`);
    }
    
    return numValue;
  },
  
  url: (value, name) => {
    const strValue = validate.string(value, name);
    
    try {
      new URL(strValue);
    } catch (error) {
      throw new ValidationError(`${name} must be a valid URL`);
    }
    
    return strValue;
  },
  
  email: (value, name) => {
    const strValue = validate.string(value, name);
    
    // Basic email validation regex
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    
    if (!emailRegex.test(strValue)) {
      throw new ValidationError(`${name} must be a valid email address`);
    }
    
    return strValue;
  },
  
  object: (value, name) => {
    validate.required(value, name);
    
    if (typeof value !== 'object' || value === null || Array.isArray(value)) {
      throw new ValidationError(`${name} must be an object`);
    }
    
    return value;
  },
  
  array: (value, name, itemValidator) => {
    validate.required(value, name);
    
    if (!Array.isArray(value)) {
      throw new ValidationError(`${name} must be an array`);
    }
    
    if (itemValidator) {
      return value.map((item, index) => itemValidator(item, `${name}[${index}]`));
    }
    
    return value;
  }
};
```

Using the validation helpers:

```javascript
// controllers/site.js
import { validate } from '../utils/validation.js';
import { ValidationError } from '../errors/index.js';

export async function createSite(req, res, next) {
  try {
    // Validate request body
    const body = validate.object(req.body, 'Request body');
    
    // Extract and validate fields
    const url = validate.url(body.url, 'URL');
    const name = validate.string(body.name, 'Name', { min: 3, max: 100 });
    
    // Optional fields
    const description = body.description !== undefined
      ? validate.string(body.description, 'Description', { max: 500 })
      : undefined;
    
    // Process the validated input
    const site = await siteService.create({ url, name, description });
    res.status(201).json(site);
  } catch (error) {
    if (error instanceof ValidationError) {
      // Pass validation errors directly
      next(error);
    } else {
      // Handle unexpected errors
      next(error);
    }
  }
}
```

### Validation Middleware

For routes with similar validation requirements, create validation middleware:

```javascript
// middleware/validation.js
import { validate } from '../utils/validation.js';
import { ValidationError } from '../errors/index.js';

export const validateSite = (req, res, next) => {
  try {
    const body = validate.object(req.body, 'Request body');
    
    req.validatedData = {
      url: validate.url(body.url, 'URL'),
      name: validate.string(body.name, 'Name', { min: 3, max: 100 }),
    };
    
    if (body.description !== undefined) {
      req.validatedData.description = validate.string(
        body.description, 
        'Description', 
        { max: 500 }
      );
    }
    
    next();
  } catch (error) {
    next(error instanceof ValidationError ? error : new ValidationError(error.message));
  }
};

export const validateId = (paramName = 'id') => (req, res, next) => {
  try {
    const id = req.params[paramName];
    
    if (!id || !/^\d+$/.test(id)) {
      throw new ValidationError(`Invalid ${paramName}`);
    }
    
    req.validatedParams = {
      ...req.validatedParams,
      [paramName]: parseInt(id, 10),
    };
    
    next();
  } catch (error) {
    next(error instanceof ValidationError ? error : new ValidationError(error.message));
  }
};
```

Using validation middleware in routes:

```javascript
// router.js
import express from 'express';
import * as siteController from './controllers/site.js';
import { validateSite, validateId } from './middleware/validation.js';

const router = express.Router();

router.post('/sites', validateSite, siteController.createSite);
router.get('/sites/:id', validateId(), siteController.getSite);
router.put('/sites/:id', validateId(), validateSite, siteController.updateSite);

export { router };
```
