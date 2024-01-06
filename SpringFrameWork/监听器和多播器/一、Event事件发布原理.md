# IOC容器刷新结束，会发布ContextRefreshedEvent事件

首先，我们在自己编写的监听器（例如MyApplicationListener）内的onApplicationEvent方法处打上一个断点，如下图所示。很明显，现在我们看到的是收到的第一个事件，即ContextRefreshedEvent事件。那么问题来了，我们是怎么收到的该事件呢？
![[Pasted image 20240106213324.png]]

我们不妨从IOCTest_Ext测试类中的test01方法，从头开始，来梳理一遍整个流程。鼠标单击IDEA调用栈左下角中的IOCTest_Ext.test01() line:22，
![[Pasted image 20240106213353.png]]

这时程序来到了IOCTest_Ext测试类的test01方法中，如下图所示。
![[Pasted image 20240106213447.png]]

继续跟进代码，当容器刷新完成时，就会调用finishRefresh方法，容器刷新完成时调用publishEvent，而且传递进该方法的参数是new出来的一个ContextRefreshedEvent对象。这一切都在说明着，容器在刷新完成以后，便会发布一个ContextRefreshedEvent事件。
![[Pasted image 20240106213659.png]]

# 用户自定义发布的事件

Resume Program F9，再次来到回调方法处，可以看到我们自己发布的事件
![[Pasted image 20240106214544.png]]

# 容器最后关闭，会发布ContextClosedEvent事件

Resume Program F9，再次来到回调方法处，可想而知，就应该是要轮到最后一个事件了，即容器关闭事件。
![[Pasted image 20240106214901.png]]
下面取消此处的断点，在容器关闭的位置打上新的断点
![[Pasted image 20240106215009.png]]
stepinto，发现调用了doClose方法
![[Pasted image 20240106215041.png]]
最终发现在doClose方法中，发布了ContextCloseEvent事件
![[Pasted image 20240106215103.png]]
