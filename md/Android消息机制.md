## Android消息机制

Android消息机制，代码就不详细分析，都在提交的记录里面，这里简单介绍一下流程
1、在目标线程中通过Looper.prepare()方法创建Looper实例，创建Looper实例过程创建MessageQueue

2、调用Looper.loop()方法，不断的从MessageQueue取Message处理

3、在目标线程中创建Handler实例，一般会覆盖Handler中的handleMessage方法，用于处理Message，默认构造方法会获取当前线程中的Looper,另外也可以使用Looper创建Handler方法

4、工作线程中通过Handler.obtainMessage()获取Message

5、通过Handler.sendMessage()发送Message，Message放入到MessageQueue中

6、Looper.loop()方法中获取到Message，就执行Message.target.dispatchMessage()方法

7、Handler.dispatchMessage()方法调用handleMessage或者handleCallback方法

详细可以参考另外一文章
[Android 消息处理机制（Looper、Handler、MessageQueue,Message）](https://www.jianshu.com/p/02962454adf7)

放出两张图
![](https://upload-images.jianshu.io/upload_images/966283-3e81643b7e2604ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

![](https://upload-images.jianshu.io/upload_images/966283-bd9c24f19114108a.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)
