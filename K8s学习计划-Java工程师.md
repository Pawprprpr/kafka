# K8s学习计划 - 针对Java工程师和CI/CD开发

## 学习目标
- 快速掌握Kubernetes核心概念
- 学会Java应用在K8s中的部署和配置
- 理解K8s与CI/CD流程的集成
- 掌握华为云CodeArts Build与K8s的集成

## 第一阶段：核心概念理解（1-2天）

### 1. K8s基础架构
- **Master节点组件**：
  - API Server：集群的入口，处理所有API请求
  - etcd：分布式键值存储，保存集群状态
  - Scheduler：调度器，决定Pod运行在哪个节点
  - Controller Manager：控制器管理器，维护集群状态

- **Worker节点组件**：
  - kubelet：节点代理，管理Pod生命周期
  - kube-proxy：网络代理，实现Service负载均衡
  - 容器运行时：Docker、containerd等

### 2. 核心资源对象

#### Pod
- 最小部署单元，包含一个或多个容器
- 共享网络和存储
- 生命周期管理

#### Service
- 服务发现和负载均衡
- 类型：ClusterIP、NodePort、LoadBalancer
- 标签选择器匹配Pod

#### Deployment
- 应用部署和滚动更新
- 副本管理
- 回滚机制

#### ConfigMap/Secret
- ConfigMap：非敏感配置管理
- Secret：敏感信息（数据库密码、API密钥）
- 环境变量注入

#### Namespace
- 资源隔离
- 多租户支持
- 资源配额管理

### 3. 网络和存储
- **Service网络**：ClusterIP、NodePort、LoadBalancer
- **Ingress**：外部访问和路由规则
- **PersistentVolume**：数据持久化
- **StorageClass**：动态存储分配

## 第二阶段：Java应用部署实践（2-3天）

### 1. Java应用容器化

#### Dockerfile示例
```dockerfile
# 多阶段构建优化镜像大小
FROM maven:3.8.4-openjdk-11 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=build /app/target/myapp.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 2. K8s部署文件编写

#### Deployment配置
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  labels:
    app: java-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
    spec:
      containers:
      - name: java-app
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "k8s"
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.url
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: database-password
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### Service配置
```yaml
apiVersion: v1
kind: Service
metadata:
  name: java-app-service
spec:
  selector:
    app: java-app
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

#### ConfigMap配置
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.url: "jdbc:mysql://mysql-service:3306/myapp"
  logging.level: "INFO"
  server.port: "8080"
```

#### Secret配置
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  database-password: bXlwYXNzd29yZA==  # base64编码
  api-key: bXlhcGlrZXk=  # base64编码
```

### 3. Ingress配置
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: java-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-app-service
            port:
              number: 80
```

## 第三阶段：CI/CD集成（2-3天）

### 1. 华为云CodeArts Build集成

#### 构建配置
```yaml
# .codearts-build.yml
version: '2.0'
stages:
  - build:
      maven:
        goals: ["clean", "package", "-DskipTests"]
        settings: "settings.xml"
        
  - test:
      maven:
        goals: ["test"]
        
  - docker:
      build:
        dockerfile: Dockerfile
        image: swr.cn-north-4.myhuaweicloud.com/myproject/myapp:${BUILD_NUMBER}
        registry: swr.cn-north-4.myhuaweicloud.com
        credentials: "huawei-cloud-registry"
        
  - security:
      image_scan:
        image: swr.cn-north-4.myhuaweicloud.com/myproject/myapp:${BUILD_NUMBER}
        
  - deploy:
      kubernetes:
        manifests:
          - k8s/deployment.yaml
          - k8s/service.yaml
          - k8s/configmap.yaml
          - k8s/secret.yaml
        namespace: production
        cluster: "my-k8s-cluster"
        credentials: "huawei-cloud-k8s"
```

### 2. 高级部署策略

#### 蓝绿部署
```yaml
# 使用两个Deployment实现蓝绿部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: java-app
      version: blue
  template:
    metadata:
      labels:
        app: java-app
        version: blue
    spec:
      containers:
      - name: java-app
        image: myapp:v1.0
        # ... 其他配置
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-green
spec:
  replicas: 0  # 初始为0
  selector:
    matchLabels:
      app: java-app
      version: green
  template:
    metadata:
      labels:
        app: java-app
        version: green
    spec:
      containers:
      - name: java-app
        image: myapp:v2.0
        # ... 其他配置
