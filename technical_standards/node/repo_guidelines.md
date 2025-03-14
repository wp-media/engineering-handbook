---
notion_page: https://www.notion.so/wpmedia/Node-js-Environment-Setup-1b6ed22a22f0801cb496dabc636da13a?pvs=4
title: Node.js - Environment Setup
---

# Node Environment Setup

## Formatting

We use Prettier for code formatting but follow specific practices to maximize its benefits.

### Prettier Philosophy

The fundamental principle of using a formatter is that **you should not need to care about formatting**. Formatters eliminate debates about code style by applying consistent rules automatically.

### Best Practices

1. **Don't configure Prettier at the repository level**

   Avoid creating a `.prettierrc` file with custom configurations. Use Prettier's defaults whenever possible.

   ```diff
   - // .prettierrc
   - {
   -   "singleQuote": true,
   -   "trailingComma": "all",
   -   "endOfLine": "auto"
   - }
   ```

   **Why?**
   - Single vs. double quotes have no functional difference in JavaScript
   - `trailingComma: all` is already Prettier's default
   - Line endings should be standardized via EditorConfig, not in Prettier

2. **Use editor integrations, not linting integration**

   Run Prettier through editor plugins that format on save, not through ESLint.

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

3. **Avoid mixing formatting configurations**

   Don't sneak formatting preferences into editor settings:

   ```diff
   - // .vscode/settings-template.json
   - {
   -   "editor.formatOnSave": true,
   -   "prettier.singleQuote": true
   - }
   ```

   This creates inconsistency between editor formatting and CI checks.

## Project Structure

Our project structure follows Node.js best practices to maximize clarity and minimize complexity.

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

2. **Keep `.gitignore` minimal and relevant**

   ```
   .vscode/settings.json
   /node_modules
   .env
   ```

   Extend only as needed for specific project requirements.

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

### Environment Configuration

For environment variables:

1. Use Node.js built-in `.env` support (v20.6.0+) with the `--env-file` flag when needed
2. Remove `.env` files until actually required
3. Consider Docker/docker-compose for development environment configuration

When you do need an `.env` file, launch the application with:

```bash
node --env-file=.env app.js
```

## Getting Started

For new team members joining a project:

1. Clone the repository
2. Copy `.vscode/settings-template.json` to `.vscode/settings.json`
3. Install recommended VSCode extensions
4. Install dependencies with `npm install`
5. Start the development server

This standardized setup ensures all team members can quickly begin working with consistent formatting and linting.