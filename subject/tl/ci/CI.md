# CI

## 发布流程

联调环境 > FAT 环境 > UAT 环境 > PRE > PRO 环境

### 联调环境

可以选择采用fat还是uat的配置启动联调环境，服务通过tag路由。

### FAT

冒烟测试

### UAT

测试

### PRE

测试

### PRO
 
生产环境

## 选型

### GitLab Runner

```
image: maven:3.6-openjdk-11

stages:
  - test
  - notify
  - deploy

test-sonar:
  stage: test
  only:
    - dev
    - master
  script:
		- mvn clean test sonar:sonar -Dsonar.projectKey=${project.name} -Dsonar.host.url=${sonarqube} -Dsonar.login=${sonarqube.token}
	artifacts:
		name: "coverage_report"
    when: on_success
    paths:
    	- api-market-admin/target/coverage-reports/jacoco.csv
    	- api-market-common/target/coverage-reports/jacoco.csv
      - api-market-gateway/target/coverage-reports/jacoco.csv
      - api-market-stats/target/coverage-reports/jacoco.csv
      
success-report:
  stage: notify
  only:
    - dev
    - master
  dependencies:
    - test-sonar
  script:
    - echo "=============== Report ==============="
    - bash ./gitlab-ci-notification.sh "on_success" ${CI_COMMIT_REF_SLUG} ${CI_COMMIT_SHA:0:8}
  when: on_success
  
deploy:
  stage: deploy
  only:
    - master
  script:
    - echo "=============== 提交测试环境上线单 ==============="
    - bash ./gitlab-ci-deploy.sh ${GITLAB_USER_EMAIL} ${CI_COMMIT_REF_SLUG} ${CI_COMMIT_SHA:0:7} test
  when: on_success
```

#### 步骤

- test 单元测试:

  每次代码提交自动执行单元测试，上报测试覆盖率；dev，master分支下执行。

- notify 结果通知:

  每次代码提交触发CI任务，任务结果通知任务；dev,master分支下执行。

- deploy 镜像构建&推送:

  构建镜像推送私有镜像仓库，依赖test任务结果（test成功），merge request合并请求期间执行；master分支下执行。

#### 分支构建

### JIB

> 最小镜像

#### 基础镜像

##### JDK 11

```xml
adoptopenjdk/openjdk11:jre-11.0.10_9-alpine@sha256:a97add2e5cc7588fb7bf67a9be4815dc0acabe914af123fbac4e028a0c3140bf
```

##### JDK 17

> 注意目前 maven不支持jdk16，可以采用gradle 7.0+

```gradle
adoptopenjdk/openjdk16:x86_64-alpine-jre-16.0.1_9
```

#### Maven Settings

```xml
<pluginManagement>
    <plugins>
        <plugin>
            <groupId>com.google.cloud.tools</groupId>
            <artifactId>jib-maven-plugin</artifactId>
            <version>3.3.2</version>
            <configuration>
                <to>
                    <image>${harbor}/${project.artifactId}:${git.commit.id.abbrev}</image>
                </to>
                <from>
                    <image>
                        adoptopenjdk/openjdk11:jre-11.0.10_9-alpine@sha256:a97add2e5cc7588fb7bf67a9be4815dc0acabe914af123fbac4e028a0c3140bf
                    </image>
                </from>
                <extraDirectories>
                    <paths>target/${project.artifactId}</paths> <!-- Copies files from-->
                </extraDirectories>
                <allowInsecureRegistries>true</allowInsecureRegistries>
                <container>
                    <creationTime>USE_CURRENT_TIMESTAMP</creationTime>
                </container>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>build-helper-maven-plugin</artifactId>
        </plugin>
    </plugins>
</pluginManagement>

mvn jib:build -Djib.to.auth.username=${JIB_AUTH_NAME} \
    -Djib.to.auth.password=${JIB_AUTH_PWD} \
```

#### Shell

> 不用为每个服务单独配置，标准化