```

#### 金丝雀发布
```yaml
# 使用Istio实现金丝雀发布
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: java-app-vs
spec:
  hosts:
  - java-app-service
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: java-app-service
        subset: v2
  - route:
    - destination:
        host: java-app-service
        subset: v1
      weight: 90
    - destination:
        host: java-app-service
        subset: v2
      weight: 10
```

### 3. 环境管理
```yaml
# 不同环境的配置
# k8s/dev/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: dev
spec:
  replicas: 1
  # ... 开发环境配置

# k8s/staging/deployment.yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: staging
spec:
  replicas: 2
  # ... 测试环境配置

# k8s/prod/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: production
spec:
  replicas: 5
  # ... 生产环境配置
```

## 第四阶段：监控和运维（1-2天）

### 1. 应用监控

#### Prometheus配置
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'java-app'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: java-app-service
```

#### Grafana仪表板
- JVM指标监控
- 应用性能指标
- 业务指标监控

### 2. 日志管理

#### ELK Stack配置
```yaml
# Filebeat配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*java-app*.log
    output.elasticsearch:
      hosts: ["elasticsearch:9200"]
```

### 3. 故障排查

#### 常用命令
```bash
# 查看Pod状态
kubectl get pods -n production

# 查看Pod日志
kubectl logs -f deployment/java-app -n production

# 进入Pod调试
kubectl exec -it <pod-name> -- /bin/bash

# 查看Service
kubectl get svc -n production

# 查看Ingress
kubectl get ingress -n production

# 查看事件
kubectl get events -n production --sort-by='.lastTimestamp'

# 查看资源使用情况
kubectl top pods -n production
kubectl top nodes
```

## 第五阶段：Jenkins CI/CD学习（2-3天）

### 1. Jenkins基础概念

#### Jenkins是什么
- **持续集成/持续部署（CI/CD）工具**：自动化构建、测试、部署流程
- **开源项目**：基于Java开发，插件化架构
- **核心功能**：
  - 自动化构建和测试
  - 代码质量检查
  - 自动化部署
  - 流水线编排

#### Jenkins架构
- **Master节点**：调度任务、管理配置
- **Agent节点**：执行构建任务（可选）
- **插件系统**：扩展功能
- **Pipeline**：定义构建流程

### 2. 安装和配置（基于JDK 17）

#### 环境要求
- **JDK 17 LTS**：Jenkins LTS版本支持JDK 17
- **系统要求**：至少2GB RAM，20GB磁盘空间
- **操作系统**：Linux、Windows、macOS

#### Docker方式安装（推荐）
```dockerfile
# Dockerfile
FROM jenkins/jenkins:lts-jdk17
USER root
RUN apt-get update && apt-get install -y git maven docker.io
USER jenkins
```

```bash
# 运行Jenkins容器
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk17
```

#### 系统级安装
```bash
# Ubuntu/Debian
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins

# 配置JDK 17
sudo update-alternatives --config java
```

#### 初始配置
1. **访问Jenkins**：`http://localhost:8080`
2. **获取初始密码**：
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   # 或
   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```
3. **安装推荐插件**：Git、Pipeline、Docker、Maven等
4. **创建管理员账户**

### 3. Jenkins基本使用

#### 创建第一个Job（自由风格项目）

**步骤1：新建Item**
- 选择"Freestyle project"
- 输入项目名称

**步骤2：源码管理**
```
Repository URL: https://github.com/yourusername/myapp.git
Credentials: 添加Git凭证
Branch: */main
```

**步骤3：构建触发器**
- 定时构建：`H 2 * * *` (每天凌晨2点)
- 轮询SCM：`H/5 * * * *` (每5分钟检查)
- 构建其他项目后触发
- GitHub/GitLab Webhook触发

**步骤4：构建环境**
- 设置环境变量
- 删除工作空间
- 使用工具位置（JDK、Maven、Docker）

**步骤5：构建步骤**
```bash
# Maven构建
mvn clean package -DskipTests

# 单元测试
mvn test

