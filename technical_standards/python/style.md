---
notion_page: https://www.notion.so/wpmedia/Python-Code-Style-81237a443b514d8dafb033efe7cdb4f1?pvs=4
title: Python - Code Style
---

# Python - Code Style

This document summarizes our standard code styling tools for Python codebases.

Our code style guidelines include linting and formatting. All the following should automatically be checked as part of the CI of all repositories.

## Linting

### pylint

For linting, we use `pylint`. 

#### Install and configure

To use it, the package must be added to `requirements/ci.txt`. Note that`pylint-django` must also be added for Django projects. To do so, add the following lines to `requirements/ci.txt` and replace the targeted version with the latest one available.

```
pylint==<latest_version>
pylint-django==<latest_version>
```

Add the following lines `pyproject.toml`. Note that some lines mentioning *django* are only needed for Django projects.

```toml
[tool.pylint.MAIN]
load-plugins = ["pylint_django"]
ignore = [
    "migrations",
    "tests",
    "test",
    "venv",
    ".venv",
    "setup.py",
    "manage.py",
]
django-settings-module = "repository_name.settings.ci"

```

#### Run

Now you can run `pylint` and resolve any linting issues you will encounter ; either in a virtual environment, in your docker image or directly in the CI:

```
pylint --recursive=y .
```

## Formatting

For code formatting, we use two tools: 
- [`black`](https://black.readthedocs.io/en/stable/): This tool is a code formatter. While strongly opinionated, its capacity to automatically format code and to promote small diff codebase changes makes it very useful to work on large projects.
- [`isort`](https://pycqa.github.io/isort/): This tool automatically sorts imports at the beginning of Python files, making dependency management easier and more readable.

### black

#### Install and configure

To use `black`, the package must be installed by adding the following line to `requirements/ci.txt`:

```
black==<latest_version>
```

Paste the following lines to `pyproject.toml` to configure the tool:

```toml
[tool.black]
exclude = '''
/(
    \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | venv
  | \.local
  | \.pytest_cache
  | _build
  | buck-out
  | build
  | dist
  | migrations
)/
'''
```

#### Run

Now you can run `black .` and resolve any linting issues you will encounter ; either in a virtual environment or in your docker image.

To use `black` in a CI, you don't want the format to be enforced but just to report not compliant lines so that you can fix them. To do so, you can use the check option: 

```
black --check .
```

### isort

#### Install and configure

To use `isort`, the package must be installed by adding the following line to `requirements/ci.txt`:

```
isort==<latest_version>
```

Paste the following lines to `pyproject.toml` to configure the tool:

```toml
[tool.isort]
py_version = 311
multi_line_output = 3
include_trailing_comma = true
skip_glob = ["venv/*", "**/migrations/**"]
profile = "black"
```

#### Run

Now you can run `isort .` and resolve any linting issues you will encounter ; either in a virtual environment or in your docker image.

To use `isort` in a CI, you don't want the format to be enforced but just to report not compliant lines so that you can fix them. To do so, you can use the check option: 

```
isort --check .
```