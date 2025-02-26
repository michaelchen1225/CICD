# 使用情境


### 遠端 repo 一旦有 push 就自動更新本地 dir

```yaml
stages:
  - auto-fetch

variables:
  TARGET_DIR: /path/to/your/target/dir

.git-fetch-template: &git-fetch-template
  - RUNNER_DIR=$(pwd)
  - cd ${TARGET_DIR}
  - git pull https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.com/${CI_PROJECT_PATH}.git $CI_COMMIT_REF_NAME
  - sudo rsync -avu --delete --exclude ".git" ${RUNNER_DIR}/ ${TARGET_DIR}


deploy_airflow_service:
  tags:
    - <shell executor>
  stage: auto-fetch
  script:
    - *git-fetch-template
    - <do other things>
```

> gitlab runner 只少需要有以下權限：

```bash
# /etc/sudoers.d/gitlab-runner
gitlab-runner ALL=(ALL) NOPASSWD: /usr/bin/rsync
```

### 打包 docker image & push 到 GitLab registry

> [Official doc](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-in-docker-executor)


```yaml
image_build:
  image: docker:19.03 
  services:
    - name: docker:19.03-dind # 
  tags: 
    - <docker executor> 
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker build -t $CI_REGISTRY_IMAGE/$CI_PROJECT_NAME:${IMAGE_TAG} .
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE/$CI_PROJECT_NAME:${IMAGE_TAG}
```

> When you use the dind service, you must instruct Docker to talk with the daemon started inside of the service.
> The daemon is available with a network connection instead of the default /var/run/docker.sock socket.
> Docker 19.03 does this automatically
> If your not using 19.03, you need to set the DOCKER_HOST environment variable to tcp://docker:2375.
> If you want to skip TLS verification, you can set the DOCKER_TLS_CERTDIR environment variable to "". (when the runner didn't config to use TLS)



* /etc/gitlab-runner/config.toml

```toml
[[runners]]
  name = "docker-runner"
  url = "https://gitlab.com/"
  id = 27401866
  token = "sjfoifjsoidjfops"
  token_obtained_at = 2023-09-01T08:54:52Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "docker:19.03"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/certs/client","/cache"]
    shm_size = 0
```