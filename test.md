#2017 年工作报告

##清算对账平台
###时间
8月 ~ 11月

###工作
2017年清算对账平台主要工作是二期建设，从平台性质来看，我们将一个被动展示我行各系统对账结果的门户，变为了一个能自主监控对账情况的平台。
我们迈出了较小但较为成功的一步：余额为零内部账户的自动监控、其它内部账户对账情况自动跟踪、核心新开内部账户自动拉取、内部账户管理。

###思考
通过对接大数据平台，积累了一套好用的导数工具(HiveToMysql)：支持字段映射，部分字段导入，多对一导入，常量设置，重复导入等。
解决了行内导数使用 sqoop 存在的限制：只能同构表全部导入(不兼容于将来可能的字段精细化授权)，bash 脚本难以维护且复用度不高。

###展望
目前该平台已经在 docker 平台上部署，渴望这项技术能早日上生产，引导我积累更多的先进经验和技术！
我相信，未来清算对账平台要做的事情还有更多，我们赋予它越多的主动性它就能为我行在清算、对账方面提供越大的便捷性。

##实体卡
###时间
5月 ~ 8月， 10 ~ 12月
###工作
完成了实体卡平台二期从零开始的搭建，业务开发，优化。
实体卡二期大的业务场景：预制卡、新客户发卡、已有一类二类户发卡、补换卡/token。
涉及到的流程较复杂：申请、制卡、入库、外呼、呼入、核身任务自动调度、上门核身、二次核身、账户升级、卡激活。
对接系统：核心，卡管，ecif，安全，外呼、呼入平台，HJ，核身APP；本系统提供服务接口约30个，调用约20个

###思考：
通过从零开始搭建这个系统到完成这些较多，较复杂的业务，使自己在架构能力上了一个层次，对架构有了一些初步认识：一个成熟的系统并非借助于一个优良的开源框架就可以完成，而是要结合我们的业务，将其合理模块化，理清联系与边界，不停思考，不停改良，方可慢慢达到较好的程度。

###展望：
随着业务发展，实体卡全面推广，为节省发卡成本，实体卡平台将来有一个非常重要的优化方向：如何就近安排上门人员？如何确定更低成本的发卡时间？对哪些地区的人们邀请？
这将涉及到”发卡任务，上门人员，客户，时间，地点“这五个维度的调度。
目前来看，可能的优化方向有两个：
1.从算法本身入手，研究对比，找到更优良的调度算法 
2.从数据入手，待沉淀较多数据，使用机器学习(如分类，推荐等)，训练出一个较好的调度任务模型

##有折
###工作
- 2月-3月: 负责卡券管理(基于新框架开发实时发券，动账通知，回收，核对)
- 3月-5月：有折商户端子系统，负责以下功能：用户及权限管理(收银员，管理员)，商户门店层级管理，登录及更改密码，收银明细查询，统计报表，与CCS对账
- 11月: 负责腾讯视频接入

###思考
短短两个多月，完成了有折商户端上面所有功能，历经了挑战，但更感收获颇丰，get 到很多新技能：
- 公众号开发流程
- 加密机，加密算法，消费类加解密行业标准做法
- 支付码生成与扫描
- 安全漏洞：csrf与xss攻击
这些都是非常有趣且实在的知识，边做边学是一种有意思的体验！希望将来有更多机会迎接这样的挑战！

后面花了三周左右时间完成了腾讯视频接入，联调测试，上线的工作，虽然功能不多，但考虑到是纯线上交易，于是做了较完善的容错处理，性能优化，日志。还是花了一番苦功，也乐在其中！希望这个模块真正做到小而精悍！

##2017年总结
走过2017，做了较多事情，学了很多东西，经历了一些挑战，还经历了有折元旦生产事故，至今仍历历在目！它们提醒了我：不仅业务，代码能力要跟上，而且测试，性能分析，业务推广策略无一不相当重要！它们将成为我的经验，作为以后的指导，时时鞭策自我：勤练业务，勤学技术，关键时刻要顶住！



