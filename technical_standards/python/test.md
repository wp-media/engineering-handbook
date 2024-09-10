---
notion_page: https://www.notion.so/wpmedia/Python-Tests-b6d5253be84142ee87a2fd120c192ca9?pvs=4
title: Python - Tests
---

# Python - Tests

For built-in unit and integration tests of our Python projects, we use [`pytest`](https://docs.pytest.org/en/stable/).
Running `pytest` and ensuring all tests are passing must be part of the CI of all repositories.

## Setting up pytest.

To start using `pytest`, some packages must be added to `requirements/ci.txt`: `pytest` and `pytest-mock`. Note that`pytest-django` must also be added for Django projects. To do so, add the following lines to `requirements/ci.txt` and replace the targeted version with the latest one available.

```
pytest==<latest_version>
pytest-django==<latest_version>
pytest-mock==<latest_version>
```

Make sure that in `requirements/dev.txt` the first line is 

```
-r ci.txt
```

For the new packages to be added to your docker image, build the image again with

```
docker-compose build
```

In your `pyproject.toml` file, add the following lines.

```toml
[tool.pytest.ini_options]
minversion = "6.0"
DJANGO_SETTINGS_MODULE = "repository_name.settings.ci"
addopts = "-x --strict-markers --ds=repository_name.settings.ci"
python_files = ["test*.py", "test_*.py"]

```

Creates a `tests.py` file and add the following code.

```python

def test_dummy(client):
    result = 1+1
    assert result == 2
```

Now enter the container and run the tests.

```bash
docker-compose exec app bash
pytest
```

You should see one passing test.

## Reporting test coverage

`pytest` can report test coverage after running. This is useful to understand which lines of code are covered with tests. We use this feature as part of our [automated CI](../../ways_of_working/processes/reviews.md).

To generate an XML coverage report, use the following options when running `pytest`:

```
pytest --cov=. --cov-report=xml
```
