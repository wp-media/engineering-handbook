---
notion_page: https://www.notion.so/wpmedia/IDE-Configuration-1b6ed22a22f080c6b003c1a4313c6a47?pvs=4
title: IDE Configuration
---

# Development Environment Setup

Setting up a consistent development environment is crucial for team productivity and code quality. This guide outlines our best practices for editor configuration, formatting, and project structure in Node.js applications.

## Editor Configuration

We prioritize a consistent editing experience while allowing developers to use their preferred tools. Our approach focuses on standardizing critical aspects while providing flexibility.

### VSCode Configuration

For Visual Studio Code users (our recommended editor), we provide a standardized setup:

1. **Ignore personal settings**

   Add the following to `.gitignore` to allow developers to customize their workspace:

   ```
   .vscode/settings.json
   ```

2. **Recommend essential extensions**

   Create `.vscode/extensions.json` with our recommended extensions:

   ```json
   {
     // See http://go.microsoft.com/fwlink/?LinkId=827846 to learn about workspace recommendations.
     // List of extensions which should be recommended for users of this workspace.
     "recommendations": [
       "editorconfig.editorconfig",
       "esbenp.prettier-vscode",
       "dbaeumer.vscode-eslint"
     ]
   }
   ```

3. **Provide a settings template**

   Create `.vscode/settings-template.json` for developers to copy:

   ```json
   {
     "editor.formatOnSave": true,
     // Will make VSCode auto importing include the file extension, which is
     // necessary in node.js when using ESM modules.
     "javascript.preferences.importModuleSpecifierEnding": "js"
   }
   ```

   Developers should copy this file to `.vscode/settings.json` when setting up the project.

### EditorConfig

To standardize basic formatting across all editors, include an `.editorconfig` file in the project root:

```
root = true

[*]
end_of_line = lf
insert_final_newline = true
charset = utf-8
trim_trailing_whitespace = true
indent_size = 2
indent_style = space

[Makefile]
indent_style = tab
```

This ensures consistent line endings, indentation, and other basic formatting regardless of editor choice.
