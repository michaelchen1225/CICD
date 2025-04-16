# 使用情境

* [遠端 repo 一旦有 push 就自動更新本地 dir](#遠端-repo-一旦有-push-就自動更新本地-dir)

* [打包 docker image & push 到 GitLab registry](#打包-docker-image--push-到-gitlab-registry)

* [當上游 pipeline 打包好容器後，觸發下游 pipeline 來替換 image tag](#當上游-pipeline-打包好容器後觸發下游-pipeline-來替換-image-tag)

  * [事前設定](#事前設定)
  * [上游 pipeline](#上游-pipeline)
  * [下游 pipeline](#下游-pipeline)

### 遠端 repo 一旦有 push 就自動更新本地 dir

```yaml
stages:
  - auto-fetch

variables:
  TARGET_DIR: /path/to/your/target/dir


.git-fetch-template: &git-fetch-template
  - pwd
  - RUNNER_DIR=$(pwd)
  - git branch
  - echo latest commit = ${CI_COMMIT_SHA} 
  - sudo rsync -avu --exclude ".git" --chown=root:root ${RUNNER_DIR}/ ${TARGET_DIR}



some_job:
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

### 當上游 pipeline 打包好容器後，觸發下游 pipeline 來替換 image tag

* **code-repo**：負責儲存程式碼、image

* **k8s-repo**：負責儲存 k8s yaml

#### 事前設定

1. 下游 pipeline 會被上游 pipeline 的 Job Token 觸發，前提是要「允許」上游 pipeline 觸發下游 pipeline：前往「**Settings**」>「**CI/CD**」>「**Job token permissions**」，把上游 pipeline 的 repo 加入白名單：

    ![alt text](image-15.png)

2. 下游 pipeline 會繼承上游的變數，因此得確保上游 pipeline **有權限** 去設定下游 pipeline 的變數。

    > 以範例來說，`k8s-repo` 為下游，因此需要將「**Settings**」>「**CI/CD**」>「**Variables**」>「**Minimum role to use pipeline variables**」，改成「**Developer**」。

    ![alt text](image-16.png)

3. 需要一個 shell executor

    > 參考[這裡](./04-regist-runner.md)的介紹

4. 設定 ssh key

   * 進入 executor 所在的 server，切換身分成 gitlab-runner：

      ```bash
      sudo su - gitlab-runner
      ```

    * 產生 ssh key：
  
      ```bash
      ssh-keygen -t rsa -C "gitlab-runner"
      ```
      > 一直按 enter

    * 將 public key 複製到 k8s-repo 的 `Settings` > `Repository` > `Deploy keys` 中：

      ```bash
      cat ~/.ssh/id_rsa.pub
      ``` 

### 上游 pipeline

```yaml
# code-repo/.gitlab-ci.yml
variables:
  IMAGE_TAG: v0.0.3
stages:
  - build_image
  - update_image_tag

.build_image:
  stage: build_image
  image: docker:19.03 
  services:
    - name: docker:19.03-dind 
  tags: 
    - my-builder
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker build -t $CI_REGISTRY_IMAGE/$CI_PROJECT_NAME:${IMAGE_TAG} .
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE/$CI_PROJECT_NAME:${IMAGE_TAG}

update_image_tag:
  stage: update_image_tag
  trigger:
    project: test-ci6881616/argo/k8s-repo
```

### 下游 pipeline

```yaml
# k8s-repo/.gitlab-ci.yml
change_tag:
  tags: 
  - test-ci-group
  rules:
    - if: $CI_PIPELINE_SOURCE == "pipeline"
  script:
    - git clone https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.com/${CI_PROJECT_PATH}.git
    - cd ${CI_PROJECT_NAME}
    - git remote set-url --push origin git@gitlab.com:test-ci6881616/argo/k8s-repo.git
    - OLD_TAG=$(grep "image" ./k8s-files/pod.yaml | awk -F ':' '{print $3}')
    - sed -i "s/${OLD_TAG}/${IMAGE_TAG}/g" ./k8s-files/pod.yaml
    - git add .
    - git commit -m "Update image tag to ${IMAGE_TAG} [skip ci]"
    - git push origin HEAD:main
```

* 使用 `rules + if` 來限定 change_tag 這個 job 只會在上游 pipeline 被觸發時才會執行。

* `$IMAGE_TAG` 是上游 pipeline 的變數，會被下游 pipeline 繼承。

* `git commit 中的 [skip ci] 是為了避免下游 pipeline 不斷的觸發 job，因此在更換 image tag 時，會跳過本次的 pipeline。

**結果**：

![alt text](image-17.png)



![alt text](image-18.png)