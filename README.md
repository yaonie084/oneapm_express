#浅谈oneapm在express项目中的使用
oneapm是一个优秀的性能监控平台。为什么我们要使用性能监控呢？
并不是为了炫耀我有多么酷的玩具，仅仅因为我们希望在问题发生的第一时间就能知道。
在第一时间发现问题，把问题解决于无形之中，总比出了大麻烦通宵达旦加班舒服得多。

然而有的人喜欢说：“有些问题留着也不会有什么影响。”，但我觉得服务端的事情，
凡是冒烟的地方，终究会着火的。

还有的人喜欢说：“我的代码绝对不可能有bug”。不过这只是吹牛逼。

##废话不说了，直接上干货吧。

oneapm的监控服务主要分以下几块

* Application Insight 应用程序监控
* Browser Insight 浏览器客户端监控
* Mobile Insight 移动客户端监控
* Infrastructure Insight 服务器监控

使用oneapm监控自己的项目，首先你需要去 oneapm.com注册一个开发者账号。

##Application Insight 应用程序监控

登录平台以后根据自己项目的语言选择探针，我这里项目是用的express，所以选择了nodejs, 
在oneapm里面对怎么安装探针写得很详细，大概就是在项目的目录下运行
```
    npm install oneapm --registry http://npm.oneapm.com
```
然后配置文件从node_modules/oneapm里面拷出来,改一下License Key，就这么简单。

我们安装好探针以后，过几分钟让插件收集到数据，就能在面板里面看到各种图标。

###首先需要关注的是 响应时间图表

![response time](http://7u2o75.com1.z0.glb.clouddn.com/94DB5DA2-A820-454A-89A0-AF95DB9DA59E.png)

这个图表会对服务端耗时给一个大体印象，大家可以发现我们项目最慢的时候，
是发生在8月18号晚上左右，有请求大约1.25s才结束。紫色的占了绝大多数，
这些都是外部服务消耗的时间。

###右上角的窗口叫做apdex

![apdex](http://7u2o75.com1.z0.glb.clouddn.com/3A0FC92F-334D-418E-9E8E-17DE55AA5701.png)

这是一个评估用户满意度的指标，从这个指标可以看到用户是否满意我们的响应速度，
最右上角有 1[0.5] 可以看到我们100%的用户都满意我们的响应速度，小于0.5秒的请求，
我们称之为满意。我们这里是用的oneapm的默认设置，小于0.5秒表示满意，0.5-2秒是可容忍，
2秒以上则不满意。

###cpm图表
![cpm](http://7u2o75.com1.z0.glb.clouddn.com/48BF1EC4-2AF5-439F-9FB2-47C298A1275C.png)

这个图表代表吞吐量

我们可以看到项目最高的时候，大概每分钟80次请求，平均每分钟17.88次请求。

###web事务图表
![web](http://7u2o75.com1.z0.glb.clouddn.com/72910B87-0407-43A0-B6E9-CDA403221E64.png)

这是一个很重要的图表，在这里我们能看到性能最差的几个web事务，我们通过url，
能找到代码中对应的controller函数，从而找到这个接口中性能的瓶颈

我们来仔细看一个请求吧，第一条express/POST/api/ex...
（鼠标放上去可以显示全部的url, 实际上这一条是这样的 Expressjs/POST/api/exams/signup-all）

我们可以点进去，查看接口的详细情况。

里面有一些仅对这个接口的吞吐量，执行时间等等的图表，具体含义和前面介绍的差不多
，只不过考察的对象变成了唯一这一个接口。

我认为最重要的一个图表是breakdown table

![breakdown](http://7u2o75.com1.z0.glb.clouddn.com/218645DE-E75D-4550-9FA7-B65E4FE2CBC0.png)

这个图表反应了我们这个接口对外部应用（external）,数据库(database)的调用情况。
从图表上可以发现，每次我们调用这个接口，我们会调用37次一个叫做xxxxxtct.com是http协议的
外部服务。执行的时间占到了96.88%，另外查了2次数据库。分别占0.49%和 0.07%

看到这里，咱们就知道怎么优化啦~~拿我这个接口来说，这里的瓶颈主要是卡在发送37次http请求给
xxxxxtct.com这个地方，这个xxxxxtct.com其实是我们自己的一个子系统，如果我在子系统里面
写一个接口，把现在37个请求的内容合并，这个性能问题就完美的解决了。

另外oneapm的Application Insight还给我们提供了，系统拓扑图，按web事务查找瓶颈的功能，按sql查找瓶颈的功能，
外部服务的具体执行时间（这个很重要，看谁在拖我们的后腿）以及后台服务的监控。

###最后说一下错误率这个table，这是我个人的经验

express在抛出系统异常的时候，有可能会挂掉。下面举2个栗子


```
    exports.show = function(req, res) {
      a.b //a == undefined
    }
```
抛出异常

```
    exports.show = function(req, res) {
      request.post({
        url: xxx-service.com
      }, function(err, response, body) {
        a.b //a == undefined
      })
    } 
```
抛出异常,然后服务挂掉。

oneapm是被express程序启起来的，算是express进程的一个子进程，如果express挂掉了，
oneapm也跟着挂了，所以，不可能有机会发回错误信息。
结论是只要在回调里面抛出的异常，任何探针都没有办法收集到错误，
因为在这一层无法做这件事情。

当然，我们虽然有pm2这样优秀的进程管理工具来帮我们，挂掉之后自动重启服务。。。
但我们需要在第一时间获得报错信息啊。。。。即使pm2的errpr.log里面会保留异常，
但谁又会没事专门盯着error这个日志看呢。

针对这个问题，我自己写了一段代码来收集错误日志，希望对大家有帮助。

```
    var pm2 = require('pm2');
    var Slack = require('slack-node');
    
    pm2.launchBus(function(err, bus) {
      console.log('connected');
    
      bus.on('log:err', function(data) {
        var webhookUri = "{你的slack webhook}";
        var slack = new Slack();
        slack.setWebhook(webhookUri);
    
        slack.webhook({
          channel: "#general",
          username: "cq-tct",
          icon_emoji: ":ghost:",
          text: data.data
        }, function(err, response) {
          console.log(response);
        });
      });

    });
```
把这一段保存为 err_notifier.js 放在项目根目录下，每次启完服务之后运行 

node err_notifier.js

这样就能通过slack第一时间收到报错了。即使服务挂掉也能发过来。 

这里用了另一个叫做slack的工具，slack是一款即时通信的办公协作工具，相信大家或多或少都听说过
（就是创业半年估值11亿美元，一年变28亿那个家伙）。国外类似的还有hipchat,国内我不太清楚。

首先去slack申请一个team, 然后创建一个room，为room打开一个webhook, 
把webhook的地址赋值给webhookUri， 这样我们无论在哪里，只要项目报错，就能第一时间
收到通过slack推送过来的错误日志。

当然，你可以把推送的工具改成，hipchat,邮件,短信,这个随大家高兴了。
关于pm2的event monitor，还有更多事情可做，大家可以参考这里
https://github.com/xiaoyang2022/PM2/blob/dadf0f5806536ae95636ac929155c39b8bf030bb/doc/PROGRAMMATIC.md


###最后

oneapm虽然可以帮大家在开发初期铺平道路，但也不意味着因为有了监控就可以胡作非为
（反正项目只要冒烟了，oneapm一目了然）。

我认为保证代码稳定性最好的实践还是:
一个监控系统 + 100%覆盖率的单元测试 + 几套集成测试 + 一套可靠的发布流程