Docker Github Actions Runner
============================

This will run the [new self-hosted github actions runners](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/hosting-your-own-runners).

## Quick-Start
### Adding a self-hosted runner to an organization
1. On GitHub.com, navigate to the main page of the organization.
1. Under your organization name, click  Settings.
1. In the left sidebar, click  Actions, then click Runners.
1. Click New runner, then click New self-hosted runner.
1. Note the token

### Create your `docker-compose.yml`
```yaml
version: '2.3'
services:
  worker:
    image: ghcr.io/terrateamio/self-hosted-runner:latest
    restart: unless-stopped
    environment:
      ORG_NAME: myGithubOrg
      RUNNER_NAME: terrateam-self-hosted
      RUNNER_TOKEN: runnerToken
      RUNNER_WORKDIR: /tmp/runner/work
      RUNNER_SCOPE: 'org'
      LABELS: terrateam-self-hosted
      EPHEMERAL: 1
    security_opt:
      - label:disable # needed on SELinux systems to allow docker container to manage other docker containers
    volumes:
      - '/var/run/docker.sock:/var/run/docker.sock'
      - '/tmp/runner:/tmp/runner' # docker-in-docker requires the same path on the host and inside the container
```

### Update your Terrateam GitHub Actions workflow file
Replace `ubuntu-latest` with `terrateam-self-hosted`:
```
bender@nibbler:~/terrateam-demo/terraform$ git diff .github/workflows/terrateam.yml
diff --git a/.github/workflows/terrateam.yml b/.github/workflows/terrateam.yml
index 1f77912..2eb8f5f 100644
--- a/.github/workflows/terrateam.yml
+++ b/.github/workflows/terrateam.yml
@@ -24,7 +24,7 @@ jobs:
     permissions: # Required to pass credentials to the Terrateam action
         id-token: write
         contents: read
-    runs-on: ubuntu-latest
+    runs-on: terrateam-self-hosted
     timeout-minutes: 1440
     name: Terrateam Action
     steps:
bender@nibbler:~/terrateam-demo/terraform$
```

Commit and push to your `default branch`.

### Run Docker Compose
```sh
docker-compose up -d
```

### Run a Terrateam operation
Validate the job is running on your self-hosted runner by running `docker-compose logs -f`.

## Environment Variables ##

| Environment Variable | Description |
| --- | --- |
| `RUN_AS_ROOT` | Boolean to run as root. If `true`: will run as root. If `True` and the user is overridden it will error. If any other value it will run as the `runner` user and allow an optional override. Default is `true` |
| `RUNNER_NAME` | The name of the runner to use. Supersedes (overrides) `RUNNER_NAME_PREFIX` |
| `RUNNER_NAME_PREFIX` | A prefix for runner name (See `RANDOM_RUNNER_SUFFIX` for how the full name is generated). Note: will be overridden by `RUNNER_NAME` if provided. Defaults to `github-runner` |
| `RANDOM_RUNNER_SUFFIX` | Boolean to use a randomized runner name suffix (preceded by `RUNNER_NAME_PREFIX`). Will use a 13 character random string by default. If set to a value other than true it will attempt to use the contents of `/etc/hostname` or fall back to a random string if the file does not exist or is empty. Note: will be overridden by `RUNNER_NAME` if provided. Defaults to `true`. |
| `ACCESS_TOKEN` | A [github PAT](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) to use to generate `RUNNER_TOKEN` dynamically at container start. Not using this requires a valid `RUNNER_TOKEN` |
| `APP_ID` | The github application ID. Must be paired with `APP_PRIVATE_KEY` and should not be used with `ACCESS_TOKEN` or `RUNNER_TOKEN` |
| `APP_PRIVATE_KEY` | The github application private key. Must be paired with `APP_ID` and should not be used with `ACCESS_TOKEN` or `RUNNER_TOKEN` |
| `APP_LOGIN` | The github application login id. Can be paired with `APP_ID` and `APP_PRIVATE_KEY` if default value extracted from `REPO_URL` or `ORG_NAME` is not correct. Note that no default is present when `RUNNER_SCOPE` is 'enterprise'. |
| `RUNNER_SCOPE` | The scope the runner will be registered on. Valid values are `repo`, `org` and `ent`. For 'org' and 'enterprise', `ACCESS_TOKEN` is required and `REPO_URL` is unnecessary. If 'org', requires `ORG_NAME`; if 'ent', requires `ENTERPRISE_NAME`. Default is 'repo'. |
| `ORG_NAME` | The organization name for the runner to register under. Requires `RUNNER_SCOPE` to be 'org'. No default value. |
| `ENTERPRISE_NAME` | The enterprise name for the runner to register under. Requires `RUNNER_SCOPE` to be 'enterprise'. No default value. |
| `LABELS` | A comma separated string to indicate the labels. Default is 'default' |
| `REPO_URL` | If using a non-organization runner this is the full repository url to register under such as 'https://github.com/terrateamio/repo' |
| `RUNNER_TOKEN` | If not using a PAT for `ACCESS_TOKEN` this will be the runner token provided by the Add Runner UI (a manual process). Note: This token is short lived and will change frequently. `ACCESS_TOKEN` is likely preferred. |
| `RUNNER_WORKDIR` | The working directory for the runner. Runners on the same host should not share this directory. Default is '/_work'. This must match the source path for the bind-mounted volume at RUNNER_WORKDIR, in order for container actions to access files. |
| `RUNNER_GROUP` | Name of the runner group to add this runner to (defaults to the default runner group) |
| `GITHUB_HOST` | Optional URL of the Github Enterprise server e.g github.mycompany.com. Defaults to `github.com`. |
| `DISABLE_AUTOMATIC_DEREGISTRATION` | Optional flag to disable signal catching for deregistration. Default is `false`. Any value other than exactly `false` is considered `true`. See [here](https://github.com/terrateamio/self-hosted-runner/issues/94) |
| `CONFIGURED_ACTIONS_RUNNER_FILES_DIR` | Path to use for runner data. It allows avoiding reregistration each the start of the runner. No default value. |
| `EPHEMERAL` | Optional flag to configure runner with [`--ephemeral` option](https://docs.github.com/en/actions/hosting-your-own-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling). Ephemeral runners are suitable for autoscaling. |
| `DISABLE_AUTO_UPDATE` | Optional environment variable to [disable auto updates](https://github.blog/changelog/2022-02-01-github-actions-self-hosted-runners-can-now-disable-automatic-updates/). Auto updates are enabled by default to preserve past behavior. Any value is considered truthy and will disable them. |
| `START_DOCKER_SERVICE` | Optional flag which automatically starts the docker service if set to `true`. Useful when using [sysbox](https://github.com/nestybox/sysbox). Defaults to `false`. |
| `NO_DEFAULT_LABELS` | Optional environment variable to disable adding the default self-hosted, platform, and architecture labels to the runner. Any value is considered truthy and will disable them. |
