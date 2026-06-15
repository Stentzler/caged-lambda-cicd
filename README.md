# CAGED Lambda CI/CD

Shared GitHub Actions workflow for deploying CAGED Python Lambda functions as ZIP
artifacts.

This repository does not contain Lambda application code. It contains the
reusable CI/CD pipeline that the Lambda repositories call from their own
`.github/workflows/deploy.yml` files.

## What It Does

The reusable workflow lives at:

```text
.github/workflows/lambda-python-zip.yml
```

When a Lambda repository calls this workflow, it:

1. Checks out the Lambda repository.
2. Sets up Python and `uv`.
3. Installs locked dependencies with `uv sync --frozen`.
4. Runs the repository test and lint gates:
   - `uv run pytest`
   - `uv run ruff check .`
   - `uv run ruff format --check .`
5. Builds the Lambda ZIP by running `make package`.
6. Verifies that the expected ZIP file exists.
7. Assumes the AWS deployment role through GitHub OIDC.
8. Updates the Lambda function code.
9. Publishes a new Lambda version.
10. Moves the configured alias, such as `dev`, to that new version.

## How Lambda Repositories Use It

Each Lambda repository should define a small workflow that calls this reusable
workflow:

```yaml
name: Deploy Lambda

on:
  push:
    branches:
      - develop
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  deploy-dev:
    uses: Stentzler/caged-lambda-cicd/.github/workflows/lambda-python-zip.yml@main
    with:
      environment_name: dev
      function_name: caged-dev-example
      alias_name: dev
      zip_path: dist/example-lambda.zip
    secrets: inherit
```

The caller repository provides the Lambda-specific values. This shared
repository provides the deployment behavior.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `function_name` | Yes | - | Name of the AWS Lambda function to update. |
| `zip_path` | Yes | - | Path to the ZIP artifact created by `make package`. |
| `alias_name` | No | `dev` | Lambda alias to promote after publishing the new version. |
| `environment_name` | No | `dev` | GitHub environment used by the deploy job. |
| `aws_region` | No | `us-east-1` | AWS region where the Lambda function exists. |
| `python_version` | No | `3.14` | Python version installed in the CI runner. |

## Required Secret

The workflow expects this secret to be available to the caller workflow:

```text
AWS_DEPLOY_ROLE_ARN
```

That role must be assumable from GitHub Actions through OIDC and must have
permission to update the Lambda function, publish versions, and update the
target alias.

## The Makefile Contract

This workflow depends on the caller repository's `Makefile`.

The most important target is:

```bash
make package
```

`make package` must create the ZIP file at the same path passed through the
`zip_path` input. For example, if the workflow receives:

```yaml
zip_path: dist/download-lambda.zip
```

then the caller repository's `Makefile` must produce:

```text
dist/download-lambda.zip
```

A typical Lambda repository Makefile includes targets like:

```make
.PHONY: install lint format test package clean

install:
	uv sync --all-groups

lint:
	uv run ruff check src tests
	uv run ruff format --check src tests

format:
	uv run ruff check src tests --fix
	uv run ruff format src tests

test:
	uv run pytest

package:
	rm -rf build dist
	mkdir -p build dist
	uv pip install \
		--target build \
		--python .venv/bin/python \
		.
	cp -r src/* build/
	cd build && zip -r ../dist/example-lambda.zip .

clean:
	rm -rf build dist .pytest_cache .ruff_cache
```

The shared workflow intentionally does not know how each Lambda should be
packaged. That logic stays in each Lambda repository, where source layout,
dependencies, and artifact names belong.

## What Other Repositories Inherit

Repositories that use this workflow inherit a consistent deployment pipeline:

- Same Python and `uv` setup.
- Same required test, lint, and format checks.
- Same `make package` packaging entry point.
- Same ZIP artifact validation.
- Same AWS OIDC deployment flow.
- Same Lambda version publishing and alias promotion behavior.

Each Lambda repository remains responsible for:

- Its own source code and tests.
- Its own `pyproject.toml` and lock file.
- Its own `Makefile`.
- Its own deployment trigger branches.
- Its own `function_name`, `zip_path`, environment, and alias values.

## Local Usage

Before relying on CI, a Lambda repository should be able to run the same core
steps locally:

```bash
uv sync --all-groups
uv run pytest
uv run ruff check .
uv run ruff format --check .
make package
```

If those commands pass locally and the generated ZIP path matches the workflow
input, the repository is ready to use this shared deployment workflow.
