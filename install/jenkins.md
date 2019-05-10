# Jenkins Configuration

* #### *Jenkins部署采取jenkins master + jenkins slave的方式部署到Kubernetes集群上。一旦在 Jenkins 中把构建节点和 job 都容器化了，迁移工作平台将变得十分简单易行。在 Kubernetes 集群中的 Jenkins 代理中运行构建是非常简单，并且降低同时构建多个任务时造成的单个jenkins代理负载。*

* #### 定制部署到Kubernetes的Jenkins镜像

  * 需要根据需要对Jenkins原始镜像进行修改重建。由于挂载外部持久化存储会改变容器里对应挂载目录的属主，造成因权限不足无法启动服务。本项目需要对挂载持久化存储的/var/jenkins_hoem目录进行赋权，否则没有写权限，Jenkins服务无法启动。（具体见印象笔记docker volume部分说明）

  * 制作思路：编写Dockerfile对官方jenkins镜像启动后使用root用户修改/var/jenkins_home目录属主，再使用jenkins用户启动jenkins服务，确保了目录权限正确、服务启动正确。

  * `FROM jenkinsci/blueocean:1.14.0`

    `USER root`

    `ENV GOSU_VERSION 1.11`

    `RUN set -eux; \`

    ​        `\`

    ​        `apk add --no-cache --virtual .gosu-deps \`

    ​                `dpkg \`

    ​                `gnupg \`

    ​        `; \`

    ​        `\`

    ​        `dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \`

    ​        `wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \`

    ​        `wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \`

    ​        `\`

    `\# verify the signature`

    ​        `export GNUPGHOME="$(mktemp -d)"; \`

    `\# for flaky keyservers, consider https://github.com/tianon/pgp-happy-eyeballs, ala https://github.com/docker-library/php/pull/666`

    ​        `gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \`

    ​        `gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \`

    ​        `command -v gpgconf && gpgconf --kill all || :; \`

    ​        `rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \`

    ​        `\`

    `\# clean up fetch dependencies`

    ​        `apk del --no-network .gosu-deps; \`

    ​        `\`

    ​        `chmod +x /usr/local/bin/gosu; \`

    `\# verify that the binary works`

    ​        `gosu --version; \`

    ​        `gosu nobody true`

    `COPY entrypoint.sh ./`

    `RUN chmod +x entrypoint.sh`

    `ENTRYPOINT ["./entrypoint.sh"]`

  * <u>entrypoint.sh文件内容如下</u>：

    `#!/bin/bash -e`

    `chown -R 1000:1000 "$JENKINS_HOME"`

    `exec gosu jenkins /bin/tini -- /usr/local/bin/jenkins.sh`

  

* #### 设置Jenkins时区取值文件

  * jenkins运行的获取时间不是使用系统的/etc/localtime文件，而是单独的/etc/timezone文件，因此需要单独设置挂载。我们采取conifgmap的方式设置和挂载此文件。确保构建时间取值是正确的。

  * <u>timezone的yaml文件如下</u>：

    `apiVersion: v1`

    `data:`

      `timeZone: Asia/Shanghai`

    `kind: ConfigMap`

    `metadata:`

      `labels:`

    ​    `cattle.io/creator: norman`

      `name: timezone`

      `namespace: cicd`

      `selfLink: /api/v1/namespaces/cicd/configmaps/timezone`

    

* #### 设置/var/jenkins\_home/ 目录挂载到网络存储，做到数据持久化

  * 在集群安装了Rook-Ceph分布式存储，供pod使用挂载持久化数据，做到支持有状态应用。
  * jenkins的home目录必须要持久化存储，以便在jenkins master的pod可以调度到Kubernetes集群里的任意node上，出现问题重新建立时不会影响服务和数据。
  * 本项目使用Ceph的rbd块存储挂载home目录

