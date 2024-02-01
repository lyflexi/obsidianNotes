要做到项目的快速迭代，敏捷开发，就需要DevOps，即开发运维由一个团队负责，开发阶段，就要把部署、运维的工作考虑进去，而不是发布一个war包或者jar包后扔给运维不管了。
![[Pasted image 20240201104821.png]]

DevOps **是一系列做法和工具**，可以使 IT 和软件开发团队之间的**流程实现自动化**。

其中，随着敏捷软件开发日趋流行，持续集成 (CI) 、持续交付(Continuous Delivery)和持续部署 (Continuous Deployment)**已经成为该领域一个理想的解决方案。在 CI/CD 工作流中，每次集成都通过自动化构建来验证，包括编码、发布和测试，从而帮助开发者提前发现集成错误，这些关联的事务通常被统称为 CI/CD 管道（pipeline），由开发和运维团队以敏捷方式协同支持。

**DevOps落地** [https://kubesphere.com.cn/docs/devops-user-guide/understand-and-manage-devops-projects/overview/](https://kubesphere.com.cn/docs/devops-user-guide/understand-and-manage-devops-projects/overview/) **内置的动态供应 Jenkins Agent** [https://kubesphere.com.cn/docs/devops-user-guide/how-to-use/choose-jenkins-agent/](https://kubesphere.com.cn/docs/devops-user-guide/how-to-use/choose-jenkins-agent/)

# CI 持续集成（Continuous Integration）

持续集成，从字面意思上理解，就是不断的集成

持续集成（CI）可以帮助开发者更加方便地将代码更改合并到主分支

举一反三，持续集成让合并到**其他分支**也会更加方便，开发流程一般会先合入其他分支，在测试完毕以后，在最后阶段才会合入主干
![[Pasted image 20240201104913.png]]

- **开发人员(代号，10101)提交代码到 Source Repository (源代码仓库，如 GitLab)**
    
- **有代码更新到代码仓库后，会通过 WebHook 自动触发 CI Server(持续集成服务器，如 Jenkins)的相关功能，执行编译-测试-输出结果的流程，这里的测试一般只包含单元测试，不是我们常说的点点点功能测试，也不是接口测试。持续集成仅仅是让所有开发提交的代码成功集成到代码库中并正常协同工作。但并没有经过针对合入代码的自动化测试，所以集成的代码并不能马上发布到生产环境**
    
- **CI Server 会将执行结果返回给开发人员**
    

# CD 持续交付（Continuous **Delivery**）

持续交付将集成后的代码部署到更贴近真实运行环境的预生产环境（Staging）中。比如，我们完成单元测试后，可以把代码部署到连接数据库的 Staging 环境中更多的测试。如果代码没有问题，可以继续手动部署到生产环境中。
![[Pasted image 20240201104940.png]]

这里借用阮一峰老师的说法，持续交付（Continuous delivery）指的是，==频繁地将软件的新版本，交付给质量团队==或者用户，以供评审。持续交付强调的是，不管怎么更新，软件是随时随地可以交付的。注意，持续交付在自动化测试和集成结束后，不一定会自动部署。如果有自动部署，则是持续部署的概念了。

# CD 持续部署（Continuous Deployment）

持续部署是指当交付的代码通过评审之后，自动部署到生产环境中。持续部署是持续交付的最高阶段。这意味着，所有通过了一系列的自动化测试的改动都将自动部署到生产环境。它也可以被称为“Continuous Release”。
![[Pasted image 20240201104959.png]]
持续部署扩展了持续交付，开发人员提交代码到编译、测试、部署的全流程都不需要人工干预。因此，如果软件构建通过所有测试，则将自动部署。在这样的过程中，不需要人来决定何时生产以及生产什么。可以**向客户自动快速准确**分发组件，功能和修补程序。