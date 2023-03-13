# gitlab
gitlab依赖[[#runner]]和[[#gitlab-ci yml]]来实现CI/CD。需要注意gitlab与runner之间的版本关系不能差太多。

## runner
### install
通过docker安装运行
```shell
docker run -d --name gitlab-runner --restart always \
     -v /srv/gitlab-runner/config:/etc/gitlab-runner \
     -v /var/run/docker.sock:/var/run/docker.sock \
     gitlab/gitlab-runner:v13.12.0
```

`/srv/gitlab-runner/config`文件夹是存放`config.toml`的地方
### register
通过docker注册
```shell
docker run -it --rm -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner:v13.12.0 register 
```
根据提示输入配置。url，token需要从gitlab对应的project中settings -> CI/CD -> Runners 中获取。

register的产物就是[[#config.toml]]文件，也可以手动编写该文件而不必register。

### config.toml
#### runners.docker.volumes
volumes配置runner运行的镜像挂载本地文件，如果需要使用`docker in docker`，需要在这里新增`/var/run/docker.sock:/var/run/docker.sock`。

#### runners.docker.pull_policy
配置runner默认的镜像拉取策略，默认是`"always"`，如果需要可以换成`"if-not-present"`.


## .gitlab-ci.yml
### image
```yaml
image: docker:dind
```
`image`标签是在executor为docker时，指定所有[[#Job]]的默认基础镜像，如果job指定了`image`关键词，则使用指定的image。

### services
```yaml
services:
  - docker:dind
  - postgresql:laster
```
为所有的job提供其他镜像的服务，类似`docker --link`。

### variables
```yaml
variables:
  MAVEN_CLI_OPTS: "-s /root/.m2/settings.xml --batch-mode"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
```
以key-value的形式指定所有容器的环境变量，这些环境变量通常都是数据不明感的。

### stages
```yaml
stages:
	- build
	- test
	- deploy
```
`stages`中定义任意数量的构建步骤，这些步骤由`job`中的`stage`关键字定义。定义好的步骤会按序执行。如果有多个Job使用同一个stage，则Job会同步执行。

常用的步骤有：
- build: 构建步骤，一般有一个叫做`compile`的job.
- test: 测试步骤。
- staging：部署到测试环境的步骤，job可以命名`deploy-to-stage`.
- production：部署到生产环境，job可以命名为`deploy-to-prod`.

### Job
```yaml
maven-build:
  stage: build
  tags:
    - '34-tam'
  image: maven3:dist
  script:
    - mvn $MAVEN_CLI_OPTS package -pl com.dist.xdata.tam:tam-union -am
    - cp tam-runtime/tam-cloud/tam-union/target/*.war /usr/share/target
    - unzip -o /usr/share/target/*.war -d /usr/share/target/tam/
    - rm -rf /usr/share/target/*.war
  only:
  - master
  cache:
    paths:
      - .m2/repository
```

Job就是CI过程中需要执行的一项工作。以上例子中,`maven-build`就是job的name，可以任意决定（不能使用关键字）。

#### stage
必填，指定该Job在哪一个步骤执行。

#### tags
指定监听该Job的Runner。

#### image
如果executor是docker，该配置指定Job的基础镜像。

#### script
至少一个，表示执行`Job`的指令。

#### only
指定触发Job的条件，可以指定`Branches`，`Tags`之类的

#### cache
指定路径下的文件会被缓存，后续相同的Job会共享一份cache。常用来缓存一些依赖文件。

cache也可以指定在外层，这样就是全Job共享的cache。

### artifacts
可以将Job的产物发布到gitlab中供下载，但是有大小和有效期的限制。

## Variables
该`Variables`与`.gitlab-ci.yml`中的类似，都是指定Job的环境变量，不过这是在gitlab的project中配置的，settings->CI/CD->Variables，该配置无权限不可阅读，相对比`.gitlab-ci.yml`中的variables安全。