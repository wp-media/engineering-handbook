---
notion_page: https://www.notion.so/wpmedia/Node-js-Environment-Setup-1b6ed22a22f0801cb496dabc636da13a?pvs=4
title: Node.js - Environment Setup
---

# Node Environment Setup

## Formatting

We use Prettier for code formatting but recommend specific practices to maximize its benefits. This guide provides our recommended standards, but each repository can have an adjusted configuration to meet its needs.

### Prettier Philosophy

Formatters eliminate debates about code style by applying consistent rules automatically.

1. **Rely on default configurations**

   The fundamental principle of using a formatter is that you should not need to care about formatting. Therefore, use Prettier's defaults whenever possible to reduce the complexity of the project.

2. **Use editor integrations over linting integration**

   Run Prettier through editor plugins that format on save, not through ESLint. Therefore, avoid:

   ```diff
   - // .eslintrc.js
   - {
   -   "extends": ["prettier"],
   -   "plugins": ["prettier"],
   -   "rules": {
   -     "prettier/prettier": "error"
   -   }
   - }
   ```

   **Why?** Separating formatting from linting:
   - Makes the linting process faster
   - Reduces configuration complexity
   - Aligns with each tool's purpose: linters for code quality, formatters for style

3. **When configuration is needed, manage it within prettier directly**

   If you really need specific Prettier configuration, use the `.prettierrc` file directly.
   Don't sneak formatting preferences into editor settings:

   ```diff
   - // .vscode/settings-template.json
   - {
   -   "editor.formatOnSave": true,
   -   "prettier.singleQuote": true
   - }
   ```

   **Why?** This creates inconsistency between editor formatting and CI checks.

## Project Structure

Our recommended project structure aims at increasing clarity and minimizing complexity.

### Directory Organization

1. **Avoid unnecessary `src` directory**

   ```diff
   - /src
   -   /api
   -   /db
   -   app.js
   + /api
   + /db
   + app.js
   ```

   **Why?** The `src` directory typically implies a compilation step, which we don't have in our Node.js applications. We run the code directly without transpilation.

### Module System

Use ECMAScript Modules (ESM) consistently throughout the project:

```javascript
// Do this (ESM syntax)
import express from 'express';
import { router } from './router.js';

export function someFunction() {
  // ...
}

// Not this (CommonJS syntax)
const express = require('express');
const router = require('./router');

module.exports.someFunction = function() {
  // ...
};
```

**Why?** ESM is the standard JavaScript module system and offers benefits like:
- Static analysis
- Tree-shaking
- Top-level await
- Better compatibility with modern JavaScript features