* #### 设置Jenkins集成Kubernetes凭证

  * 为了保证 Jenkins 能够访问 K8s 集群的资源，首先需要创建Kubernetes凭证。

  * 有两种方式可以创建凭证。

  * <u>第一种，按照Kubernetes的ServiceAccount建立凭证</u>。yaml文件如下（jenkins-rbac.yaml）：

    `apiVersion: v1`
    `kind: ServiceAccount`
    `metadata:`
      `labels:`
        `k8s-app: jenkins`
      `name: jenkins-ci`
      `namespace: cicd`

    `---`

    `apiVersion: rbac.authorization.k8s.io/v1beta1`
    `kind: ClusterRoleBinding`
    `metadata:`
      `name: jenkins-ci`
      `labels:`
        `k8s-app: jenkins`
    `roleRef:`
      `apiGroup: rbac.authorization.k8s.io`
      `kind: ClusterRole`
      `name: cluster-admin`
    `subjects:`

    `- kind: ServiceAccount`
        `name: jenkins-ci`
        `namespace: cicd`

  * <u>第二种设置方式（本人未实际验证）</u>：

    1. 进入 Jenkins 的 UI 界面，点击左边导航栏里的凭据链接

    2. 点击 Stores scoped to Jenkins 列表下 global 中的 Add credentials (将鼠标悬停在链接旁边即可看到箭头)

    3. 点击添加凭证

    4. 添加类型 Kubernetes Service Account

    5. 将范围设置为全局

    6. 点击 OK 按钮

    这样 Jenkins 就可以使用这个凭据去访问 K8s 的资源。

    

* #### Jenkins 服务部署pod的yaml文件如下：

  * [jenkins.yaml](install/jenkins_yaml.md) 

* #### 安装本项目使用的Jenkins plugins包

  * Jenkins部署是在内网部署，无法连接互联网，因此使用的jenkins plugins需要先到官网下载后再安装，并且，由于各个包之间有可能有依赖关系，需要按照顺序进行安装。目前使用的插件包有如下几项：

    ###### <u>Jenkins安装的插件和顺序</u>：

    ###### Maven集成构建扩展包（本项目未使用）

    JDK Tool 

    Command Agent Launcher 

    Javadoc 

    bouncycastle API 

    Maven Integration

    ###### Docker集成插件（本项目未使用）

    SSH Slaves Success

    Docker API Success

    Docker Success

    ###### Kubernetes plugin集成插件

    Kubernetes Credentials Success

    Kubernetes Success

    Kubernetes Cli Success

    ###### Gitlab插件

    GitLab 

    Gitlab Authentication 

    WMI Windows Agents 

    ruby-runtime 

    Gitlab Hook 

    ###### Role插件Role-based Authorization

    Ant 

    jQuery 

    PAM Authentication 

    LDAP 

    External Monitor Job Type 

    jQuery UI 

    Role-based Authorization Strategy 

    ###### Pipeline 集成插件

    Pipeline: REST API 

    JavaScript GUI Lib: Handlebars bundle 

    JavaScript GUI Lib: Moment.js bundle 

    Pipeline: Stage View 

    ###### SonarQube插件

    Sonargraph Integration 

    SonarQube Scanner 

    Quality Gates 

    Sonargraph 

    ###### email扩展插件

    Email Extension

* #### 集成Kubernetes设置

  * Jenkins服务启动后配置好登录用户密码后，下一步就是在 Jenkins 中设置集成kubernetes云的配置：

    1. 进入 Jenkins UI 界面，点击 系统管理 → 系统设置；

    2. 进入管理界面后查找 『云』（一般在下面），然后点击 『新增一个云』，选择 kubernetes 类型；

    3. 如下这些是必填的参数：

       Name 自定义， 默认是 kubernetes；

       Kubernetes URL https://kubernetes.default -一般是从 service account 自动配置；

       Kubernetes Namespace 一般是 default  除非要在一个特殊的命名空间 ，否则不要动；

       Credentials 如果是第一种方式设置的凭证则空着不配置，如果是第二种则选择上一步创建的凭证；

       Jenkins URL http://<your_jenkins_hostname>:8080；

       Jenkins tunnel <your_jenkins_hostname>:5555 - 用来和 Jenkins 启动的 agent 进行交互的端口；

       ![1557388941452](D:\gitbook\cicd_project\install\1557388941452.png)

    4. 点击test connection按钮，验证连接。

  * 现在已经可以通过定义一些 pod 实现Jenkins master 访问 K8s 集群了。我们可以在这个 pod 中执行一些构建工作。每一个 Jenkins 节点都是作为 K8s pod 来启动的。这个 pod 里面经常会包含一个默认的 JNLP 容器，还有一些pod 模板中定义的容器。现在有至少两种方法来定义pod template。

  * *通过 Jenkins UI 配置一个 pod template* 和 *用 jenkinsfile 来实现相同的功能*

