# gentoo travis ci scripts

These scripts can be used with Travis CI in an custom gentoo overlay.

## How to use these scripts

1.) add this repository as a submodule to your overlay repository
1.) create a GITHUB GRAPHQL QUERY TOKEN to get the latest portage version
1.) create an encryption key in travis for GITHUB Query Token with the key _GITHUB_GRAPHQL_QUERY_TOKEN_

# example of .travis.yaml
Here is this repo cloned in .travis-scripts directory of the overlay:
 
```shell script
dist: bionic

language: python

python:
  - pypy3

git:
  depth: 1
  quiet: true

env:
  global:
    - secure: _ENCRYPTED_GITHUB_GRAPHQL_QUERY_TOKEN_
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

after_success:
  - bash <(curl --silent https://copilot.blackducksoftware.com/ci/travis/scripts/upload)

notifications:
  email:
    on_failure: change
```