```shell
ssh init.script

export JAVA_MODEL_OPS="--add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.desktop/java.awt.font=ALL-UNNAMED"

export JIB_AUTH_NAME=xxx
export JIB_AUTH_PWD=xxx
export tag=$(date '+%Y%m%d%H%M%S')

mvn compile com.google.cloud.tools:jib-maven-plugin:3.3.2:build \
    -Dimage=${harbor}/${APP_KEY}:${tag} \
    -Djib.to.auth.username=${JIB_AUTH_NAME} \
    -Djib.to.auth.password=${JIB_AUTH_PWD} \
    -Djib.extraDirectories.paths=target/${project.artifactId} \
    -Djib.allowInsecureRegistries=true \
    -Djib.container.creationTime=USE_CURRENT_TIMESTAMP \
    -Djib.container.ports=8080 \
    -Djib.container.format=Docker \
    -Djib.container.entrypoint=$JAVA_CMD \
```

#### Gradle

```groovy

jib {
    from {
        image = "adoptopenjdk/openjdk16:x86_64-alpine-jre-16.0.1_9@sha256:485f58b1e084af00492c3c7704c5b0e223622f8017cbedca8c885bf2136e23bf"
    }
    to {
        image = "${harbor.domain}/customize-${project.name}"
        auth {
            username = ""
            password = ""
        }
        project.afterEvaluate {
            tags = ["${-> project.ext.gitProps['git.commit.id.abbrev']}"]
        }
    }
    allowInsecureRegistries = true
    container({
        creationTime = "USE_CURRENT_TIMESTAMP"
        ports = ["8080"]
        format = "Docker"
        entrypoint = [
                'java',
                '-Duser.timezone=Asia/Shanghai',
                '-Dfile.encoding=utf-8',
                '-XX:+UnlockExperimentalVMOptions',
                '-XX:+UseZGC',
                '-Xms2048m',
                '-Xmx2048m',
                '--add-opens',
                'java.base/java.util=ALL-UNNAMED',
                '--add-opens',
                'java.base/java.lang=ALL-UNNAMED',
                '--add-opens',
                'java.desktop/java.awt.font=ALL-UNNAMED',
                '-cp',
                '/app/resources:/app/classes:/app/libs/*',
                'com.mobtech.fin.msxf.MSXFApplication',
                '>/dev/null',
                '2>&1'
        ]
    })
}

./gradlew jib
```

### 测试覆盖率

#### SonarQube

单元测试覆盖，代码静态分析。技术指标清晰可量化。

#### JacocoAgent

单元测试覆盖

### 配置中心

#### Apollo

> 放弃不同环境分开构建的老路，所有环境一个镜像，采用配置中心统一管理不同环境的配置

### Q&A

Q: 为什么不是k8s ConfigMap?
A: apollo能够集中管理应用在不同环境、不同集群的配置，修改后实时推送到应用端，并且具备规范的权限、流程治理等特性。

Q: 最小镜像怎么debug?

A:

#### telnet, tcp dump etc

```shell
kubectl debug --kubeconfig=config-exercise/config api-market-stats-5c9df7bc7b-6fx9s  -n mobfin

telnet 10.21.xx.xx [port]
```

#### Arthas Agent

```shell
java -javaagent:/workspace/app/jarlibs/arthas/arthas-agent.jar -jar math-game.jar
```

注意：agent方式通常只允许在测试环境生效

#### JVM dump

```shell
kubectl debug --kubeconfig=config-exercise/config api-market-stats-5c9df7bc7b-6fx9s  -n ${namespace} --image openjdk:11.0.11-jdk-slim-buster

jps
# 1 ApiMarketStatisApplication

jstack -l 1
```

#### Alibaba Arthas

```shell
kubectl debug --kubeconfig=config-exercise/config-mob api-market-stats-5c9df7bc7b-6fx9s  -n mobfin --image hengyunabc/arthas:latest /bin/sh

# 注意当前版本的kubectl debug 需要
chroot /proc/1/root
```

Issue: 

.  https://github.com/aylei/kubectl-debug/issues/81

#### 其他

> 注意：需要容器包含bash，wget。即容器不是最小镜像

```shell
kubectl exec -it ${pod} --container ${containerId} -- /bin/bash -c "wget https://arthas.aliyun.com/arthas-boot.jar && java -jar arthas-boot.jar"
```

## 附录

https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html