# 代码质量检查
mvn sonar:sonar
```

**步骤6：构建后操作**
- 归档构建产物
- 发布测试报告
- 发送邮件通知
- 部署到服务器

#### 全局工具配置

**JDK配置**
- 路径：`系统管理` → `全局工具配置`
- JDK 17路径：`/usr/lib/jvm/java-17-openjdk-amd64`
- 名称：`JDK-17`

**Maven配置**
- Maven安装路径：`/usr/share/maven` 或自动安装
- 版本：Maven 3.8+

**Docker配置**
- Docker安装路径：`/usr/bin/docker`
- 确保Jenkins用户有Docker权限

### 4. Pipeline编写

#### Pipeline概念
- **Declarative Pipeline**：声明式语法（推荐）
- **Scripted Pipeline**：脚本式语法（更灵活）
- **Jenkinsfile**：将Pipeline代码化

#### 基础Pipeline示例
```groovy
// Jenkinsfile
pipeline {
    agent any
    
    tools {
        maven 'Maven-3.8'
        jdk 'JDK-17'
    }
    
    environment {
        APP_NAME = 'my-java-app'
        DOCKER_REGISTRY = 'your-registry.com'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('检出代码') {
            steps {
                checkout scm
            }
        }
        
        stage('编译构建') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('单元测试') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('代码质量检查') {
            steps {
                script {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('构建Docker镜像') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('推送镜像') {
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}").push()
                    }
                }
            }
        }
        
        stage('部署到K8s') {
            steps {
                sh '''
                    kubectl set image deployment/my-app \
                    my-app=${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                    -n production
                '''
            }
        }
    }
    
    post {
        success {
            echo '构建成功！'
            // 发送成功通知
        }
        failure {
            echo '构建失败！'
            // 发送失败通知
        }
        always {
            // 清理工作空间
            cleanWs()
        }
    }
}
```

#### 高级Pipeline特性

**并行构建**
```groovy
stage('并行测试') {
    parallel {
        stage('单元测试') {
            steps {
                sh 'mvn test'
            }
        }
        stage('集成测试') {
            steps {
                sh 'mvn verify -Pintegration-tests'
            }
        }
    }
}
```

**条件构建**
```groovy
stage('条件部署') {
    when {
        branch 'main'
    }
    steps {
        sh 'kubectl apply -f k8s/'
    }
}
```

**参数化构建**
```groovy
parameters {
    choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: '部署环境')
    string(name: 'VERSION', defaultValue: '1.0.0', description: '版本号')
}

stage('部署') {
    steps {
        sh "kubectl set image deployment/my-app my-app=my-app:${params.VERSION} -n ${params.ENVIRONMENT}"
    }
}
```

### 5. Java项目集成实践

#### Maven项目配置

**pom.xml配置**
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <source>17</source>
                <target>17</target>
            </configuration>
        </plugin>
        
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.8</version>
            <executions>
                <execution>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <execution>
                    <id>report</id>
                    <phase>test</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

#### Spring Boot项目Pipeline
```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven-3.8'
        jdk 'JDK-17'
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
                sh 'mvn jacoco:report'
            }
            post {
                always {
                    publishHTML([
                        reportDir: 'target/site/jacoco',
                        reportFiles: 'index.html',
                        reportName: '覆盖率报告'
                    ])
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    def image = docker.build("myapp:${BUILD_NUMBER}")
                    image.push()
                }
            }
        }
        
        stage('Deploy') {
            steps {
                sh '''
                    kubectl set image deployment/springboot-app \
                    springboot-app=myapp:${BUILD_NUMBER} -n default
                '''
            }
        }
    }
}
```

### 6. 常见问题和排查指南

#### 问题分类和诊断

**1. Jenkins无法启动或启动慢**

**症状**：
- 访问 `http://localhost:8080` 无响应
- 启动日志显示错误

**排查步骤**：
```bash
# 检查Jenkins进程
ps aux | grep jenkins

# 查看Jenkins日志
tail -f /var/log/jenkins/jenkins.log
# 或Docker方式
docker logs -f jenkins

# 检查端口占用
netstat -tulpn | grep 8080

# 检查磁盘空间
df -h

# 检查内存使用
free -m
```

**可能原因和解决**：
- **磁盘空间不足**：清理工作空间和构建历史
- **内存不足**：增加JVM内存 `-Xmx2g`
- **JDK版本不兼容**：确认使用JDK 17
- **配置文件损坏**：检查 `config.xml`

**2. 构建失败：找不到JDK或Maven**

**症状**：
```
ERROR: JAVA_HOME is not set
或
mvn: command not found
```

**排查步骤**：
```bash
# 检查Java版本
java -version

# 检查Maven
mvn -version

# 在Jenkins中配置
# 系统管理 → 全局工具配置 → JDK/Maven
```

**解决方案**：
- 在Jenkins全局工具配置中添加JDK 17路径
- 设置环境变量 `JAVA_HOME`
- 使用自动安装选项

**3. Git检出失败**

