# Argo CD

https://www.hwchiu.com/docs/2020/iThome_Challenge/cicd-14

https://medium.com/ikala-tech/argocd-%E7%9A%84%E8%A6%8F%E5%8A%83%E8%88%87%E5%AF%A6%E8%B8%90-93ed88303585

https://huanlin.cc/docs/devops/ci-cd/argo-cd/

## 先搞定

* 兩個 repo：
  * 程式碼：ci 負責打包 image、更新 k8s yaml image tag
  * k8s yaml：給 argo cd 用。

* https://github.com/mkaraminejad/cicd_pipeline/blob/main/2-AgroCD/code-repo/.gitlab-ci.yml

* code-repo

* k8s-repo