---
layout:     post
title:      分层namespace控制器的设计文档
subtitle:   多租户
date:       2019-12-26
author:     shmily
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - kubernetes 
    - namespace
    - 实习
---

# 分层namespace控制器的设计文档

## 介绍

&nbsp; &nbsp; 多租户共享集群在kubernetes中通常是通过namespaces来实现的。但是，随着租户规模和他们管理的资源数量的增长，单个命名空间通常不再能满足其需求。例如，一个团队可能希望在专用名称空间中管理多个微服务，以确保服务帐户被隔离，或者只是避免名称冲突。 因此，需要允许使用在租户之间共享的某些通用策略和资源来统一管理名称空间组。

&nbsp; &nbsp; 实现这一目标的一个建议是在Kubernetes生态系统中引入一个新的“Tenant”概念，其中包括有针对性的端到端流程，其中包括有关模板和管理的建议。 相比之下，本文档提出了**分层命名空间控制器（HNC）**-一种扩展K8s命名空间以包括分层概念的实用方法。 它仅允许用户声明namespace之间的父子关系，并通过生成的层次结构复制策略对象（例如角色或网络策略）

&nbsp; &nbsp;  HNC支持“租户”概念的许多用例，但级别较低；这种“租户的”概念是实施租约的基石，而不是成熟的租户本身。 对于需求与租户运营商所认为的流程有所不同的用户来说，这可能会很有吸引力。 此外，它还允许委派管理，例如通过赋予“名称空间管理员”创建“子名称空间”的能力而无需集群级特权。 最后，它定义了一个描述层次关系的标准，该层次关系可以由其他控制器使用，例如层次资源配额，而不必自己实现这些功能。

如果HNC帮助发现K8s生态系统中对层次结构的大量需求，它可以作为Kubernetes apiserver中first-class层次结构支持的原型。 然后，这种支持可以用来逐步扩展围绕namespace构建的其他功能(如identity、quotas 和其他policies)，以涵盖重要的新用例，而不会丢失现有功能的工作或只在现有的namespace上使用。

## 背景
Kubernetes多租户工作组（MTWG）一直在探索如何使多个团队或客户更容易通过其apiserver共享K8s集群。 MTWG的当前记录计划是开发一个名为“租户CRD”的operator，该operator在K8s生态系统中提出了一个新的标准化概念-“租户”-以及与租户相关的两种具体行为：

*  **namespace管理员**：每个租户可以与多个命名空间相关联，并且可以具有多个管理员。 这些管理员将自动获得名称空间中的某些RBAC权限。
*  **namespace templates**：创建新的Tenant时，它可能包含要包含的命名空间的描述以及这些命名空间的“模板”，其中包括要在这些命名空间中实例化的资源列表。 这些资源通常是策略对象，例如RBAC角色或网络策略。

我认为此建议有两个缺点：

*  **关注点分离**：模板和管理是正交的关注点，因此将它们绑在一起会产生成本，尤其是在尝试定义一个基本的新概念时。 例如，租户运营商选择的模板解决方案可能无法满足所有用户的需求。 其他用户可能需要当前投标书未涵盖的“子承租人”。 分离正交关注点可使每个关注点更轻松地独立发展。
* **生态系统复杂性**：租户CRD向K8s引入了一个基本的新概念，而不是扩展现有概念，这增加了K8s生态系统的整体复杂性。 还将上面列出的实现问题与该概念联系在一起。 所有这些都可能阻止用户使用新概念，从而使引入它们的目的无法实现。

作为替代方案，本文档将租户CRD的某些方面分解为一个新的，更简单的原语，然后可以独立于租户CRD以及租户CRD本身的构建基块使用它

这个新原语是在Kubernetes命名空间之间引入父子关系的，它是由分层命名空间控制器（HNC）定义和实现的。 HNC的主要目的是允许父代中的K8s资源在子代命名空间中“可见”或“继承”。 它没有在Kubernetes中添加任何新的基本概念，而是扩展了现有概念（命名空间）的行为。

将名称空间的模板管理和生命周期管理与其管理分离，为用户提供了更多选择，并提供了与其他工具的更轻松的集成点。 例如，日志记录或监视系统可以识别并支持分层名称空间，而无需用户遵循Tenant CRD的生命周期模型。 它还允许在多租户用例之外使用租户CRD的某些有用概念，例如，单个租户可能希望使用HNC功能来组织其资源。

