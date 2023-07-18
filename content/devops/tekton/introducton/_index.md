---
linkTitle: "Tekton组件及资源对象"
weight: -400
---

# Tekton组件及资源对象

Tekton由如下7个组件构成

1）Tekton Pipeline：Tekton Pipeline是Tekton的基础组件，定义了一组Kubernetes自定义资源。作为构建模块的基础，你可以使用它们装配CI/CD流水线。

2）Tekton Trigger：Tekton Trigger可以实现基于事件实例化的流水线。例如，你可以在GitHub代码库合并PR时触发流水线的实例化和执行，也可以创建一个用于启动特定的Tekton触发器的用户接口。

3）Tekton CLI：Tekton CLI提供了一个名为tkn的命令行界面，你可以使用它与Tekton进行交互。

4）Tekton Dashboard：Tekton Dashboard是Tekton Pipeline的Web图形界面，展示有关流水线的执行状况。

5）Tekton Catalog：Tekton Catalog是一个高质量的、由社区贡献的构建模块（任务、流水线等）仓库。这些模块可随时在你的流水线中使用，例如docker-build、create-gitlab-release、git-clone等共用的构建模块。如果我们在Tekton中需要实现docker-build功能，无须自己编写Task，可直接使用Catalog项目中提供的功能配置，也可以根据自己的需求进行修改。

6）Tekton Hub：Tekton Hub是用于访问Tekton Catalog的Web图形界面。

7）Tekton Operator：Tekton Operator是一个Kubernetes Operator，可用于便捷地在Kubernetes集群上安装、更新以及删除Tekton项目。

Tekton引入了Task、Pipeline、TaskRun、PipelineRun、PipelineResource概念，通过它们指定任何想要运行的工作负载。

1）Task：定义了一个由Step组成的有序集合，每个步骤基于一组特定的输入调用特定的构建工具，并产生一组特定的输出。

2）Pipeline：定义了一系列与构建或交付相关的Task，可由事件触发或PipelineRun调用。

3）TaskRun：通过特定的输入、输出、执行参数来实例化Task。换句话说，Task告诉了Tekton该做什么，而TaskRun则告诉Tekton在什么参数基础上做。

4）PipelineRun：实例化特定的Pipeline，在一组特定输入的基础上执行Pipeline并产生一组特定的输出到特定目标。

5）PipelineResource：用于定义Task中Step的输入和输出的位置。

每个任务都在自己的Kubernetes Pod中执行。因此，默认情况下，流水线中的任务不共享数据。要想在任务之间共享数据，你必须显式配置每个任务，以使其输出可用于下一个任务，并将先前执行的任务的输出作为其输入。任何一个任务都适用此规则。

Task与Pipeline的适用场景如下。

1）Task适用于简单的工作负载，诸如运行测试、代码检查或构建Kaniko缓存。每个Task在独立的Kubernetes Pod中执行，使用独立的存储空间。Task在定义简单任务的同时保持配置上的简单。

2）Pipeline适用于复杂的工作负载，诸如对代码的静态分析、测试、构建、部署这类需要完成多个任务的综合项目。