# Description

This repository containt an example of buildpack integration pipeline and could be easily customize to fit
your requirement of course using [Concourse CI](http://concourse.ci/) 


!["Image"](images/img-pipeline.png)

#### Docker Images used
We use the docker image [getourneau/alpine-cfcli-golang](https://github.com/shinji62/alpine-docker-cfcli-golang) 
Which containt the CF cli and golang on a an image based on alpine linux

## Resources

### Buildpack resource
We use the Github release of buildpack.
```yaml
- name: gh-release-binary-buildpack
  type: github-release
  source:
    user: cloudfoundry
    repository: binary-buildpack
    access_token: {{github-access-token}}
```

### Environment configuration
Most of the time there are multiple environment like PROD, STG, DEV and so on.
We use the github repository and the branch to specify multiple env, and smoke test config.
You can take a look here [env-config](https://github.com/shinji62/env-config/tree/pcfdev)
```yaml
- name: gh-env-config-pcfdev
  type: git
  source:
    uri: git@github.com:shinji62/env-config.git
    branch: pcfdev
    private_key: {{private-key-github-concourse}}
```

Enviroment config file example

```json
{
    "env": {
        "api": "api.local.pcfdev.io",
        "apps_domain": "local.pcfdev.io",
        "password": "admin",
        "user": "admin"
    },
    "smoke_test": {
        "backend": "diego",
        "cleanup": true,
        "enable_windows_tests": false,
        "logging_app": "",
        "org": "CF-SMOKE-ORG",
        "runtime_app": "",
        "skip_ssl_validation": true,
        "space": "CF-SMOKE-SPACE",
        "suite_name": "CF_SMOKE_TESTS",
        "use_existing_org": false,
        "use_existing_space": false
    }
}
```


## Job
Job are pretty easy you can just take a look at the pipeline

### Buildpack uploading
`BUILDPACK_NAME` is the name of the buildpack in cloudfoundry 
```yaml
- name: binary_buildpack_pcfdev
  public: true
  serial: true
  plan:
  - aggregate:
    - get: buildpack-github-release
      resource: gh-release-binary-buildpack
      trigger: true
    - get: env-info
      resource: gh-env-config-pcfdev
      trigger: true
    - get: buildpack-pipeline
  - task: upload-binary-buildpack
    file: buildpack-pipeline/ci/updatebuildpack/updatebuildpack.yml
    params:
      BUILDPACK_NAME: binary_buildpack
```


### Smoke test and Acceptance Test
There is currently two type of test, [smoke-test](https://github.com/cloudfoundry/cf-smoke-tests) and [acceptance-test](https://github.com/cloudfoundry/cf-acceptance-tests) from Cloudfoundry

Important
* smoke-test can be run in any environment.
* Acceptance-test are more aggressive and should be run in non-production env.


```yaml
- name: acceptance-sandbox
  public: true
  serial: true
  plan:
  - aggregate:
    - get: acceptance-tests
      resource: cf-acceptance-tests
      trigger: true
      passed: [smoketest-sandbox]
    - get: env-info
      resource: gh-env-config-pcfdev
      trigger: true
    - get: buildpack-pipeline
  - task: acceptance-tests
    file: buildpack-pipeline/ci/acceptancetests/acceptancetests.yml 

```





## Usage
Deploy to concourse is easy as 
```bash
fly -t target set-pipeline -c ci/pipeline.yml --load-vars-from credentials.yml -p buildpack-pipeline
```