## 其他相关工作
Google的[Anthos Config Management](https://cloud.google.com/anthos-config-management/docs/)（ACM）通过在整体Git存储库的文件系统中建模名称空间层次结构，并将对象从父名称空间复制到K8s apiserver中的子名称空间，从而支持与该提议类似的模型。 相比之下，层次命名空间控制器（HNC）完全在K8s apiserver中运行，并使用命名空间中的CRD来表达层次结构。 如果ACM足够成熟，将来可能会允许使用HNC。

先前提出的将标签添加为[labels as a first-class resource](https://docs.google.com/document/d/1Ql_Y-DRbvPCSA-o36CLOs3t4tQUoSvbQxtVEI4FCtEo/edit#heading=h.tx3ialsfu4bx)（具有自己的ACL，更复杂的子字符串匹配）的建议也可以用于实现层次关系。 实际上，可以使用标签对任何层次进行建模，而HNC本身正是出于此目的将标签添加到名称空间中。 该设计没有对常规标签提出任何特殊的行为，因为它们似乎是非常正交的问题，但是可以与一流标签或其他基于标签的解决方案（例如[Namespace Configuration Controller](https://blog.openshift.com/controlling-namespace-configurations/)）结合使用。

社区中对分层配额的兴趣也很浓厚，对分层名称空间的讨论也很早。

## 整体设计

HNC的运行方式是允许名称空间管理员（在本文档中：对名称空间具有较高特权的用户，如下定义）指定名称空间之间的父子关系，然后将某些对象（如RBAC RoleBindings）从父对象复制到子对象。 这启用了三个主要的分层用例：

* **策略执行和管理**：RBAC，配额和网络策略之类的策略从父级传播到子级，从而允许在命名空间的子树中统一执行。 RBAC传播的结果是，在父命名空间中具有特权的用户在子命名空间中也具有相同的特权。 这允许诸如团队和子团队之类的结构。
* **资源共享**：ConfigMap和Secrets可以从父母共享给孩子。
* **自助式名称空间的创建**：名称空间管理员可能会要求HNC创建子名称空间，即使他们没有（群集级别）特权来自己创建这些名称空间也是如此。 但是，它们在这些子名称空间中没有获得除父名称空间之外的任何其他特权。

所有这些都是在遵守管理员的权利法案的情况下完成的，如下所述。

在较高的层次上，用户模型的工作方式如下：

* 每个名称空间都包含一个具有Kind```hnc.x-k8s.io/HierarchyConfiguration```和名称层次结构的自定义资源。 最重要的字段是```.spec.parent```（用于标识其父名称空间（如果有））和requiredChildren（用于请求自助名称空间）。 该状态还列出有关根于命名空间（如果有）的子树的信息
* HNC传播的对象可能无法修改或删除。 例如，子名称空间中传播的```ResourceQuota```可能不会被该名称空间的任何用户编辑，即使是通常可以更新```ResourceQuotas```的用户也是如此。
* 任何具有非平凡层次结构的名称空间（即任何父级或子级）都将获得一组标签，以标识它们所属的所有子树。 例如，如果foo是bar的父级，则bar名称空间将标记为```foo.tree.hnc.x-k8s.io/depth=1```表示它是以foo为根的子树的成员。 期望其他工具和控制器使用```HierarchyConfiguration``` CRD或namespaces上的标签来了解层次结构。

由HNC管理的任何命名空间K8s资源（即已传播的资源）都将被赋予hnc.x-k8s.io/inheritedFrom = <ancestorNS>标签。 标签的值对用户具有参考意义，但其主要目的是让HNC知道它可以随意更新或删除资源。

管理员可以配置要传播的资源类型。 例如，大多数用户可能希望传播RBAC角色和可能的秘密，而不是服务和部署。 我们可能禁止传播某些（已知）对象类型，但显然我们不能禁止用户定义类型。 管理员可以确保仅传播适当的类型。

根据定义，此控制器对未命名空间的资源没有任何影响。因此，它仅适用于“软多租户”方案。它也没有消除群集中所有名称空间必须唯一命名的限制。它仅增加了在命名空间之间定义父子关系的功能。

HNC的MVP将分两个阶段实施。第1阶段实现了主要逻辑，但缺少第2天的功能，因此无法在实际用例中使用它。它基本上适合OSS社区的成员评估该技术，但不应在任何实际用例中使用。

第二阶段的目标是允许HNC在非产品集群中使用，并将引入缺少的功能。这些功能中的一些今天已经很清楚了，例如允许管理员更改层次结构，而另一些则是未知的。下文所述的用户旅程概述了阶段1和阶段2之间的区别。

从MVP到1.0的其余过程将基于用户反馈以及不断提高的稳定性和性能。

## 超出范围
该提案未解决：

* **Hierarchical Resource Quotas**: 这些可以作为遵守该提议中数据结构的另一个Webhook添加。 另外，我们可以扩展现有的ResourceQuota以在多个名称空间中运行，并使用下面描述的树标签集成到层次结构中

## 用户历程
租用具有多个名称空间的租户

HNC可用于实现“租户”概念，其中租户中的用户可以访问多个名称空间（在单个名称空间内实现租用是微不足道的）。

使用HNC，可在阶段1中按以下步骤完成此操作：

* **群集管理员（cluster administrator ）**创建一个或多个**root namespaces**根名称空间，并使用将从该根继承的所有租户共享的策略对象填充这些名称空间。 例如，他们可以分别创建具有或多或少限制策略的```prod-root```和```nonprod-root```

    * 策略对象集被硬编码为RBAC对象，NetworkPolicy，ResourceQuota，PodSecurityPolicy，Secret和ConfigMaps。 如果存在，这些对象将始终复制到其子名称空间

* 要创建租户，集群管理员通过将所需的tenant namespace 命名空间名称添加到适当的根命名空间的HierarchyConfiguration对象中的requiredChildren字段中来创建租户命名空间。然后，群集管理员通过在租户名称空间中创建适当的RBAC对象来指定租户管理员。 HNC强制这些管理员不能修改从根名称空间复制的任何对象，但是对于附加策略（例如RBAC，NetworkPolicy和PodSecurityPolicy），群集管理员必须确保租户管理员受信任或必须避免提供他们创建这些对象的权限。此限制不适用于限制性策略，例如ResourceQuotas。
* 如果租户管理员有权修改租户名称空间中的HierarchyConfiguration对象，则他们可以创建其他子命名空间，并委派已在这些命名空间中拥有的任何RBAC权限。
    * 请注意，由于ResourceQuotas不是分层的，因此每个新的子命名空间实质上都使租户可以访问更多资源。
* 租户管理员可以修改层次结构，只要所做的任何更改都不会剥夺父级名称空间管理员对子级的访问权限（请参阅下文）。

在第2阶段，我们可能会解决上述过程的一些缺点，例如：
* 允许群集甚至租户管理员修改被复制的策略对象列表。
* 限制任何一个租户可以创建的子命名空间的数量，以限制解决ResourceQuota限制的能力。

## 详细设计
### 定义
在本文档的其余部分，我们将使用以下定义：

* **祖先和后代**：由于名称空间链接到树中，因此每个名称空间可能具有多个祖先和后代。 它也可能有一个单亲父母以及零个或多个孩子。 命名空间的所有后代的集合，加上命名空间本身，被称为命名空间的子树。
* **Namespace admin（或仅是admin）**：如果用户可以修改命名空间的子树的结构，则该用户是admin（在此文档的上下文中）。 下面定义了更正式的规则。
* **Superadmin超级管理员**：如果用户是名称空间祖先之一的管理员，则该用户是相对于该名称空间的超级管理员。 超级管理员通常比管理员更强大，如下所述。
* **Subadmin**：用户是相对于名称空间的子管理员，如果他们是其后代之一的管理员。如下所述，子管理员通常不如管理员强大。

## 创建和修改层次结构

Anthos Config Management在Kubernetes之外的Git中定义了层次结构。 因此，任何能够修改存储库的用户都可以完全控制整个层次结构。 相比之下，HNC在Kubernetes中定义了层次结构，因此，我们必须非常小心地控制如何定义和更新此结构。

### HierarchyConfiguration资源

HNC在每个命名空间中使用以下结构自动创建HierarchyConfiguration资源：

```
apiVersion: hnc.x-k8s.io/v1alpha1
kind: HierarchyConfiguration
metadata:
  name: hierarchy
  namespace: <ns name>
spec:
  parent: <parent ns name, or absent if top-level>
  # ... will likely add configuration fields in the future ...
status:
  conditions: [] # like PodConditions, nonempty if there are errors, eg a cycle
  children: <list of children>
  # ... future extensions: eg num objs copied, effective config, ...
  
```

因此，具有名称空间中的层次结构的获取和监视权限的用户可以检查层次结构或将其更改通知。 如本文所述，这可以解决K8s apiserver中的限制。

### 分配管理员权限

命名空间管理员定义为对该命名空间的HierarchyConfiguration资源具有更新权限的任何用户。 这可能包括具有ClusterRoleBinding的用户，该用户授予他们对所有命名空间中的HierarchyConfiguration资源的访问权限，或仅在一个命名空间内进行绑定。

尽管未在后代名称空间中授予管理员任何显式权限，但如果HNC正在传播RoleBindings，通常也将向他们授予后代权限

如果我们允许任何管理员通过修改.spec.parent字段随时允许其名称空间自由移动到另一个父级，则会出现严重问题：

* 他们可以将名称空间移动到新的父对象，从而获得对该名称空间中任何传播的对象（包括Secrets）的访问权限。
* 他们可能会留下旧的父命名空间，从而剥夺对（以前）超级管理员的访问权限

由于这些原因，HNC在准入控制器中实现自定义授权规则。 我们的目标是对管理员实施以下限制（以实现管理员BoR）：

1. 他们应该能够修改以其命名空间为根的子树的层次结构（委托管理）。
2. 管理员不能从任何超级管理员中删除权限。 例如，管理员可能不会简单地删除.spec.parent字段，因为这将删除超级管理员对命名空间的访问。
3. 如果名称空间是free的根，则仅当该树的管理员同意允许它时，管理员才应该能够“加入”另一个子树。 双方（当前的管理员和将来的超级管理员）都必须同意这一点，因为命名空间既可以访问新祖先（例如Secrets）的资源，又可以失去新超级管理员的自治权。

为了实现这些愿望，HNC的验证Webhook限制了谁可以修改.spec.parent字段以及如何进行以下操作：

1. 如果名称空间没有父级，则管理员可以将一个名称空间设置为它们已经是管理员的任何名称空间。 但是，除非遵守规则2，否则他们也不能取消设置或更改现有的父项，除非它们也是名称空间当前树的根的管理员。  
 * 例外：如果父名称空间不存在（因为它的设置不正确，或者父名称空间已删除），则管理员可以取消设置/更改它。
2. 管理员可以更改任何后代的父母，只要他们是新父母和新父母的最近共同祖先的管理员即可。 没有此规则，管理员可以有效地删除超级管理员对后代名称空间的访问。

如果两个名称空间都已经存在，那么这一切都可行，但是我们还希望允许超级用户通过```kubectl apply -f```在目录上创建整个层次结构的场景。 但是，在这种情况下，我们不一定要确保每个父名称空间都存在，或者已经在其中创建了```HierarchyConfiguration```对象，但是我们也不想让任何名称空间管理员简单地设置没有父名称空间的父对象。 尚未创建。

因此，在确定用户是否是父母的管理员时，Webhook将遵循以下过程：

1. 如果名称空间存在，但 ```HierarchyConfiguration```对象不存在，则如果该用户对该名称空间中的```HierarchyConfiguration```对象具有创建权限，则该用户将被视为该名称空间的管理员。 然后，HNC将代表用户创建```HierarchyConfiguration```对象。
2. 如果名称空间尚不存在，并且该用户同时具有名称空间的创建权限和```HierarchyConfiguration```对象的创建权限（即通过```ClusterRoleBinding```），则将被视为该（丢失）名称空间的管理员。但是，在这种情况下，我们不一定要确保每个父名称空间都存在，或者已经在其中创建了HierarchyConfiguration对象，但是我们也不想让任何名称空间管理员简单地设置没有父名称空间的父对象。 尚未创建


如果名称空间存在，但```HierarchyConfiguration```对象不存在，则如果该用户对该名称空间中的```HierarchyConfiguration```对象具有创建权限，则该用户将被视为该名称空间的管理员。 然后，HNC将代表用户创建```HierarchyConfiguration```对象。
如果名称空间尚不存在，并且该用户同时具有名称空间的创建权限和```HierarchyConfiguration```对象的创建权限（即通过```ClusterRoleBinding```），则将被视为该（丢失）名称空间的管理员

当名称空间的```HierarchyConfiguration```对象引用不存在的父代时，将在其条件状态中设置适当的条目。 创建命名空间后，HNC将注意到新添加的内容，在新的命名空间中创建```HierarchyConfiguration```对象，并删除子级的条件。


### 创建和删除名称空间
HNC的第一阶段将不限制用户创建或删除名称空间的方式，因为目前尚不清楚该领域的要求，“不执行任何操作”是最简单的实施决策。

例如，这具有以下（可能令人惊讶）的后果：

1. 创建名称空间的用户不会自动成为该名称空间的管理员。 必须有人授予该用户对该名称空间中HierarchyConfiguration资源的更新权限。
2. 现有名称空间的管理员无法创建子名称空间。
3. 命名空间可以保留后代，但可以删除。 这不会删除后代名称空间，甚至不会更改其父级标签。 如果使用与删除的名称空间相同的名称创建新的名称空间，则新的名称空间也将自动成为新的父名称空间

这些限制几乎肯定不适合HNC的beta或GA版本； ＃2特别有限制。 在第二阶段中克服它们的方法是待定，但可以包括（按与上面相同的顺序）：

1. 当用户创建名称空间时，mutating admission webhook 会将用户名添加到名称空间对象的注释中（例如```hnc.x-k8s.io/initialAdmin```）。 创建名称空间及其```HierarchyConfiguration```后，HNC将更新RBAC以使该用户成为管理员，并删除注释。 显然，除变异许可webhook之外，没有人可以设置或修改此注释的值
2. 为了允许任何用户创建子名称空间，我们有两个选择。 首先，我们可以授予所有用户在RBAC中创建名称空间的权利，并且完全依靠HNC来实施限制。 另外，我们可以将子名称空间的创建委托给HNC本身。 后者可能更安全，尽管不利之处在于，任何其他验证器都将无法识别谁在创建名称空间。 可能的用户模型是允许名称空间管理员在```HierarchyConfiguration```资源中设置```.spec.new-children```字段。 一旦创建子名称空间，协调器将删除此字段。
3. 我们可以使用ownerReferences来确保在删除祖先时删除后代名称空间（类似于rm -rf）。 这比允许孤立命名空间更可取，因为删除祖先很可能会导致非集群管理员无法访问命名空间，并且不良的清理工作不应导致向被委托管理的集群管理员发出凭单。 其他人。 最好避免删除父名称空间（类似于不带-f的```rm -r```），因为清理大型层次结构很困难。 也许这可以是每个命名空间的配置选项

需要进一步的工作来发现此处的真正要求。

### 确保法律层次

HNC准入控制器将尝试拒绝对创建周期的```HierarchyConfiguration```的任何更改。如果某个循环经过控制器（例如，禁用了HNC，创建了一个循环，并重新启用了HNC），则通过在.status中添加一个条目，将将该循环涉及的所有命名空间中的```HierarchyConfiguration```标记为无效。 ```.status.exceptions```,并且HNC将在该循环中停止对对象进行任何更改-也就是说，现有对象将不会被删除，新对象也不会被添加。

### 处理缺乏交易性

HNC的一个重要缺点（也是ACM最初未使用这种模型的原因）是它不是事务性的。也就是说，随着层次结构和可继承对象的修改，这些更改需要花费时间才能传播，并且在此期间，系统状态可以与源状态或目标状态任意不同。在RBAC对象的上下文中，这意味着可以任意拒绝访问或错误地提供访问，直到系统收敛为止。

但是，其余的K8s生态系统（如许多分布式系统）也不是事务性的。例如，在将服务部署到生产中时，通常必须以特定的顺序部署其工件，例如架构更新，二进制映像，最后是部署本身。同样，您必须在CR之前应用CRD，而不是依靠足够智能的系统来理解声明性规范并找出安全部署所有组件的方法。

尽管这不是理想的选择，但现在可以简单地警告用户不要在一次提交中对层次结构进行某些类型的更改，这可能同样可以接受。在某些情况下，我们应该已经这样做了，因为即使在今天，Nomos也不是完全事务性的（更新所有名称空间仍然需要时间）。但是，我们将尝试做出一些保证，例如确保在添加“新”对象之前（例如从新的父对象中）删除所有“旧”对象（例如，从已删除的先前父对象中删除）。由于Kubernetes调用协调器的方式，很难一概保证始终遵循此规则，但是任何违规行为都应短暂存在。

将来，将此功能从附加控制器迁移到OSS apiserver本身可以改善或解决此问题。它还可以允许其他可用性增强功能，例如层次监视和更有用的命名空间CRD，以及在OSS中标准化模型的固有好处。但是，这是对该提案的可选的将来扩展。

### 与层次结构集成

HNC本身之外的许多控制器可能希望检查层次结构。 例如：

* 网络策略可能希望允许名称空间的子树内的某些类型的连接，而不是单个名称空间。
* 同样，我们可能希望在子树上应用**ResourceQuota**

其中一些对象（例如**ResourceQuota**）仅在单个名称空间上运行； 这些将需要编写具有层次结构意识的替换，这超出了本文档的范围。 其他-例如NetworkPolicy-可以基于标签（**.spec.ingress.from.namespaceSelector**）选择多个命名空间。
因此，如果HNC将层次结构导出到标签，这将允许当前（**NetworkPolicy**）和将来的（**hierarchical ResoureQuota**）控制器完全集成到层次结构中。

### tree标签

天真地，我们可能希望能够编写一个标签选择器，例如ancestor = foo，但是由于每个命名空间可能具有多个祖先，并且由于每个标签可能仅具有一个值，因此这种方法将行不通。

但是，添加多个实现相同目的的名称相似的标签并不容易。 结果，如果名称空间栏是名称空间foo的祖先，则HNC在foo上设置以下标签：

``` bar.tree.hnc.x-k8s.io/depth = <distance to bar> ```

bar本身也包含此标签，深度为0。
然后，我们可以在子树上使用以下标签选择器：

``` 
#The subtree rooted at bar, including bar:
matchExpressions: 
  - {key: bar.tree.hnc.x-k8s.io/depth, operator: Exists}

# All descendants of bar, excluding bar:
matchExpressions:
  - {key: bar.tree.hnc.x-k8s.io/depth, operator: NotIn, values: [0] }
  
# The immediate children of bar:
matchLabels:
  bar.tree.hnc.x-k8s.io/depth: 1
  
```
  
  
  这种解决方案的明显问题是，每个名称空间都将为其每个祖先包含一个标签。 对于非常深的层次树，这可能会导致大量附加标签。 但是，大多数层次树可能不会很深，因此希望可以管理标签的数量。


### 祖先注解

树标签的选择效率很高，但很难阅读。 作为附加的集成点，HNC将以DNS名称的形式用整个祖先树对命名空间进行注释。 例如，如果名称空间foo是bar的子级，它将收到注释
```hnc.x-k8s.io/ancestry = foo.bar。```

在简单的情况下，此方法效果很好，我们可以想象我们可能会看到诸如v1.service1.team1.org1之类的名称。 不幸的是，在K8s中，每个名称空间都必须是唯一的，因此，名称空间不太可能会收到诸如v1之类的名称-该名称空间被称为```v1-service1-team1-org1```之类的可能性更大。 但是，如果天真地扩展，将产生不幸的注释：
```hnc.x-k8s.io/ancestry = v1-service1-team1-org1.service1-team1-org1.team1-org1.org1```

为了避免这种情况，如果HNC可以简单地复制现有的DNS后缀（所有时间段都转换为连字符），则在创建此注释时，HNC可以自动删除任何带后缀的后缀。 不幸的是，结果路径将不是唯一的：例如，如果org1有两个孩子，其中一个被称为team1，另一个被称为team1-org，则它们都将被赋予相同的team1.org1注释。 类似的问题可能会在层次结构树的任意深度发生。

解决此问题的选项包括：

* 只需接受可能发生歧义。
* 允许用户指定是否要在层次结构树中进行这种折叠，如果需要，请接受由此产生的任何歧义。
* 允许用户指定他们要折叠，如果是，请启用Webhook以防止创建歧义路径。
* 使用明确，完全扩展的名称创建一个额外的fullAncestry注释（名称为TBD）。

## 层次结构中常规对象的行为
### 正常行为
在阶段1中，每种类型都可以具有以下三种行为之一：

* 没有对象被复制（除Roles和RoleBindings外的所有对象的默认值）
* 复制所有对象
* 标有hnc.x-k8s.io/inherit = true的对象将广播

在HNC的第1阶段中，将由HNC控制的kinds列表是用户可配置（最初的里程碑将对可能要复制的对象的类型进行硬编码-例如RBAC命名空间的对象，但这将由alpha版本修复。）的，但在群集范围内。在第2阶段中，我们可能会添加更复杂的行为，例如标签选择器（用于排除（请参见下文）和包含），以及针对每个子树配置这些Kinds（而不是全局）的功能。有关如何表达此配置的详细信息为TBD，但几乎可以肯定，它将在阶段1中的群集范围内的config.spec中，如果需要每个子树的配置，则在```HierarchyConfiguration.spec```中。

一旦HNC确定对象应该对子命名空间可见，就将其从父对象复制到子对象。在复制的对象上设置了```hnc.x-k8s.io/inheritedFrom = <ancestorNS>```批注，不仅可以让HNC知道这是一个托管对象，还可以使用户了解为什么存在该对象。 HNC使用此批注的唯一方法是知道如果删除了源对象，或者源名称空间不再是祖先，则必须删除该对象。

验证webhook会拒绝任何修改这些对象的尝试（但请参见下一节有关替代的内容）；如果已经存在相同名称的对象，则HNC不会删除它，而是将其视为错误并将其添加到命名空间及其所有祖先的```.status.conditions```数组中。

### 排除

尽管绝不应该传播许多对象（例如Pod），但在其他更微妙的情况下，不应将对象复制到后代名称空间。 这里的关键示例是服务帐户令牌，这是一种Secret类型。 这些内容会由K8s内置控制器自动删除，永远不要复制。

结果，HNC将支持排除的概念，该概念最初仅由一个硬编码规则组成：不得复制SA令牌。 在第2阶段或更高版本中，我们可以为管理员增加基于其他条件（如标签选择）设置其他排除项的功能。

总体而言，流程图将始终为：

* 如果从未复制对象的Kind，请停止。
* 如果由于某种原因排除了该对象，请停止。
* 如果包含该对象，则将其传播给子对象。

### 覆盖父母的对象

作为第2阶段的功能，我们可以为名称空间管理员提供“覆盖”对象的功能，例如通过将overrideFrom注释替换为overrideFrom注释。 此注释将指示在子名称空间中具有“冲突”对象不是错误。 此功能的优先级是待定（TBD）。

### 在namespaces继承标签和注释

除了由HNC本身添加的标签和注释（例如**hnc.x-k8s.io/inheritedFrom**）之外，由HNC传播的任何常规对象都将保留其所有标签和注释，而无需进行任何修改。但是，名称空间本身也可能具有标签和注释，并且我们可能发现将其中的一些子集从父级传播到子级很有用，就像传播对象的某些子集一样有用。

注意：在本节的其余部分，我将引用“标签”，但整个讨论也适用于注释。

如果选择继承名称空间标签，则将在阶段1中按以下方式实现它：

1. 全局配置对象或**.spec.propagatedLabels**字段将包含标签密钥的白名单，如果存在，将在以命名空间为根的子树中传播。在此列表中包含所有**hnc.x-k8s.io**标签是错误的。
2. 当一个名称空间成为另一个名称空间的子集时，所有带有白名单键的标签都将从父级复制到子级。同样，如果将白名单标签添加到名称空间，则该标签将广播到其所有后代。
3. 在阶段1中，子名称空间包含与主名称空间冲突的键的标签将是一个错误，就像存在同名对象的错误一样。与冲突对象一样，现有的冲突标签将被报告（不被破坏），但是网络挂钩将确保不引入新的冲突。
4. 当名称空间的父项发生更改时，它与父项共享的所有标签都将被传播，并且可以安全地删除。

禁止从父级覆盖对象的限制过于严格，因此对于名称空间标签也可能变得过于严格。因此，在阶段2中，我们可能希望允许覆盖。但是，这意味着名称空间上的标签是从其父项继承而来的，还是（例如）在分配父项之前在名称空间上设置的，这是模棱两可的。多亏了InheritedFrom标签，因此对象没有此问题-但您不能将标签放在标签上。

结果，如果我们在阶段2中实现标签覆盖，则HNC将遵守**HierarchyConfiguration**对象中的**.spec.overriddenLabels**字段，该对象将明确列出未从其父级继承的所有标签键（根据K8s文档（这表明注释可能是“小”或“大”，我假设此注释的大小没有有意义的限制。）。可以通过以下两种方式之一将密钥添加到此列表中：

* 足够强大的管理员可以显式添加密钥，以允许子名称空间与其父名称空间不同。例如，如果由于祖先名称空间中的**.spec.propagatedLabels**字段而传播标签，则只有该名称空间的管理员可以添加例外，以允许该键被覆盖。
* 更改名称空间的父级后，该名称空间中已经存在的所有标签（即在从父级传播任何标签之前）都将添加到**overriddenLabels**列表中。如果我们不这样做，那么如果您设置了一个新的父项，然后立即取消设置，则命名空间可能会“丢失”标签，因为假定所有重复的标签都已传播。

## 其他实施细节
HNC控制器将作为kube-system 2.0控制器部署在hnc-system命名空间中。


# 附录
## 拒绝了高级用户模型的替代方案
### 拒绝：在名称空间上使用父标签
非常感谢liggit@google.com开发本节。

如果只有一个重要设置，为什么要引入**HierarchyConfiguration**资源？ 为什么不简单地在名称空间本身上放置一个父标签（所有有效的名称空间名称都可以表示为标签）呢？ 这似乎更加优雅（单个字段而不是任意命名的对象）。 更实际地讲，它允许名称空间作为原子操作在层次结构中创建，而不是创建名称空间，等待实例化**HierarchyConfiguration**，然后在单例上设置父级。

尽管它具有不可否认的优雅和吸引力，但该建议仍存在几个关键问题。 首先是中心假设不太可能保持正确：也就是说，**.spec.parent**不太可能成为层次结构中唯一值得注意的属性。 例如，其他潜在属性包括：

* [很有可能]用于自助式名称空间创建的“必需子级”列表（请参见下文）。
* [类似]传播和覆盖的名称空间标签的列表（请参见下文）。
* [可能]每个名称空间的“传播状态”，可用于控制新策略的部署（详细信息将在后面进行介绍）；
* [类似]在子树中传播的对象种类（例如Secrets，ConfigMaps）列表。
* [可能]级联删除选项：应该将孩子与父母一起删除，还是应该删除父母作为孤儿？
第二个问题是，我们已经具有复杂的状态要报告，包括（至少）子树上的条件或错误列表。复杂的数据结构很难匹配名称空间上的键值注释。

显然，所有这些规范或状态字段都可以简单地编码为名称空间本身的标签或注释，但是我们拥有的字段越多，解决方案就越糟糕。虽然没有理由既不能为简单情况提供父标签，也不能为复杂配置提供**HierarchyConfiguration**对象，但这种划分似乎会削弱解决方案的精髓。

第三，名称空间层次结构将是任何群集管理中高度敏感的部分，并且将受到Webhook的保护。理想情况下，此Webhook应该无法关闭，这意味着，如果Webhook服务暂时中断，则对层次结构的所有更改都将被拒绝。如果层次结构是在名称空间上表示的，则意味着对名称空间的所有更改也将被拒绝。考虑这可能会对正在缓慢采用HNC的群集造成的影响：HNC中的任何错误或配置错误都可能导致群集的许多常见操作被阻止，即使对于未使用HNC功能（例如，名称空间）的用户没有父母或子女）。

第四，最小的**HierarchyConfiguration API**并不比标签复杂得多。

最后，这将是一个棘手的决定。 如果HNC在决定切换到HierarchyConfiguration API之前得到了广泛采用，我们要么让用户经历相当痛苦的过渡，要么花费大量工程努力使过渡变得更加容易-这意味着要处理多个（可能会发生冲突） 真理之源。

这些异议似乎是拒绝该用户模型的充分理由。

### 拒绝：在名称空间上使用**ownerReference**

注意：**ownerReferences**不能是命名空间范围的，但是它们本身可以是命名空间。在本节中，我假设实际上是允许这样做的。

标准**ownerReference**元数据数组允许对象引用其他对象作为其所有者。通常，这用于垃圾回收：如果所有所有者都已删除，则对象也将被删除。假设我们希望子级名称空间与其父级一起删除（参见下文），使用**ownerReference**条目来表示“父母身份”似乎也是合理的。

但是，这里有一些并发症：

* 拒绝父标签的相同原因在这里也适用：除了简单指定父标签外，可能还有许多与层次结构相关的字段，并且如果HNC Webhook不起作用，我们不想阻止名称空间的创建，编辑或删除。正确地。
* 可能要在名称空间上使用**ownerReferences**来指示除父母身份之外的其他关系。例如，**test-resources**命名空间可能受工作负载依赖于其存在的其他命名空间的依赖。由于**ownerReferences**通常禁止交叉命名空间引用，因此本示例可能有些人为设计，但是如果允许交叉命名空间引用，则可能会出现问题。
* 允许对象具有多个所有者，但只能有一个父对象（请参见下文）。因此，我们需要验证控制器来防止创建多个父对象。另外，我们可以使用**ownerReference.isController**（仅允许设置一个条目）作为“是父项”的同义词，但这不是很优雅。

如此说来，我们可能最终会在HNC的实现中使用ownerReferences。 例如，如果我们选择删除子项及其父项（或将其作为配置选项提供），则几乎可以肯定的是，通过在命名空间之间设置ownerReference条目来做到这一点。

### 拒绝：使用群集范围的**HierarchyConfiguration**资源

为什么不在每个命名空间中只有一个**HierarchyConfiguration**资源，为什么不拥有一个与命名空间同名的集群范围资源呢？ 基本上有三个原因：

1. 管理员继承：如上一节所述，对名称空间范围的对象的权限可以自然地从父级继承到子级，而对群集范围的对象的权限则必须进行大量的额外开发。
2. 可用性：如果您授予某人访问“名称空间中的所有资源”的权限，这似乎并不奇怪，其中不包括修改名称空间子树的功能。
3. 审美：我通常更倾向于将资源的范围缩小，并且将概念紧密地联系在一起。 这样可以使事物不在群集范围之内，而在名称空间之内。（同样，将来，如果可能的话，越来越多的群集作用域资源（例如CRD）可以移入名称空间，那将是很好的。）


### 拒绝/超出范围：支持具有多个父项的名称空间，或仅使用标签

允许名称空间具有多个父级-创建DAG而不是树-会允许某些其他用例，但要复杂得多，而且尚不清楚父子关系是对其建模的最佳方法。 一种可能的选择是完全放弃层次结构，而只使用标签，这是其他控制器（如命名空间配置控制器（NCC））采用的方法，并且除层次结构外，**Anthos Config Management**也支持该方法。


但是，虽然标签比层次结构灵活得多，但是这种灵活性也有缺点：

* 层次结构施加了严格的防护栏和明智的默认值，这是组织经常需要的。虽然可以使用标签对层次结构进行建模，但是要更难确定是否以符合组织政策的方式使用它们。
* K8的标签当前未使用ACL，因此HNC（或类似的控制器）将必须实现复杂的逻辑以允许委派和按标签的授权。通过将自己限制为层次结构，可以极大地减少问题（尽管仍然很复杂）。

在考虑了层次结构与标签的优缺点之后，很显然它们并不是真正的替代品，而是互补的。例如，发现“纯”层次结构过于受限的用户可能将HNC与基于标签的控制器（例如NCC）结合使用。

在这种组合中，拥有父名称空间可提供隐含的默认策略，而标签可用于添加特定功能。也就是说，子名称空间通常会带有在某种程度上需要其起作用的所有策略，而如果与所有同级项所需要的不同，则可以添加标签以赋予其他权限。这种组合将提供一种更易于理解的方式来管理应用于名称空间的策略。

