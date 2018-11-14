==================================
 Mesh  已经实现的功能
==================================


协议栈基础功能
""""""""""""""""""""""""""""""""""""

* 基础消息控制：Bear层到Access层消息加解密与处理。
* Provision入网：支持ADV入网和GATT入网，但是同时只能二选一
* Relay 
* Proxy ： 实现手机APP连接控制
* IV index Update
* 节点信息保存：Provision完毕之后，断电不会丢失入网信息。
* 节点复位：APP发送node reset消息，让节点重新恢复 Unprovision Device 状态。



Configuration 配置消息
""""""""""""""""""""""""""""""""""""

* Config Composition Data  类消息
* Config Relay  类消息
* Config Model Publication  类消息
* Config Model Subscription  类消息
* Config NetKey  类消息
* Config AppKey 类消息
* Config Model App 类消息
* Config Node Reset 类消息
* Config Key Refresh Phase 类消息


已经实现的 Model 
""""""""""""""""""""""""""""""""""""

* Config Server/Client
* Generic OnOff Server/Client
* Generic Level Server/Client [仅Generic Level Set/Get/Status]




板子端实例应用
""""""""""""""""""""""""""""""""""""

下述example，板子作为Provisioner + Config Client。

* Provisioner + Config Client + Generic OnOff Client 
* Provisioner + Config Client + Generic Level Client

下述example，板子为 Unprovision Device + Config Server。

* Config Server + Generic Level Server
* Config Server + Generic OnOff Client
* Config Server + Generic OnOff Server


手机端APP
""""""""""""""""""""""""""""""""""""

* Unprovision Device的扫描，入网，分发Key与地址。
* Add APPKEY (Provision后会自动进行)
* Composition Data (自动获取目标板子内部的model列表，Provision后会自动进行)
* Bind AppKey
* Generic OnOff Client Model 控制台
* Generic Level Client Model 控制台
* Subscription 订阅分组地址，实现分组控制
* APP 控制 Relay 开关设置
* 节点信息存储，退出APP不需要重新配置
* 数据备份：可以将APP中的入网信息，通过SD卡备份与恢复。

 





























