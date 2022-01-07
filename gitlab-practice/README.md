---
title: gitlab部署全流程记录
date: 2021-02-04
---



## 术语解释：

### 1. Gitlab

GitLab是一个利用Ruby on Rails开发的开源应用程序，实现一个自托管的Git项目仓库，可通过Web界面进行访问公开的或者私人项目。
它拥有与GitHub类似的功能，能够浏览源代码，管理缺陷和注释。可以管理团队对仓库的访问，它非常易于浏览提交过的版本并提供一个文件历史库。团队成员可以利用内置的简单聊天程序（Wall）进行交流。它还提供一个代码片段收集功能可以轻松实现代码复用，便于日后有需要的时候进行查找。

### 2. Gitlab-CI

[Gitlab-CI](https://docs.gitlab.com/ce/ci/quick_start/README.html)是GitLab Continuous Integration（Gitlab持续集成）的简称。
从Gitlab的8.0版本开始，gitlab就全面集成了Gitlab-CI,并且对所有项目默认开启。
只要在项目仓库的根目录添加`.gitlab-ci.yml`文件，并且配置了Runner（运行器），那么每一次合并请求（MR）或者push都会触发CI [pipeline](https://docs.gitlab.com/ce/ci/pipelines.html)。

### 3. Gitlab-runner

[Gitlab-runner](https://docs.gitlab.com/ce/ci/runners/README.html)是`.gitlab-ci.yml`脚本的运行器，Gitlab-runner是基于Gitlab-CI的API进行构建的相互隔离的机器（或虚拟机）。GitLab Runner 不需要和Gitlab安装在同一台机器上，但是考虑到GitLab Runner的资源消耗问题和安全问题，也不建议这两者安装在同一台机器上。

Gitlab Runner分为两种，Shared runners和Specific runners。
Specific runners只能被指定的项目使用，Shared runners则可以运行所有开启` Allow shared runners`选项的项目。

### 4. Pipelines

Pipelines是定义于`.gitlab-ci.yml`中的不同阶段的不同任务。
我把[Pipelines](https://docs.gitlab.com/ce/ci/pipelines.html)理解为流水线，流水线包含有多个阶段（[stages](https://docs.gitlab.com/ce/ci/yaml/README.html#stages)），每个阶段包含有一个或多个工序（[jobs](https://docs.gitlab.com/ce/ci/yaml/README.html#jobs)），比如先购料、组装、测试、包装再上线销售，每一次push或者MR都要经过流水线之后才可以合格出厂。而`.gitlab-ci.yml`正是定义了这条流水线有哪些阶段，每个阶段要做什么事。

### 5. Badges

[徽章](https://docs.gitlab.com/ce/ci/pipelines.html#badges)，当Pipelines执行完成，会生成徽章，你可以将这些徽章加入到你的README.md文件或者你的网站。

徽章的链接形如：
`http://example.gitlab.com/namespace/project/badges/branch/build.svg`



## 部署实现

第一步：helm部署gitlab命令：

```code
helm install gitlab gitlab/  --timeout 600s --set global.hosts.domain=linjb.com 
--set certmanager-issuer.email=1576654308@qq.com --set global.hosts.gitlab.https=false 
--set global.hosts.https=false --set global.ingress.tls.enabled=false
```

第二步：在coredns中配置域名的解析为集群内地址。

第三步：登录集群：

root账号为：kubectl get secret <name>-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ; echo

第四步：登录集群后的操作

* 设置允许本机IP的流量访问
* 查看gitlab-runner是否正常
* 配置operation即k8s，用作CD

```code
kubectl apply -f gitlab-admin-service-account.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: gitlab
    namespace: kube-system
---

kubectl get secrets | grep default-token
kubectl get secret default-token-v2n9k   -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
```


## 注意

1. 集群必须不是并设置一个动态sc
2. 集群必须部署coredns，用于后续指定内部解析
3. codedns的指定解析，需要指定ingress-controller 的svc的地址，域名为gitlab.linjb.com
4. gitlab-runner有时候会报错不信任证书，此时建议采用部署命令里面的关闭https



## 使用示例

第一步：项目跟目录新建.gitlab-ci.yml文件

.gitlab-ci.yml

```yaml
stages:
    - mvn-build
    - docker-build
mvn-build:
  image: maven:latest
  stage: mvn-build
  script:
    - echo '打包开始'
    - mvn package -Dmaven.test.skip=true
    - echo '打包完成'
  only:
    - master

docker-build:
  # Official docker image.
  image: docker:latest
  stage: docker-build
  services:
    - docker:dind
  before_script:
    - docker login -u * -p * 10.20.*.*
  script:
    - docker build --pull -t 10.20.*.*/library/ecs .
    - docker push 10.20.*.*/library/ecs
  only:
    - master
```

第二步：新建Dockerfile文件

Dockerfile

```code
FROM tomcat

ADD target/*.war /usr/local/tomcat/webapps/

EXPOSE 8080
```

提交之后即可触发编译。

部分日志：

```code
[INFO] --- maven-surefire-plugin:2.22.2:test (default-test) @ resourcemanagement ---
Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/maven-surefire-common/2.22.2/maven-surefire-common-2.22.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/maven-surefire-common/2.22.2/maven-surefire-common-2.22.2.pom (11 kB at 12 kB/s)
Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-annotations/3.5.2/maven-plugin-annotations-3.5.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-annotations/3.5.2/maven-plugin-annotations-3.5.2.pom (1.6 kB at 2.1 kB/s)
Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-tools/3.5.2/maven-plugin-tools-3.5.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugin-tools/maven-plugin-tools/3.5.2/maven-plugin-tools-3.5.2.pom (15 kB at 8.3 kB/s)
Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-api/2.22.2/surefire-api-2.22.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-api/2.22.2/surefire-api-2.22.2.pom (3.5 kB at 2.9 kB/s)
Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-logger-api/2.22.2/surefire-logger-api-2.22.2.pom
Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-logger-api/2.22.2/surefire-logger-api-2.22.2.pom (2.0 kB at 2.9 kB/s)
Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/surefire/surefire-booter/2.22.2/surefire-booter-2.22.2.pom
```

## 完成示例

my-nginx

gitlab-ci-k8s-demo