**症状**：
```
ERROR: Error fetching remote repo 'origin'
或
Authentication failed
```

**排查步骤**：
```bash
# 测试Git连接
git clone https://github.com/user/repo.git

# 检查SSH密钥
ssh -T git@github.com

# 在Jenkins中添加凭证
# 凭据 → 添加凭据 → Username with password 或 SSH Username with private key
```

**解决方案**：
- 配置Git凭证（用户名密码或SSH密钥）
- 检查网络连接和防火墙
- 验证仓库URL是否正确

**4. Maven构建失败**

**症状**：
```
[ERROR] Failed to execute goal ...
或
Could not resolve dependencies
```

**排查步骤**：
```bash
# 本地测试构建
mvn clean install

# 检查Maven settings.xml
cat ~/.m2/settings.xml

# 查看Maven日志详细输出
mvn clean package -X
```

**可能原因**：
- **依赖下载失败**：配置Maven镜像源
- **JDK版本不匹配**：检查pom.xml中Java版本
- **内存不足**：增加Maven内存 `export MAVEN_OPTS="-Xmx2g"`

**解决方案**：
```xml
<!-- settings.xml 配置阿里云镜像 -->
<mirrors>
    <mirror>
        <id>aliyun</id>
        <mirrorOf>central</mirrorOf>
        <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
</mirrors>
```

**5. Docker构建失败**

**症状**：
```
Cannot connect to the Docker daemon
或
permission denied
```

**排查步骤**：
```bash
# 检查Docker服务
sudo systemctl status docker

# 检查Jenkins用户权限
groups jenkins

# 测试Docker命令
docker ps
```

**解决方案**：
```bash
# 添加Jenkins用户到docker组
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# 或使用Docker socket挂载（容器方式）
docker run -v /var/run/docker.sock:/var/run/docker.sock ...
```

**6. Pipeline语法错误**

**症状**：
```
groovy.lang.MissingPropertyException
或
No such DSL method
```

**排查步骤**：
- 使用Pipeline语法检查工具
- 检查Jenkinsfile语法：`http://your-jenkins/pipeline-syntax/`
- 查看Groovy语法文档

**常见错误**：
```groovy
// 错误：缺少agent
pipeline {
    stages { ... }
}

// 正确
pipeline {
    agent any
    stages { ... }
}
```

**7. 构建缓慢**

**症状**：
- 构建时间过长
- 频繁超时

**优化建议**：
- **使用构建缓存**：Maven本地仓库、Docker层缓存
- **并行构建**：使用parallel步骤
- **选择合适节点**：使用性能更好的Agent
- **优化构建步骤**：跳过不必要的测试和检查

```groovy
// 使用Docker缓存
stage('Build') {
    steps {
        sh '''
            docker build \
            --cache-from myapp:latest \
            -t myapp:${BUILD_NUMBER} .
        '''
    }
}
```

**8. 权限问题**

**症状**：
```
Permission denied
或
Access denied
```

**排查**：
- 检查Jenkins用户权限
- 配置角色权限（Role-Based Strategy插件）
- 检查文件系统权限

#### 日志查看技巧

**Jenkins日志位置**：
```bash
# 系统日志
/var/log/jenkins/jenkins.log
或
/var/jenkins_home/logs/

# 构建日志
查看具体构建的控制台输出
```

**有用的命令**：
```bash
# 实时查看日志
tail -f /var/log/jenkins/jenkins.log

# 搜索错误
grep -i error /var/log/jenkins/jenkins.log

# 查看最近100行
tail -n 100 /var/log/jenkins/jenkins.log
```

### 7. 最佳实践

#### 安全实践
- **启用认证**：配置用户权限
- **定期更新**：保持Jenkins和插件最新
- **凭证管理**：使用Jenkins凭证系统，不要硬编码
- **最小权限原则**：只授予必要权限

#### 性能优化
- **使用Agent节点**：分散构建负载
- **清理策略**：定期清理构建历史和工件
- **优化Pipeline**：减少不必要的步骤
- **使用缓存**：Maven仓库、Docker镜像缓存

#### 代码管理
- **Jenkinsfile版本控制**：提交到Git仓库
- **Pipeline共享库**：复用通用步骤
- **配置即代码（JCasC）**：用YAML管理配置

#### 监控和通知
- **构建状态通知**：邮件、企业微信、钉钉
- **构建监控**：使用Prometheus插件
- **构建报表**：测试覆盖率、代码质量报告

