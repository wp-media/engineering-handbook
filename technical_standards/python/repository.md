---
notion_page: https://www.notion.so/wpmedia/Python-Repository-preparation-3e292f3b1ffc4ee3adfdf1aceefa3fe2?pvs=4
title: Python - Repository preparation
---

# Python - Repository preparation

## Requirements

Dependencies for a codebase usually differ based on the environment the code is running: linters and test tools are required for development and CI, but not for production for instance. Therefore, we recommend using separate requirement files to ease dependency management and keep the footprint of the app small.

In the app repository, we use a `requirements/` directory, containing the following files:
- `base.txt`: this file lists the dependencies needed in all contexts;
- `ci.txt`: this file lists the dependencies needed in a CI context, on top of `base.txt`. It usually begins with `-r base.txt`.
- `dev.txt`: this file lists the dependencies needed in a development environment. It usually begins with `-r ci.txt`.
- `prod.txt`: this file lists the dependencies only needed in production and deployed environments. It usually begins with `-r base.txt`.

Note that those files are related to each others as some requirements include other ones. To achieve this, we use the following syntax as first line of a file, so that it includes all dependencies from the referenced file:

```
`-r base.txt`
```

To automatically generate those files, you can use:

```
mkdir requirements
touch requirements/{base.txt,ci.txt,dev.txt,prod.txt}
```

## Configuration file

As we use various tools to enhance the developer experience, it is easier to gather all configuration within a single `.toml` file.
In the project’s root directory, create a `pyproject.toml` file. This will include all the configuration for third party packages.

```bash
touch pyproject.toml
```

## Setting up a local virtual environment

As developers often switch from one repository to another, and that the environment usually differ (different dependencies, versions, etc.), it is safer to use a virtual environment per project. This will allow you to install dependencies for this repository only, and maybe use different ones on another repository without interferences.
To create and run a virtual environment, you can use the following commands in the repository:
```bash
# create the virtual env
python3 -m venv .venv
# activate the virtualenv
. .venv/bin/activate
```

## Setting up a Docker container