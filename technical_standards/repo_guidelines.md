---
notion_page: https://www.notion.so/wpmedia/Repository-Guidelines-1c3ed22a22f08022a3e3fa9161610bc4?pvs=4
title: Repository Guidelines
---

# Repository Guidelines

While the structure and content of codebases and repositories vary a lot between projects, some fundamental elements should be present to facilitate onboarding on the project. All our repositories should follow those recommendations.

## Getting Started

When starting to work on an unknown project, it is key that developers can easily get their environment set up in a similar way than other developers, and having key elements running locally such as the app itself, linters, tests, formatting, etc.
The recommended approach is to include a "Getting Started" section in the `readme.md` file of the repository. This section should list the following guidelines:
- How to install/run the app locally, including basic commands or URLs to validate the setup.
- How to configure the IDE for this project.
- How to configure & run the linter / formatter / tests.

1. Clone the repository
2. Copy `.vscode/settings-template.json` to `.vscode/settings.json`
3. Install recommended VSCode extensions
4. Install dependencies with `npm install`
5. Start the development server

This standardized setup ensures all team members can quickly begin working with consistent formatting, linting & testing.



## `.gitignore` file

`.gitignore` files tend to grow quickly as the project evolves, and developers runs some manual tests locally, etc. Also, reusing files from another project to get started faster contributes to overgrown `.gitignore` files. 
   
This can lead to unforeseen issues down the road with incomplete commits & pushes.

Extend only as needed for specific project requirements.