### 8. 推荐的Jenkins插件

#### 核心插件
- **Pipeline**：流水线支持
- **Git**：Git集成
- **Docker Pipeline**：Docker构建
- **Maven Integration**：Maven构建

#### 代码质量插件
- **SonarQube Scanner**：代码质量检查
- **JaCoCo**：代码覆盖率
- **Warnings Next Generation**：代码警告分析

#### 部署插件
- **Kubernetes**：K8s部署
- **SSH Pipeline Steps**：SSH部署
- **Deploy to container**：容器部署

#### 通知插件
- **Email Extension**：邮件通知
- **Slack Notification**：Slack通知
- **钉钉/企业微信通知**：国内团队协作

### 9. 学习资源推荐

#### 官方文档
- [Jenkins官方文档](https://www.jenkins.io/doc/)
- [Jenkins Pipeline语法](https://www.jenkins.io/doc/book/pipeline/)
- [Jenkins插件中心](https://plugins.jenkins.io/)

#### 实践环境
- **本地Docker环境**：快速搭建测试环境
- **云Jenkins服务**：Jenkins Cloud、GitLab CI
- **企业版Jenkins**：Jenkins Enterprise（支持）

#### 参考书籍和教程
- Jenkins权威指南
- Jenkins持续集成实战
- 官方Pipeline示例仓库

## 实践项目结构

```
my-k8s-java-app/
├── src/
│   └── main/
│       ├── java/
│       └── resources/
│           └── application-k8s.yml
├── Dockerfile
├── pom.xml
├── .codearts-build.yml
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── secret.yaml
│   ├── overlays/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   └── monitoring/
│       ├── prometheus.yaml
│       └── grafana.yaml
└── scripts/
    ├── deploy.sh
    └── rollback.sh
```

## 学习资源推荐

### 官方文档
- [Kubernetes官方文档](https://kubernetes.io/docs/)
- [华为云CCE文档](https://support.huaweicloud.com/cce/)
- [华为云CodeArts Build文档](https://support.huaweicloud.com/codeartsbuild/)
- [Jenkins官方文档](https://www.jenkins.io/doc/)
- [Jenkins Pipeline语法](https://www.jenkins.io/doc/book/pipeline/)

### 实践环境
- **本地环境**：minikube、kind、k3s
- **云环境**：华为云CCE、阿里云ACK、腾讯云TKE

### 推荐工具
- **kubectl**：K8s命令行工具
- **k9s**：终端UI工具
- **Lens**：桌面GUI工具
- **Helm**：包管理工具

## 学习时间安排

| 阶段 | 时间 | 重点内容 |
|------|------|----------|
| 第1阶段 | 1-2天 | 核心概念理解 |
| 第2阶段 | 2-3天 | Java应用部署实践 |
| 第3阶段 | 2-3天 | CI/CD集成 |
| 第4阶段 | 1-2天 | 监控和运维 |
| 第5阶段 | 2-3天 | Jenkins CI/CD学习 |
| **总计** | **8-13天** | **系统性学习** |

## 学习检查点

### 第1阶段检查点
- [ ] 理解Pod、Service、Deployment概念
- [ ] 能够编写基本的YAML文件
- [ ] 了解K8s网络和存储模型

### 第2阶段检查点
- [ ] 能够容器化Java应用
- [ ] 编写完整的K8s部署文件
- [ ] 成功部署应用到K8s集群

### 第3阶段检查点
- [ ] 配置CI/CD流水线
- [ ] 实现自动化部署
- [ ] 掌握部署策略

### 第4阶段检查点
- [ ] 配置监控和日志
- [ ] 能够排查常见问题
- [ ] 掌握运维最佳实践

### 第5阶段检查点
- [ ] 成功安装和配置Jenkins（基于JDK 17）
- [ ] 能够创建和配置基本Job
- [ ] 掌握Pipeline编写（Jenkinsfile）
- [ ] 完成Java项目的CI/CD流水线配置
- [ ] 能够排查常见Jenkins问题

## 下一步行动

1. **立即开始**：搭建本地K8s环境（minikube）
2. **实践项目**：选择一个Spring Boot项目进行容器化
3. **Jenkins学习**：使用Docker方式安装Jenkins LTS（JDK 17版本），创建第一个Pipeline
4. **持续学习**：关注K8s和Jenkins社区动态和最佳实践
5. **团队分享**：将学习成果分享给团队

---

*最后更新：2024年*
*适用于：Java工程师、CI/CD开发人员*
