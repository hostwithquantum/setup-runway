# hostwithquantum/setup-runway

![hostwithquantum/setup-runway](setup-runway-banner.jpeg)

A GitHub action to setup the Runway CLI! Questions, issues? Please use discussions or the issue tracker on the repository. If you like what you see here, **we appreciate a** :star: and if you'd subscribe to [(our monthly) mailing list](https://outreach.planetary-quantum.com/) and [check out the website to stay](https://www.runway.horse/) in the loop!

---

This action supports amd64 and arm64 Linux runners.

## Quick Start

```yaml
# ...
steps:
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
- uses: hostwithquantum/setup-runway@v0.5.0
  with:
    username: ${{ secrets.RUNWAY_USERNAME }}
    password: ${{ secrets.RUNWAY_PASSWORD }}
- run: runway whoami
```

## Action Inputs

The currently supported inputs:

| option              | default value       | description                                       |
|---------------------|---------------------|---------------------------------------------------|
| username            | _empty_             | username/email for Runway                         |
| password            | _empty_             | password for Runway                               |
| add-key             | `false`             | if set to true, add the ssh key to Runway         |
| setup-ssh           | `false`             | if set to true, setup ssh for `runway app deploy` |
| log-level           | `error`             | debug, info, warn, error                          |
| public-key          | _empty_             | a public ssh key (from a secret)                  |
| private-key         | _empty_             | a public ssh key (from a secret)                  |
| public-key-location | `~/.ssh/id_rsa.pub` | ssh public key location                           |
| public-key-location | `~/.ssh/id_rsa`     | ssh private key location                          |
| version             | `latest`            | `runway` cli version                              |
| controller          | _empty_             | controller URL for Enterprise installations       |

> [!TIP]
> For the Runway CLI version, `latest` is fine. We strive to never break your workflows. But sometimes BC breaks are necessary. Because they usually involve our client and APIs, using `latest` helps to keep all interruptions to a minimum.

> [!IMPORTANT]
> There's a BC break in this action in `v0.5.0`, previously the `public-key` and `private-key` inputs contained paths. They have been renamed to `public-key-location` and `private-key-location` to make their purpose known. The _new_ `public-key` and `private-key` inputs are optional and contain your (SSH) public and private key if used.

## Examples

The following examples show how this action and the Runway CLI can be used in workflows.

### Deployments

This is an example workflow which shows Runway CLI setup and then how to use the CLI to deploy your application `cool-app`.

Once the CLI is setup, you can run all commands and play around with output and so on. To keep it simple, we're only deploying the code. :) Since the application exists already on Runway we use the `runway gitremote` command to initialize the setup.

```yaml
# .github/workflows/release.yml
---
name: release

on:
  push:
    tags:

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      YOUR_APPLICATION_NAME: cool-app
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0 # this is important
    - uses: hostwithquantum/setup-runway@v0.5.0
      with:
        username: ${{ secrets.RUNWAY_USERNAME }}
        password: ${{ secrets.RUNWAY_PASSWORD }}
        private-key: ${{ secrets.PRIVATE_KEY }}
        public-key: ${{ secrets.PUBLIC_KEY }}
    - run: runway gitremote -a ${YOUR_APPLICATION_NAME}
    - run: runway app deploy
    - run: runway logout
```

### Running e2e tests

GitHub Actions provides a robust and comprehensive environment to run e2e tests and here's how Runway can help:

The following workflow leverages some of the context in form of `${{ github.run_id }}`. We'll use this _identifier_ to deploy an application with a unique name. Another viable option is to use the pull-requests's number: `${{ github.event.number }}`.

Once deployed, you can run end-to-end tests against it and in the end, shut it down by deleting the application (and key). :) If you decide to keep the application to have a preview available, you may also do that.

```yaml
# .github/workflows/e2e.yml
---
name: e2e

on: 
  pull_request:

jobs:
  deploy_app:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0 # this is important!
    - name: create the application name
      run: echo "APP_NAME=my-app-${{ github.run_id }}" >> $GITHUB_ENV
    - name: create an ssh key just for this run
      run: |
        mkdir -p ~/.ssh/
        ssh-keygen -b 2048 -t rsa -f ~/.ssh/test-runner -c "test-key-${{ github.run_id }}" -q -N ""
    - name: install Runway CLI, login and add ssh key
      uses: hostwithquantum/setup-runway@v0.5.0
      with:
        username: ${{ secrets.RUNWAY_USERNAME }}
        password: ${{ secrets.RUNWAY_PASSWORD }}
        public-key-location: ~/.ssh/test-runner.pub
        private-key-location: ~/.ssh/test-runner
        add-key: true
        setup-ssh: true
    - name: create application on Runway
      run: runway app create -a $APP_NAME || runway gitremote -a $APP_NAME
    - name: deploy your application to Runway
      run: runway app deploy
    # this is where your tests run!
    - name: run your e2e tests here
      run: curl https://$APP_NAME.pqapp.dev/
    # then hopefully you are done :)
    - name: cleanup application
      if: always()
      run : runway app rm -a $APP_NAME || true
    - name: cleanup key - this is brute force
      if: always()
      run: runway key rm "test-key-${{ github.run_id }}" || true
    - name: logout
      if: always()
      run: runway logout
```

### Preview apps

In the previous example, we mentioned that keeping an application for people (humans!) to look at it, may be beneficial.

The following workflow expands on those concepts and deletes an application from Runway when a pull-request is closed (merged or closed without merge). This example _assumes_ that you constructed the application name like, `my-app-${{ github.event.number }}` (instead of `github.run_id`).

```yaml
name: delete app

on:
  pull_request:
    types:
    - closed

jobs:
  delete:
    runs-on: ubuntu-latest
    steps:
    - uses: hostwithquantum/setup-runway@v0.5.0
      with:
        username: ${{ secrets.QUANTUM_RUNWAY_USERNAME }}
        password: ${{ secrets.QUANTUM_RUNWAY_PASSWORD }}
    - run: runway app delete my-app-${{ github.event.number }} || true
    - run: runway logout
```
