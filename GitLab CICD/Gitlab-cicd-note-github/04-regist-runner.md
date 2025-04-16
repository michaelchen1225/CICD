# 在自己的 server 上註冊 Gitlab Runner

如果想要讓 .gitlab-ci.yml 的 script 跑在自己的 server 上，可以參考以下步驟：

* 在 server 上安裝 gitlab runner

* 到 repo 註冊 server 上的 gitlab runner，表示若該 repo 指定用這個 runner 跑 CI/CD，則會跑在 server 上。

* 在 .gitlab-ci.yml 中指定 runner tag

## 目錄

* [安裝 gitlab runner](#安裝-gitlab-runner)

* [註冊 gitlab runner](#註冊-gitlab-runner)
  * [註冊群組 runner](#註冊群組-runner)
  * [使用指令註冊 runner](#使用指令註冊-runner)

* [使用 runner tag (指定註冊的 runner 來承接 Job)](#使用-runner-tag-指定註冊的-runner-來承接-job)

* [列出 server 上註冊過的 runner & 確認 runner 是否活著](#列出-server-上註冊過的-runner--確認-runner-是否活著)

* [Gitlab Runner 罷工了怎麼辦？](#gitlab-runner-罷工了怎麼辦)

* [移除 runner](#移除-runner)

## 安裝 gitlab runner

> [official doc](https://docs.gitlab.com/runner/install/)

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
```

```bash
sudo apt install gitlab-runner
```

```bash
sudo systemctl enable gitlab-runner
sudo systemctl status gitlab-runner
```

## 註冊 gitlab runner

(https://docs.gitlab.com/runner/register/)

> 事前需準備一個 gitlab repo

```bash
sudo gitlab-runner register
```
> 輸入指令後，會出現一些提示，依照提示填入即可：

* Enter the GitLab instance URL (for example, https://gitlab.com/):
  * 自建的 gitlab server url：`http://your-gitlab-server.com/`
  * 使用 GitLab.com：直接 Enter
  
* Enter the registration token:
  * 在 GitLab 的 repo 中，進入 `Settings` -> `CI/CD` -> `Runners`，找到 `Project runners` 區塊，在「New Project Runner」旁邊有三個點，按下去之後複製 token 回到 terminal 貼上。

* Enter a description for the runner:
  * 這個 runner 的描述，例如：`my vm runner`
  
* Enter tags for the runner (comma-separated):
  * 給予 runner 的 tag，例如：`my-vm-runner`

* Enter an optional maintenance note for the runner.
  * 沒特定需求直接 Enter

* Enter the executor:
  * executor 種類有很多種，可參考[官網](https://docs.gitlab.com/runner/executors/)挑選，這邊選擇 `shell`
  * 常見 executor：
    * shell：直接在特定 server 上跑 shell command，須事前在 server 上搞定 gitlab-runner 的權限問題與安裝會用到的指令。
    * ssh：功能和 shell 一樣，但是透過 ssh 連線到 server 上跑 script，須事前在 server 上搞定 ssh 的相關設定。 
    * docker：在 docker container 中跑 script，須事前在 server 上安裝 docker，並在 .gitlab-ci.yml 中指定 docker container 的 image。
    * kubernetes：在 k8s 中跑 script(通常是 k8s 相關操作)，須事前在 server 上安裝 k8s。

  * Gitlab repo --> gitlab-runner --> executor 的關係像是：老闆 --> 工人 --> 工作方式

  * 註冊完成後，可以在 GitLab 的 repo 中，進入 `Settings` -> `CI/CD` -> `Runners`，看到剛剛註冊的 runner。

### 註冊群組 runner

如果你想讓某群組中的所有 repo 共用同一個 runner，可以選擇註冊群組 runner，就不用每個 repo 都註冊一次了。

至於註冊步驟，參考官網很快就搞定了：(https://docs.gitlab.com/ci/runners/runners_scope/#create-a-group-runner-with-a-runner-authentication-token)

### 使用指令註冊 runner

```bash
sudo gitlab-runner register -n \
  --url "https://gitlab.com/" \
  --registration-token REGISTRATION_TOKEN \
  --executor docker \
  --description "My Docker Runner" \
  --docker-image "docker:19.03" \
  --docker-privileged \
  --docker-volumes "/certs/client"
```


## 使用 runner tag (指定註冊的 runner 來承接 Job)

* 編輯 .gitlab-ci.yml：

```bash
cd  <your-repo>
vim .gitlab-ci.yml
```
```yaml
create_hello_file:
  tags:
    - my-vm-runner
  script:
    - echo "hello" > /tmp/hello.txt
```

> 在 push 後自動在 server 上的 /tmp/ 下產生 hello.txt 檔案。


* 提交變更：

```bash
git add .
git commit -m "create hello.txt"
git push
```

* 前往 Gitlab 確認 CI/CD 的 Job 是否有跑成功：

<img src="image-14.png" width="300px">

* 在 server 上查看 /tmp/ 下是否有 hello.txt 檔案：

```bash
cat /tmp/hello.txt
```


## 列出 server 上註冊過的 runner & 確認 runner 是否活著

* 列出 server 上註冊過的 runner：

```bash
gitlab-runner list
```

* 確認 runner 是否活著：

```bash
gitlab-runner verify
```

## Gitlab Runner 罷工了怎麼辦？

之前遇到一種情況，明明 .gitlab-ci.yml 中的 tag 指定了以註冊的 Runner，但 Runner 卻一直不承接 Job，導致卡在「stuck」的狀態。

解決方法：

* 先確認 gitlab-runner 是否活著：

```bash
gitlab-runner verify
```

* 如果活著但不接任務，就是罷工(暫時不確定原因)，解法是重啟 gitlab-runner：

```bash
sudo gitlab-runner restart
sudo gitlab-runner run 
```

之後如過在別的終端登入，使用 `ps aux` 能看到 gitlab-runner run 的 process 應該就 OK 了：

```bash
ps aux | grep gitlab-runner
```

---

* 如果還是不行，先確認 gitlab-runner.service 的狀況：

```bash
systemctl status gitlab-runner.service
```

> 有錯誤的話可以去看 log，也可以直接重啟。

* 重啟 gitlab-runner.service：

```bash
systemctl restart gitlab-runner.service
```

* 再次用 systemctl status 確認狀態在 running 後，重新驗證並執行 gitlab-runner：

```bash
gitlab-runner verify
gitlab-runner restart
gitlab-runner run
```

> 參考：(https://stackoverflow.com/questions/34625885/gitlab-ci-builds-remains-pending)

## 移除 runner

* 先列出 server 上註冊過的 runner：

```bash
gitlab-runner list
```
> 會看到 runner 的名稱、token、url 等資訊。

* 移除 runner：

```bash
gitlab-runner unregister --name <runner-name>
```

or 

```bash
gitlab-runner unregister --url <runner-url> --token <runner-token>
```


or

```bash
gitlab-runner verify --delete -t <runner-token> -u <runner-url>
``` 

## 手動設定 /etc/gitlab-runner/config.toml

* 停止 gitlab-runner：

```bash
sudo gitlab-runner stop
```

* 編輯 /etc/gitlab-runner/config.toml：

```bash
sudo vi /etc/gitlab-runner/config.toml
```

* 修改後，重新啟動 gitlab-runner：

```bash
sudo gitlab-runner start
```