* #### eos项目构建使用的镜像准备

  * ###### Jenkins使用pipeline调度在Kubernetes创建jenkins slave的pod，pod里运行三个容器开展构建工作，容器镜像准备如下：

    1. jenkins slave镜像jnlp-slave；

       使用官方镜像，不做修改。

    2. 制作用于maven打包构建的镜像maven：

       eos项目maven使用本地部署的私有仓库Nexus，因此需要把官方maven镜像加入本地Nexus仓库连接配置，定制适合本地使用的maven镜像。制作镜像的Dockerfile内容如下：

       `FROM reg.ecs.local/eos/maven:3-jdk-8`

       `COPY settings.xml /root/.m2/`

       `ENTRYPOINT ["/usr/local/bin/mvn-entrypoint.sh"]`

       `CMD ["mvn"]`

    3. 制作Docker镜像，用于eos微服务构建镜像和发布服务到Kubernetes集群：

       需要 Docker 客户端使用 Docker API 进行docker镜像的构建和发布到本地私有harbo镜像仓库，另外Kubernetes集群使用Rancher进行部署和管理的，所以优选集成Rancher CLI进行操作Kubernetes发布微服务，避免出现未知问题。

       我们把本地配置好的harbor凭证、Rancher CLI凭证和客户端、kubectl文件集成到Docker官方镜像里，用于构建时使用。

       Dockerfile文件内容如下：

       `FROM docker:18.09`

       `ADD certs.d/reg.ecs.local/ca.crt /etc/docker/certs.d/reg.ecs.local/ca.crt`

       `ADD rancher-v2.2.0/rancher /etc/rancher-v2.2.0/rancher`

       `ADD cli2.json /etc/rancher-v2.2.0/cli2.json`

       `COPY kubectl /usr/local/bin/kubectl`

       `RUN chmod +x /usr/local/bin/kubectl`

       `ENTRYPOINT ["docker-entrypoint.sh"]`

       `CMD ["sh"]`

* 

* #### Jenkins plus : kubernets\_plugin

​          编写yaml文件建立ServieAccount用于jenkins访问kubernetes集群: jenkins-rbac.yaml

* 流水线设计
* * 编译和单测——模块测试——系统测试——预上线——生产/灰度——生产/全量
    | 编译和单测 | 模块测试 | 系统测试 | 预上线 | 生产/灰度 | 生产/全量 |
    | :--- | :--- | :--- | :--- | :--- | :--- |
    | 多个stage是串行执行 |  | 执行作业（job） | 质量门（通过标准）人工干预 | 决策点，人工干预 |  |
    | 前一个stage成功后自动触发 |  | stage里的多个job串/并执行 | pass/fail判定标准 | 配置人工决策，一键Approve后流水线自动执行 |  |
    | 也可以设计手工触发 |  | stage里的多个job定时执行 | 测试通过率&gt;x% | 也可以配置前序stage成功后自动触发执行 |  |
    |  |  | 自动判断或人工标记pass/fail | 代码覆盖率&gt;x% |  |  |
    |  |  | job fail&gt;stage fail |  |  |  |

* #### 流水线部署设计：
* #### 提交阶段——集成测试阶段——PRD阶段
* #### \* \#\#\#\#\# 代码设定develop、master两个主分支
* #### \* \#\#\#\#\# 开发人员只允许提交develop分支，项目主管负责合并代码到master分支，基于master分支进行创建tag、集成测试、部署。
* #### 



