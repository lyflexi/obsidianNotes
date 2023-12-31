RBAC（Role-Based Access Control），基于角色的访问控制。这是一套成熟的权限模型。
在传统权限模型中，我们直接把权限赋予用户，当用户量激增的时候，授权工作量会非常大。
![[Pasted image 20231225101613.png]]
而在RBAC中，增加了“角色”的概念，我们首先把权限赋予角色，再把角色赋予用户。这样，由于增加了角色，授权会更加灵活方便。在RBAC中，根据权限的复杂程度，又可分为RBAC0、RBAC1、RBAC2、RBAC3。其中，RBAC0是基础，RBAC1、RBAC2、RBAC3都是以RBAC0为基础的升级。我们可以根据自家产品权限的复杂程度，选取适合的权限模型。
==记住，多对多关系要新增一张关系表，并且关系表中要设置联合索引。因此，与RBAC相关联的数据库表至少有5张==
# RBAC0
譬如我们做一款企业管理产品，如果按传统权限模型，给每一个用户赋予权限则会非常麻烦，并且做不到批量修改用户权限。这时候，可以抽象出几个角色，譬如销售经理、财务经理、市场经理等，然后把权限分配给这些角色，再把角色赋予用户。这样无论是分配权限还是以后的修改权限，只需要修改用户和角色的关系，或角色和权限的关系即可，更加灵活方便。此外，如果一个用户有多个角色，譬如王先生既负责销售部也负责市场部，那么可以给王先生赋予两个角色，即销售经理+市场经理，这样他就拥有这两个角色的所有权限。
![[Pasted image 20231225101737.png]]
# RBAC1

基于之前RBAC0的例子，我们又发现一个公司的销售经理可能是分几个等级的，譬如除了销售经理，还有销售副经理，而销售副经理只有销售经理的部分权限。这时候，我们就可以采用RBAC1的分级模型，把销售经理这个角色分成多个等级，给销售副经理赋予较低的等级即可。
![[Pasted image 20231225102045.png]]
# RBAC2

RBAC2进一步对用户、角色和权限三者之间增加了一些限制。这些限制可以分成两类，即静态职责分离SSD(Static Separation of Duty)和动态职责分离DSD(Dynamic Separation of Duty)。

![[Pasted image 20231225102118.png]]
还是基于之前RBAC0的例子，我们又发现有些角色之间是需要互斥的，
- 譬如给一个用户分配了销售经理的角色，就不能给他再赋予财务经理的角色了，否则他即可以录入合同又能自己审核合同；
- 再譬如，有些公司对角色的升级十分看重，一个销售员要想升级到销售经理，必须先升级到销售主管，这时候就要采用先决条件限制了。
# RBAC3
基于RBAC模型，还可以适当延展，使其更适合我们的产品。譬如，
1. 我们可以把一个部门看成一个用户组，如销售部，财务部，==提前给这个部门赋予角色==，使部门拥有部门权限，
2. 后面再把用户加入用户组，这样这个部门的所有用户都有了部门权限。
这样用户除了拥有自身的权限外，还拥有了所属用户组的所有权限。
![[Pasted image 20231225102320.png]]

>无论是本次的权限模型，还是其他产品相关实现方案，很多都已经被前人所总结提炼。站在巨人的肩膀上我们可以看得更远，而不是再造一个轮子。