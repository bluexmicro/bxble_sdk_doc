=================
 Mesh  Q & A 
=================

\-----------------------------------------------------------   

* **[Q]** Does current firmware support multiple relay network?
      [Smartphone] --> [A] --> [B] --> [C]
* **[A]** Support
       .. image:: mesh_QA_img/mesh3node.png

\-----------------------------------------------------------   

* **[Q]** APP它会自己识别model的function，来产生 ON OFF Controls的界面？

* **[A]** 是的，在入网完毕止之后会发送 get compositoin data消息，来获取目标设备都有那些model，从而配置哪些控制界面。

\-----------------------------------------------------------   

* **[Q]** relay role也是可以通过 config model来配置的吧， 好像也有config model可以修改它？这个是纯消息，还是也是一个config model?

* **[A]** 1：relay和model无关，是configuration message，算是节点的一个属性。可以在代码中设置。我们后续发出的新版APP可以支持在手机配置relay功能。建议你可以看一下Mesh Profile里面有一些消息的介绍。2：所有configuration消息都是Foundation models 层，都是发给config server/client来进行处理的。

\-----------------------------------------------------------   

* **[Q]** gatt proxy节点（也需要relay feature吧?）

* **[A]** proxy节点自动开启relay。

\-----------------------------------------------------------   

* **[Q]** 一个设备也可以同时支持 gatt bearer和adv bearer吧？

* **[A]** 1：入网的时候不可以同时。通过宏配置选择ADV还是GATT入网。2：入网完毕之后可以接受ADV消息。如果配置过GATT那么既可以接受ADV消息又可以手机连接GATT。

\-----------------------------------------------------------   

* **[Q]** sprintf()打印好像没有输出？ 是工程没配置，还是别的什么问题。。。

* **[A]** sprintf是打印到字符串，printf需要配置fputc。我们的RTT LOG用法和printf用法一样的。串口速度慢，debug不方便的。建议你使用RTT。如果非要用串口，你需要配置一下fputc函数。

\-----------------------------------------------------------   

* **[Q]** nRF connect可以看到设备，但是nrf mesh看不到设备。

* **[A]** sdk mesh config里面有一个GATT 的宏需要打开。不然是adv入网。

\-----------------------------------------------------------   

* **[Q]** 蓝牙的mac地址怎么修改？

* **[A]** 在NVDS里面修改。bx_sys_config.h里面的BX_DEV_ADDR。可以通过量产工具批量修改。

\-----------------------------------------------------------   

* **[Q]** May I ask how to reset mesh network non-volatile parameters?

* **[A]** You should use JFlash to erase full chip. Our new Mesh SDK will support node reset message , you can reset node from nRF Mesh APP.

\-----------------------------------------------------------   

* **[Q]** How to turn on relay feature?

* **[A]** You can refer this file "SDK_v1_0_2325\\freertos\\app\\mesh\\examples\\simple_generic_onoff_server\\mesh_app_hal.c" ,the function "hal_button4_cb" is open the relay feature.

\-----------------------------------------------------------   

* **[Q]** 如何提高多个节点传输的速度？

* **[A]** 主要和 transmite count 和 interval 两个参数有关，需要根据实际情况调试这两个参数。

* 首先介绍一下transmite count 和 interval 两个参数。
* 该参数定义在example目录下，“sdk_mesh_config_pro.h”文件中。


.. code:: c

    #define TRANSMIT_DEFAULT_COUNT                  5   ///send 6 times  (count + 1)
    #define TRANSMIT_DEFAULT_INTERVAL               10  ///interval = step*10 ms
    #define TRANSMIT_DEFAULT_RELAY_COUNT            6   ///send 7 times
    #define TRANSMIT_DEFAULT_RELAY_INTERVAL         10  ///interval = step*10 ms

* 其中COUNT结尾的参数，就是一个ADV包需要重复的次数。数值为6就意味着重发5次，一共发送6次。
* 其中INTERVAL结尾的参数，就是一个两个ADV数据包之间的时间间隔，单位是10ms，参数为10就意味着100ms发送一次。（系统会在此基础上增加0-40ms额外随机延时。）
* 队列中的ADV数据包会排队发送，第一个发完之后去发送第二个。以此类推。
* TRANSMIT_DEFAULT 参数为本节点自身发出去的ADV数据包的特性。
* TRANSMIT_DEFAULT_RELAY 参数为本节点relay出去的ADV数据包的特性。
* 重发次数越多，丢包率越低，重发次数越少，丢包率越高。
* 如果想加快发送速度，可以减小这四个参数的数值，可以缩短总的发送时间，让发送速度更快。但是代价就是重发次数少，会导致丢包率增加。

\-----------------------------------------------------------   

* **[Q]** 如何判断是否要开启relay功能？

* **[A]** 通常来说，节点越密集，relay节点可以越少。节点越分散，relay节点越多。需要根据实际应用场景去调试是否开启relay功能。

\-----------------------------------------------------------   





































