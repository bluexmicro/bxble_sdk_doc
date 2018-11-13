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

* **[A]** You can refer this file "SDK_v1_0_2325\freertos\app\mesh\examples\simple_generic_onoff_server\mesh_app_hal.c" ,the function "hal_button4_cb" is open the relay feature.

\-----------------------------------------------------------   








































