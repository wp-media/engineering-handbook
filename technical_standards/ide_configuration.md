---
notion_page: https://www.notion.so/wpmedia/IDE-Configuration-1b6ed22a22f080c6b003c1a4313c6a47?pvs=4
title: IDE Configuration
---

# Development Environment Setup

Setting up a consistent development environment is crucial for team productivity and code quality. This guide outlines our best practices for editor configuration, formatting, and project structure. Our teams and projects come from different backgrounds, have different histories and use various languages. Therefore, we don't aim for a one-size-fits-all approach. Instead, we provide guidelines on how to setup each repository to make it easy for any developers to start working on it with the proper environment setup.

## Editor Configuration

A consistent IDE configuration for all developers working on a repository is key to align on code styling and facilitate collaborations and reviews. By aligning on formatting, we ensure that basic format rules of the linters are automatically applied, and that the output of `git diff` is not polluted by irrelevant changes such as identation.

### EditorConfig File

To standardize basic formatting across all editors, include a `.editorconfig` file in the project root. This ensures consistent line endings, indentation, and other basic formatting regardless of editor choice. The following example is enough to start with:

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

Each repository can then have its own `.editorconfig` file: it will be automatically applied by the IDE when working on this project. Teams are autonomous on adjusting this file to the needs of their projects.

### IDE configurations

It is recommended to provide a configuration files for the editors the team is using. This can help developers to align on the tools/plugins they use for a specific project and facilitate some operations such as formatting when saving, etc.

#### VSCode Configuration

Here is the recommended approach for Visual Studio Code:

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