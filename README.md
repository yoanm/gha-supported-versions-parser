# gha-supported-versions-parser

![GitHub Tag](https://img.shields.io/github/v/tag/yoanm/gha-supported-versions-parser?sort=semver&logo=githubactions&logoColor=white&logoSize=auto&link=https%3A%2F%2Fgithub.com%2Fyoanm%2Fgha-supported-versions-parser%2Freleases)

Lightweight composite github action parsing supported versions file

---

Designed to avoid the hassle of managing versions on multiple workflow file.

It circumvent two issues:
- Inability to use `env` variable on `strategy.matrix`, while `steps` or `needs` is available. Using this action allows to define complex matrix with the different versions
- 

### Supported versions file
A supported version file is simple a JSON object with the following structure:
````json
{
  "{DEPENDENCY_NAME}": {"min": "x.y", "max": "x.y", "next": "x.y"},
  ...
  "{DEPENDENCY_NAME}": {"min": "x.y", "max": "x.y", "next": "x.y"}
}
````

- `{DEPENDENCY_NAME}`: e.g. `php`, `symfony` or any dependency your project needs to assess
- `min`: Minimal version to assess on the CI
- `max`: Maximal version to assess on the CI
- `next`: Next version to assess on the CI, used for nightly builds to ensure project will work with currently in development version

## Usage

Basic example with PHP:
```yaml
      # Download the source code
      - name: Check out code
        uses: actions/checkout@v5

      # Fetch the versions for a specific dependency
      - name: Fetch PHP supported versions
        id: fetch-php-versions
        uses: yoanm/gha-supported-versions-parser@v1
        with:
          # Replace the following with the right path for you
          path: .github/workflows/supported-versions.json
          dependency: php

      # Use them to setup the container
      - name: Setup PHP
        id: setup-php
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ steps.fetch-php-versions.outputs.min }}
```
Based on this example, the following values are available for the current job:
- `steps.fetch-php-versions.outputs.min`
- `steps.fetch-php-versions.outputs.max`
- `steps.fetch-php-versions.outputs.next`

### Complex matrix management

1. Fetch versions on a dedicated job
2. Create a matrix for another job based on those versions

```yaml
jobs:
  fetch-supported-versions:
    name: Fetch supported versions
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      php-min: ${{ steps.fetch-php-versions.outputs.min }}
      php-max: ${{ steps.fetch-php-versions.outputs.max }}
      php-next: ${{ steps.fetch-php-versions.outputs.next }}
      symfony-min: ${{ steps.fetch-symfony-versions.outputs.min }}
      symfony-max: ${{ steps.fetch-symfony-versions.outputs.max }}
      symfony-next: ${{ steps.fetch-symfony-versions.outputs.next }}
    steps:
      # Use the following to download the supported version file (a checkout would also work)
      - name: Fetch supported versions file
        id: fetch-file
        uses: yoanm/gha-supported-versions-parser/github-downloader@v1
        with:
          file-path: .github/workflows/supported-versions.json

      - name: Fetch PHP supported versions
        id: fetch-php-versions
        uses: yoanm/gha-supported-versions-parser@v1
        with:
          path: ${{ steps.fetch-file.outputs.path }}
          dependency: php

  tests:
    name: PHP ${{ matrix.php-version }} & Symfony {{ matrix.symfony-version }}
    needs: [fetch-supported-versions]
    runs-on: ubuntu-latest
    permissions:
      contents: read
    strategy:
      matrix:
        include:
          # Up to date versions
          - php-version: '${{ needs.fetch-supported-versions.outputs.php-max }}'
            symfony-version: '${{ needs.fetch-supported-versions.outputs.symfony-max }}'
          # Bare minimum
          - php-version: '${{ needs.fetch-supported-versions.outputs.php-min }}'
            symfony-version: '${{ needs.fetch-supported-versions.outputs.symfony-min }}'
          # Late migration - PHP
          - php-version: ${{ needs.fetch-supported-versions.outputs.php-min }}
            symfony-version: '${{ needs.fetch-supported-versions.outputs.symfony-max }}'
          # Late migration - Symfony
          - php-version: '${{ needs.fetch-supported-versions.outputs.php-max }}'
            symfony-version: '${{ needs.fetch-supported-versions.outputs.symfony-min }}'
    steps:
      - name: Check out code
        uses: actions/checkout@v5

        # Configure and test your code ...
        ...
```
## yoanm/gha-supported-versions-parser
### Inputs
- `path`: **Required** Path to the supported versions file from inside the repository
- `dependency`: **Required** Path to the supported versions file from inside the repository
- `with-summary`: Default to `true`, if not `false`, a summary containing fetch versions will be posted for the workflow run

### Outputs
- `min`: Minimal version configured for the given dependency
- `max`: Maximal version configured for the given dependency
- `next`: Next version configured for the given dependency

## yoanm/gha-supported-versions-parser/github-downloader
Simple composite action relying on curl to download the supporter version file.

Use it in case you don't want/need to checkout the whole source code

### Inputs
- `file-path`: **Required** Path to the supported versions file from inside the repository
- `ref`: Default to `github.ref`, provide the right one otherwise
- `fetch-dir`: Default to `runner.temp`, provide a specific one if needed

### Outputs
- `path`: Path to the downloaded file
