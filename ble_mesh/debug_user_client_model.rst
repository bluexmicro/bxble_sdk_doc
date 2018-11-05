=================================
上位机调试新user_client_model流程
=================================

对于上位机没有实现的 user_client_model ，需要通过debug窗口手动发送调试指令，进行手动配置。debug窗口进行配置的时候，需要有五个步骤。


****************************
第一步：入网
****************************

**注：该步骤在图形界面中完成。**

provisioner入网完毕。

****************************
第二步：Add APPKEY
****************************

**注：该步骤在图形界面中完成。**

Add APPKEY完毕。


******************************
第三步：bind client 的appkey
******************************

**注：该步骤在debug窗口中完成。**

采用bind_appkey_to_model_by_element函数，对 user_client_model 进行绑定APPKEY。

使用方法：

.. code:: c

    err_t bind_appkey_to_model_by_element(uint8_t elem_idx,uint32_t model_id,uint16_t appkey_idx)


=============== ===============================================================
参数名                 参数介绍
=============== ===============================================================
elem_idx：           user_client_model 所在的element
model_id：           user_client_model 自己的ModelID
appkey_idx:          user_client_model 需要绑定哪一个APPKEY的global index。
=============== ===============================================================




*******************************
第四步：bind server 的appkey
*******************************

**注：该步骤在debug窗口中完成。**

采用config_model_app_bind_tx函数，发送《Config Model App Bind》消息，让对面的user_server_model去绑定AppKey。

使用方法：

.. code:: c

    void config_model_app_bind_tx(model_base_t *model,config_model_app_bind_param_t *param,uint16_t dst_addr,void (*cb)(access_pdu_tx_t *,uint8_t))

=============== ==================================================================================================
参数名                 参数介绍
=============== ==================================================================================================
model：              config_client的指针，可以使用"&get_config_client()->model.base"获取
param：              发送消息的内容，有elmt_addr、appkey_idx、model_id、sig_model四个成员需要填写。
dst_addr：           需要发送的目标地址，即对面的user_server_model所在的元素的地址。
cb：                 发送完成回调函数。
=============== ==================================================================================================



****************************
第五步：发送APP控制消息
****************************

**注：该步骤在debug窗口中完成。**

发送app控制消息。

用户可以调用自己实现的user_client_model的具体发送函数，去发送消息。可以参考《generic_onoff_set_tx》函数中的内容。






























    
    