---
linkTitle: "Tekton的概念模型"
weight: -500
---

# Tekton的概念模型

Tekton的主要功能是实现持续集成和交付部署。Tekton Pipeline是其核心组件，其他组件都是建立在Tekton Pipeline之上的。

## Step、Task和Pipeline

Step（步骤）是CI/CD工作流中最小的基础操作单元。Tekton通过在Step中定义的容器镜像执行每个Step。它可以实现各种需求，如编译代码后制作成镜像并推送到镜像仓库，再把相应的程序镜像发布到Kubernetes集群中，这些实施细节都需要在Step中定义。

Task（任务）是由Step组成的集合。Tekton中会以Kubernetes Pod的形式运行Task，而Task中的每个Step都将作为Kubernetes集群Pod中的一个容器运行。此种设计模式允许同一个Task中的众多Step共享相应的资源，例如，Pod中的各容器可以访问在Pod级别定义的存储卷。

Pipeline（流水线）是Task的集合，可以按特定的执行顺序定义Task。每个Task都将作为Kubernetes集群上的一个Pod运行。为了使各Task共同协作实现持续集成，Pipeline提供了如任务重试、排序控制等功能。

如下所示，Pipeline中引用了Task A、Task B、Task C、Task D，因为Pipeline中各Task默认为并发执行，无法满足后一个Task依赖前一个Task结果的需求，所以使用排序功能定义各Task的执行顺序。首先由Task A运行，当Task A运行结束后，Task B和Task C并发运行。待Task B和Task C运行结束后Task D运行。我们还可以观察到，每个Task中的Step都是按定义的顺序在执行。如果想调整Task中Step的运行顺序，只能在Task中重新对Step进行排序。

![task](/devops/tekton/images/task.png)

## 2 输入与输出资源

每个Task和Pipeline可能都有自己的输入和输出，在Tekton中称为输入和输出资源。例如，定义一个Task，此Task主要完成代码从编译到镜像的创建，这时可以将Git存储库作为输入，容器镜像作为输出。如下所示，该任务从存储库克隆源代码，然后运行一些测试，最后将源代码构建为可执行的容器镜像。

![in-output](/devops/tekton/images/in-output.png)

Tekton支持多种不同类型的资源，以下列出主要的资源类型。

·Git：Git存储库。

·Pull Request：Git存储库中的特定请求。

·Image：容器镜像。

·Cluster：为Tekton所在集群以外的Kubernetes集群提供访问。

·Storage：Blob存储中的对象或目录，例如Google云存储。

·CloudEvent：定义事件数据的规范。

## 3 TaskRun与PipelineRun

TaskRun可以在集群上实例化和执行Task。在Pipeline之外通过单独运行TaskRun来执行Task，并且通过TaskRun查看Task中每个Step执行的细节。PipelineRun可以在集群上实例化和执行Pipeline。通过PipelineRun的运行状态查看每个Task的详细信息和运行情况。

TaskRun和PipelineRun把资源与Task和Pipeline对象串联起来，通过运行TaskRun和PipelineRun来完成一次CI/CD的工作流。

创建TaskRun和PipelineRun有多种方式，可以通过手动创建、在Dashboard上创建或通过触发器触发自动创建等方式完成，如下所示。

![in-output](/devops/tekton/images/pipeline.png)

## 4 Tekton的运作方式

总体来说，Tekton Pipeline的核心功能是打包每个步骤。其中，Tekton Pipeline会在Step容器中注入一个二进制文件，将其作为入口点。当系统准备就绪时，将执行指定的命令。

Tekton Pipeline使用Kubernetes注释来跟踪Pipeline的状态。这些注释通过Kubernetes Downward API以文件的形式投射到每个Step容器中。该入口点的二进制文件密切关注投射的文件，只有在特定注释显示为文件时才会启动命令。例如，当要求Tekton在一个Task中连续运行两个Step时，注入第二个Step容器中的入口点的二进制文件将闲置等待，直到注释报告第一个Step容器成功完成。

此外，Tekton Pipeline会安排一些容器在Step容器之前和之后自动运行，以支持特定的内置功能，例如检索输入资源和将输出加载到存储库。你也可以通过TaskRun和PipelineRun跟踪它们的运行状态。在运行Step容器之前，系统还会执行许多其他操作来设置环境。
