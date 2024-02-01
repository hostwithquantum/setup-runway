# hostwithquantum/setup-runway

![hostwithquantum/setup-runway](setup-runway-banner.jpeg)

A GitHub action to setup the runway CLI! Questions, issues? Please use discussions or the issue tracker on the repository. If you like what you see here, **we appreciate a** :star: and if you'd subscribe to [(our monthly) mailing list](https://runway.planetary-quantum.com/) to stay in the loop!

## Quick Start

```yaml
# ...
steps:
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
- uses: hostwithquantum/setup-runway@v0.3.0
  with:
    username: ${{ secrets.RUNWAY_USERNAME }}
    password: ${{ secrets.RUNWAY_PASSWORD }}
- run: runway whoami
```

## Options

Currently supported options:

| option        | default value       | description                                       |
|---------------|---------------------|---------------------------------------------------|
| username      | `<none>`            | username/email for runway                         |
| password      | `<none>`            | password for runway                               |
| add-key       | `false`             | if set to true, add the ssh key to runway         |
| setup-ssh     | `false`             | if set to true, setup ssh for `runway app deploy` |
| log-level     | `error`             | debug, info, warn, error                          |
| public-key    | `~/.ssh/id_rsa.pub` | ssh public key location                           |
| public-key    | `~/.ssh/id_rsa`     | ssh private key location                          |
| log-level     | `error`             | debug, info, warn, error                          |
| version       | `latest`            | `runway` cli version                              |
| controller    | ``                  | controller URL for Enterprise installations       |

> For the version, `latest` is fine. We strive to never break your workflows. But sometimes BC breaks are necessary. Because they usually involve our client and APIs, using `latest` helps to keep all interruptions to a minimum.

## Examples

### Setup the runway CLI to deploy

This is an example workflow which shows runway CLI setup and then how to use the CLI to deploy your app `cool-app`.

Once the client is setup, you can run all commands and play around with output and so on. To keep it simple, we're only deploying the code. :) Since the app exists already on runway we use the `runway gitremote` command to initialize the setup.

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
        fetch-depth: 0
    - name: create public/private key on GHA
      run: |
        mkdir -p ~/.ssh/
        echo "${{ secrets.PRIVATE_KEY }}" > ~/.ssh/id_rsa
        echo "${{ secrets.PUBLIC_KEY }}" > ~/.ssh/id_rsa.pub
        chmod 0600 ~/.ssh/id_rsa*
    - uses: hostwithquantum/setup-runway@v0.3.0
      with:
        username: ${{ secrets.RUNWAY_USERNAME }}
        password: ${{ secrets.RUNWAY_PASSWORD }}
        setup-ssh: true
    - run: runway gitremote -a ${YOUR_APPLICATION_NAME}
    - run: runway app deploy
```

### Running e2e tests

GitHub Actions provides a robust and comprehensive environment to run e2e tests and here's how runway can help:

The following workflow leverages some of the context in form of `${{ github.run_id }}`. We'll use this _identifier_ to deploy an app with a unique name. Another viable option is to use the pull-requests's number: `${{ github.event.number }}`.

Once deployed, you can run end-to-end tests against it and in the end, shut it down by deleting the app (and key). :) If you decide to keep the application to have a preview available, you may also do that.

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
        ssh-keygen -b 2048 -t rsa -f ~/.ssh/test-runner -c "test-key-${{ github.run_id }}" -q -N ""
    - name: install CLI, login and add ssh key
      uses: hostwithquantum/setup-runway@v0.3.0
      with:
        username: ${{ secrets.RUNWAY_USERNAME }}
        password: ${{ secrets.RUNWAY_PASSWORD }}
        public-key: ~/.ssh/test-runner.pub
        private-key: ~/.ssh/test-runner
        add-key: true
        setup-ssh: true
    - name: create app on runway
      run: runway app create -a $APP_NAME || runway gitremote -a $APP_NAME
    - name: deploy your app to runway
      run: runway app deploy
    # this is where your tests run!
    - name: run your e2e tests here
      run: curl https://$APP_NAME.pqapp.dev/
    # then hopefully you are done :)
    - name: cleanup app
      if: always()
      run : runway app rm -a $APP_NAME || true
    - name: cleanup key - this is brute force
      if: always()
      run: runway key rm "test-key-${{ github.run_id }}" || true
```

### Preview apps

In the previous example, we mentioned that keeping an app for people (humans!) to look at it, may be beneficial.

The following workflow expands on those concepts and deletes an application from runway when a pull-request is closed (merged or closed without merge). This example _assumes_ that you constructed the application name like, `my-app-${{ github.event.number }}` (instead of `github.run_id`).

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
    - uses: hostwithquantum/setup-runway@v0.3.0
      with:
        username: ${{ secrets.QUANTUM_RUNWAY_USERNAME }}
        password: ${{ secrets.QUANTUM_RUNWAY_PASSWORD }}
    - run: runway app delete my-app-${{ github.event.number }} || true
```
