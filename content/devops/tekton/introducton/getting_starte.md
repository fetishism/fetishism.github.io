---
linkTitle: "Tekton应用CI/CD配置"
weight: -300
---

# Tekton应用CI/CD配置

下面的Pipeline配置中使用了镜像标签自动生成、代码构建和镜像推送、应用镜像部署三个任务。也可以根据各自持续集成和交付的需求添加诸如代码质量检查、自动化测试等任务，不断完善持续集成和交付系统。

## Java语言配置示例

通过Maven工具构建Java代码。为了提高构建效率，需要为Maven本地仓库配置持久存储，否则会导致每次运行Maven都需要远程下载依赖包。

在Tekton的最佳实践中，鼓励对Task的重用，这样可以减少维护功能重复的Task。下面的镜像标签自动生成与应用镜像部署任务可以在其他Pipeline中重用。

基于Java代码的CI/CD配置示例如下：

```yml
# 为Maven本地仓库配置持久存储，容量大小根据Maven本地存储库大小而定
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maven-repo-local
spec:
  accessModes:
   - "ReadWriteOnce"
  resources:
    requests:
      storage: "20Gi"
---
# 配置Git仓库地址和分支
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git-resource-helloworld-java-spring
spec:
  type: git
  params:
   - name: url
      value: https://github.com/knativebook/helloworld-java-spring.git
   - name: revision
      value: master
---
# 配置镜像地址（镜像标签不用配置，创建时自动生成）
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: image-resource-helloworld-java-spring
spec:
  type: image
  params:
   - name: url
      value: docker.io/{username}/helloworld-java-spring
---
# 此Task为镜像自动生成标签（重用Task）
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: generate-image-tag
spec:
  resources:
    outputs:
     - name: builtImage
        type: image
  results:
   - name: timestamp
      description: Current timestamp
  steps:
   - name: get-timestamp
      image: bash:latest
      script: |
        #!/usr/bin/env bash
        ts='date "+%Y%m%d-%H%M%S"'
        echo "Current Timestamp: ${ts}"
        echo "Image URL: $(resources.outputs.builtImage.url):${ts}"
        echo $(resources.outputs.builtImage.url):${ts} | tr-d "\n" | tee
          $(results.timestamp.path)
      volumeMounts:
       - name: localtime
          mountPath: /etc/localtime
  volumes:
   - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
---
# 此Task创建代码，然后将代码推送到Docker Registry
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: mavn-build-and-push
spec:
  params:
   - name: imageUrl
      type: string
  resources:
    inputs:
     - name: git-source
        type: git
  steps:
   - name: maven-compile
      image: maven:3.6.1-jdk-8-alpine-private
      workingDir: "$(resources.inputs.git-source.path)"
      command: ['/usr/bin/mvn']
      args:
       - 'clean'
       - 'install'
       - '-D maven.test.skip=true'
      volumeMounts:
       - name: maven-repository
          mountPath: /root/.m2
   - name: build-and-push
      image: gcr.io/kaniko-project/executor:debug-v0.24.0
      env:
       - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
       - /kaniko/executor
      args:
       ---dockerfile=$(resources.inputs.git-source.path)/Dockerfile
       ---destination=$(params.imageUrl)
       ---context=$(resources.inputs.git-source.path)
       ---log-timestamp
      volumeMounts:
       - name: localtime
          mountPath: /etc/localtime
  volumes:
   - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
   - name: maven-repository
      persistentVolumeClaim:
        claimName: maven-repo-local
---
# 此Task通过kubectl命令向本地Kubernetes集群部署应用（重用Task）
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deployment
spec:
  params:
   - name: imageUrl
      type: string
   - name: appName
      type: string
  steps:
   - name: create-ksvc
      image: bash:latest
      command:
       - /bin/sh
      args:
       --c
       - |
          cat <<EOF > /workspace/knative-ksvc.yaml
            apiVersion: serving.knative.dev/v1
            kind: Service
            metadata:
              name: $(params.appName)
              namespace: default
              labels:
                application: $(params.appName)
                tier: application
            spec:
              template:
                metadata:
                  annotations:
                    autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
                    autoscaling.knative.dev/metric: concurrency
                    autoscaling.knative.dev/minScale: "1"
                  labels:
                    application: $(params.appName)
                    tier: application
                spec:
                  containers:
                 - image: $(params.imageUrl)
                    imagePullPolicy: IfNotPresent
                    env:
                     - name: TARGET
                        value: "Tekton Sample"
                    ports:
                     - containerPort: 80
          EOF
   - name: run-kubectl
      image: lachlanevenson/k8s-kubectl:v1.17.12
      command: ['kubectl']     
      args:
       - "apply"
       - "-f"
       - "/workspace/knative-ksvc.yaml"
---
# 此Pipeline用于串联各Task
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: helloworld-java-spring-pipeline
spec:
  resources:
   - name: git-source-p
      type: git
   - name: builtImage-p
      type: image
  params:
   - name: application
      type: string
  tasks:
   - name: generate-image-url
      taskRef:
        name: generate-image-tag
      resources:
        outputs:
         - name: builtImage
            resource: builtImage-p
   - name: mavnbuild-and-push
      taskRef:
        name: mavn-build-and-push
      runAfter:
       - generate-image-url
      resources:
        inputs:
         - name: git-source
            resource: git-source-p
      params:
       - name: imageUrl
          value: "$(tasks.generate-image-url.results.timestamp)"
   - name: deployment
      taskRef:
        name: deployment
      runAfter:
       - mavnbuild-and-push
      params:
       - name: appName
          value: $(params.application)
       - name: imageUrl
          value: "$(tasks.generate-image-url.results.timestamp)"
---
# 此PipelineRun为Pipeline传递相应资源参数并触发Pipeline运行
# PipelineRun可以手工创建或通过Dashborad UI创建
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: helloworld-java-spring-pipeline-run
spec:
  pipelineRef:
    name: helloworld-java-spring-pipeline
  params:
 - name: application
    value: helloworld-java-spring
  resources:
 - name: git-source-p
    resourceRef:
      name: git-resource-helloworld-java-spring
 - name: builtImage-p
    resourceRef:
      name: image-resource-helloworld-java-spring
  serviceAccountName: docker-git-sa
  timeout: 0h10m0s
```
