# gentoo-travis-scripts

Helper scripts that bootstrap a Portage environment inside a [Travis CI][travis]
job so a Gentoo overlay can be checked with [`repoman`][repoman] on every
push and pull request.

The scripts are designed to be vendored into an overlay repository as a git
submodule and invoked from `.travis.yml`. They install a pinned Portage
release, lay down a minimal Portage configuration, register the overlay under
test, and create the `portage` user and group that the package manager expects.

[travis]: https://www.travis-ci.com/
[repoman]: https://wiki.gentoo.org/wiki/Repoman

## What it does

* `get_latest_portage_version` — query the GitHub GraphQL API for the newest
  `gentoo/portage` tag and print its version. Use it to pin `PORTAGE_VERSION`
  to the latest release at the start of each build.
* `install_portage` — download the requested Portage release tarball, install
  Portage and Repoman into `PORTAGE_ROOT`, and stage `qa_data.yaml` where
  Repoman expects it.
* `setup_portage` — fetch the Gentoo ebuild tree from the GitHub mirror, write
  `repos.conf` entries for both the `gentoo` repository and the overlay under
  test, write a minimal `make.conf`, and symlink `make.profile`.
* `add_required_user_and_group` — create the `portage` user and group (UID/GID
  250) and add the Travis build user to the `portage` group.

## Requirements

* A Travis CI job running on Ubuntu (`dist: focal` or newer) with `sudo` access.
* A [GitHub personal access token][token] with the `public_repo` scope, exposed
  to the build as `GITHUB_GRAPHQL_QUERY_TOKEN`. Store it as an [encrypted
  Travis environment variable][travis-secure] — never commit the plaintext
  token.
* `jq` and `curl` available in the build image (both are pre-installed on the
  Travis Ubuntu images).

[token]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
[travis-secure]: https://docs.travis-ci.com/user/environment-variables/#defining-encrypted-variables-in-travisyml

## Configuration

The scripts read the following environment variables:

| Variable                    | Required | Default              | Purpose                                                    |
| --------------------------- | -------- | -------------------- | ---------------------------------------------------------- |
| `GITHUB_GRAPHQL_QUERY_TOKEN`| yes      | —                    | Token used by `get_latest_portage_version`.                |
| `PORTAGE_ROOT`              | yes      | —                    | Prefix into which Portage is installed and configured.     |
| `PORTAGE_VERSION`           | yes      | —                    | Portage release to install (e.g. `3.0.13`).                |
| `PORTAGE_CONFIGROOT`        | yes      | —                    | Portage configuration root, normally equal to `PORTAGE_ROOT`. |
| `PORTAGE_PROFILE_VERSION`   | no       | `17.1`               | Gentoo profile version to symlink as `make.profile`.       |
| `PORTAGE_PROFILE_ARCH`      | no       | `${TRAVIS_CPU_ARCH}` | Profile architecture (e.g. `amd64`, `arm64`).              |

## Usage

Add this repository as a submodule of your overlay:

```sh
git submodule add https://github.com/expeditioneer/gentoo-travis-scripts.git .travis-scripts
```

Then drop a `.travis.yml` like the following into the overlay root:

```yaml
dist: focal

language: python
python:
  - pypy3

git:
  depth: 1
  quiet: true

env:
  global:
    - secure: <ENCRYPTED_GITHUB_GRAPHQL_QUERY_TOKEN>
    - PORTAGE_ROOT="$(mktemp --directory)"
    - PORTAGE_VERSION="$(.travis-scripts/get_latest_portage_version)"
    - PORTAGE_CONFIGROOT="${PORTAGE_ROOT}"
    - PATH="${PORTAGE_ROOT}/usr/bin:${PATH}"

install:
  - .travis-scripts/install_portage
  - .travis-scripts/setup_portage
  - .travis-scripts/add_required_user_and_group

before_script:
  - pip install PyYAML

script:
  - repoman full --include-dev

notifications:
  email:
    on_failure: change
```

## License

[GPL-3.0](LICENSE)
