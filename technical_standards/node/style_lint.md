---
notion_page: https://www.notion.so/wpmedia/Node-js-Code-Style-Linting-1b6ed22a22f080b581c2f37968fc8665?pvs=4
title: Node.js - Code Linting
---

# Code Linting

Maintaining high code quality is essential for building reliable, maintainable Node.js applications. This guide outlines our approach to linting and code quality tools.
Our recommended tool for this is ESLint.

## ESLint Configuration

### Modern ESLint Setup (v9+)

ESLint v9 introduced "flat config," a simpler and more powerful configuration system. We recommend this approach.
1. **Create a flat config file**

   Create `eslint.config.js` in your project root:

   ```javascript
   import js from "@eslint/js";
   import importPlugin from "eslint-plugin-import";
   import globals from "globals";

   export default [
     js.configs.recommended,
     importPlugin.flatConfigs.recommended,
     {
       languageOptions: {
         ecmaVersion: "latest",
         globals: {
           ...globals.nodeBuiltin,
         },
       },
     },
     {
       ignores: [],
     },
   ];
   ```

2. **Install required dependencies**

   ```bash
   npm install --save-dev eslint eslint-plugin-import globals
   ```

3. **Remove legacy configuration**

   If migrating from older ESLint versions, remove:
   - `.eslintrc.js`
   - `.eslintrc.json`
   - `.eslintrc.yml`
   - Any ESLint config in `package.json`

### Separating Linting from Formatting

A key principle in our setup is the separation of concerns between linting (code quality) and formatting (code style):

```diff
- // Don't use ESLint for formatting
- {
-   "extends": ["prettier"],
-   "plugins": ["prettier"],
-   "rules": {
-     "prettier/prettier": "error"
-   }
- }
```

**Why?**
- ESLint should focus on catching potential bugs and enforcing best practices
- Formatting should be handled by Prettier through editor integrations
- This separation makes both tools more efficient and reduces configuration complexity

### Running ESLint

Add scripts to your `package.json` for running ESLint:

```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix"
  }
}
```

Run checks with:

```bash
npm run lint
```

### Customizing Rules

While we generally rely on recommended configurations, you can add project-specific rules if needed:

```javascript
export default [
  js.configs.recommended,
  importPlugin.flatConfigs.recommended,
  {
    rules: {
      "no-console": ["warn", { allow: ["warn", "error"] }],
      "import/no-unresolved": "error",
    }
  }
];
```

Always document the reasoning behind custom rules, especially when disabling recommended ones.

## TypeScript Considerations

We take a pragmatic approach to TypeScript adoption, avoiding premature implementation.


### Using TypeScript Without Compilation

For new projects that need TypeScript, we prefer using Node.js's built-in TypeScript support rather than a separate compilation step:

```bash
# Run TypeScript directly with Node.js (v20+)
node --experimental-strip-type-annotations app.ts
```

From Node.js v21, you can use the `--strip-type-annotations` flag (non-experimental).

## Integrating with CI/CD

Ensure linting is part of your continuous integration pipeline:

```yaml
# Example GitHub Actions workflow
name: Lint

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
```

```yaml
# Example GitLab Actions workflow
stages:
  - test

default:
  tags:
    - k8s-small

###############################################################################
# Stage: test
lint:
  image: harbor.one.com/nodejs/node:22.14.0-noble
  stage: test
  script:
    - npm ci
    - npm run lint
```

This ensures all code is linted before being merged, maintaining code quality across the project.
