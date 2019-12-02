

初始化  MeshController
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
       //在 app.js 中初始化一下代码
       MeshController.init()
       

设置网络节点数据更新回调,连接状态回调
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

    
      MeshController=getApp().getMeshController()

        MeshController.nodeDataObserver.set(function () {
            let devices = MeshController.nodes()
            //数据更新
            that.setData({devices})
             
            console.log('notify data changed')
        });
       
        MeshController._setConnStateChangelistener(function (res) {
        // 设备连接状态更新回调
            console.log('connected:'+JSON.stringify(res))
            if (res.connected) {
               Loading.showToast('connected');
                that.setData({
                    connected: true,
                    deviceName: res.name,
                    connState: 'disconnect'
                })
            } else {
                Loading.showToast('disconnected');
                that.setData({
                    connected: false,
                    deviceName: 'Bluex Mesh',
                    connState: 'connect'
                })
            }
        });
    // 下拉刷新数据
    MeshController.Provisioner.reSlectNodes();

开始扫描
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
        
     
     //扫描未入网设备
    MeshController.scanUnprovDevice(function (device) {
                    addDevice(device)
                }).then(res => {
                }).catch(reason => {
                 Loading.hideLoading()
                    showError(reason)
                })

    //扫描已入网设备
      let params = {}
                params.cb = function (device) {
                    addDevice(device)
                }
                Loading.showLoading('scanning')
                MeshController.scanProxyNode(params).then(res => {
                }).catch(reason => {
                    Loading.hideLoading()
                    showError(reason)
                })   

   
连接设备
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

       MeshController.stopScan().then(res => {
                Loading.showLoading(title);
                let device =  bluetoothDevice;
                let conn2ProxyNode=that.data.uuid[0]===MESH_PROXY_UUID[0]//是否连接入网的节点
                MeshController.connect({device},conn2ProxyNode).then(res => {
                    //  连接设备成功
                    Loading.hideLoading()
                    switch (that.data.uuid[0]) {
                        case MESH_PROVISION_UUID[0]:
                            wx.navigateTo({
                                url: '../provision/provisioner?device=' + JSON.stringify(bluetoothDevice)
                            })
                            break;
                        case MESH_PROXY_UUID[0]:
                            MeshController.setCurNode(bluetoothDevice.name)
                            wx.navigateBack()
                            break;
                        default:break;
                    }
                }).catch(reason => {
                    Loading.hideLoading()
                    // 连接异常捕获 具体请参考errCode.js中的定义 这里的处理方式 ->重新连接
                  conn('reconnecting')
                    // showError(reason)
                })

            }).catch(reason => {
                showError(reason)
            });

开始入网设备
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
    
    //例如provisioner.js 部分代码如下显示
      //注册入网过程回调
      MeshController.setMeshProvisioningHandler({
            onStartInvite: function (res) {
                // that.sendingProvisionInvite()
                that.setProvisionState(res)

            },
            onReceivedCapabilities: function (res) {
                that.setProvisionState(res)
            },
            onProvisionStart: function (res) {
                that.setProvisionState(res)

            },
            onSendingPublicKey: function (res) {
                that.setProvisionState(res)
            },
            onReceivedPublicKey: function (res) {
                that.setProvisionState(res)

            },
            onSendConfirmData: function (res) {
                that.setProvisionState(res)
            },
            onReceivedConfirm: function (res) {
                that.setProvisionState(res)

            },
            onSendConfirmRandom: function (res) {
                that.setProvisionState(res)
            },
            onReceivedConfirmRandom: function (res) {
                that.setProvisionState(res)

            },

            onSendingProvisionData: function (res) {

                that.setProvisionState(res)
            },
            onReceivedProvisionComplete: function (res) {
                that.setProvisionState(res).disconn()
            },
           })

             //注册Mesh 消息回调（已经入网，后续的消息包括配置消息，OnOff消息）
             MeshController.registerMeshMessageHandler(KEY, function (res) {
                let state
                switch (res.opCode) {
                    case OPCODE.SEG_ACK://sending block ack
                        state = {type: TYPE.WRITE, status: 'Sending BlockAcknowledgement'};
                        break;
                    case OPCODE.SEG_RESENT://resend Segment
                        state = {type: TYPE.WRITE, status: 'Rsending Sgement'};
                        break;
                    case OPCODE.CONFIG_COMPOSITION_DATA_GET:
                        state = {type: TYPE.WRITE, status: 'Sending CompositionDataGet'};
                        break;
                    case OPCODE.CONFIG_APPKEY_ADD:
                        state = {type: TYPE.WRITE, status: 'Sending ConfigAppKeyAdd'};
                        break;
                    case OPCODE.CONFIG_MODEL_APP_BIND:
                        state = {type: TYPE.WRITE, status: 'Sending ConfigModelAppkeyBind'};
                        break;
                    case OPCODE.CONFIG_MODEL_SUBSCRIPTION_ADD:
                        state = {type: TYPE.WRITE, status: 'Sending ConfigSubsctiptionAdd'};
                        break;
                    case OPCODE.CONFIG_COMPOSITION_DATA_STATUS:
                        state = {type: TYPE.RECEIVED, status: 'Receiving CompositionDataStatus'}
                        sendingConfigAppKeyAdd();
                        break;
                    case OPCODE.CONFIG_APPKEY_STATUS:
                        state = {type: TYPE.RECEIVED, status: 'Receive ConfigAppkeyStatus'};
                        if (res.statusMessage.StatusCode == 0) {
                            initwillBindKeyModel(that);
                            nextMessageSend();
                        }
                        break
                    case OPCODE.CONFIG_MODEL_APP_STATUS:
                        state = {type: TYPE.RECEIVED, status: 'Receive ConfigModelAppkeyBindStatus'};
                        nextMessageSend();
                        break;
                    case OPCODE.CONFIG_MODEL_SUBSCRIPTION_STATUS:
                        state = {type: TYPE.RECEIVED, status: 'Receive SubscriptionStatus'};
                        nextMessageSend();
                        break;
                    default:
                        break;
                }
                if (state) {
                   //刷新界面
                    that.setProvisionState(state)
                }
            }
        )

         function nextMessageSend() {
            let msg = that.data.queue.pop()
            if (msg) {
                sendMessage(msg)
                that.pageScrollToBottom();
            } else {
            //  配置消息发送完毕  退出当前界面,
                getApp().switchTab('network')
            }

        }

            //初始化需要绑定appkey,订阅组地址的model
            function initwillBindKeyModel(context) {
               let currentNode = MeshController.getCurNode();
           let dst = currentNode.unicastAddress;
            let queue = context.data.queue;
           let groups = MeshController.getGroups();
           currentNode.elements.map((element, index, self) => {
            element.models.map(model => {
            let modelId = parseInt(model.modelId, 16)
            if (modelId === 0x1000) {
                // model绑定appkey
                let appKeyIndex = 0
                queue.push(new ConfigModelAddKeyBind(dst, element.elementAddress, appKeyIndex, modelId))
                // mdoel 订阅组地址 也就是Groups.js 界面的分组控制OnOff
                let subscriptionAddress = groups.length > 0 ? groups[0].address : 0xc000
                queue.push(new ConfigModelSubscriptionAdd(dst, element.elementAddress, subscriptionAddress, modelId))

            }

        })
    })
  

}



控制设备
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 ::
    

         //注册消息回调
         (function initMeshMsgHandler() {
            MeshController.registerMeshMessageHandler(getPageKey(), function (res) {
                switch (res.opCode) {
                    case CONFIG_COMPOSITION_DATA_GET:
                        break;

                    case CONFIG_NODE_RESET_STATUS:
                        // if connected device  is reset  should  disconnect
                        let isCurNodeReset = MeshController.isCurNodeReset()
                        console.debug('isCurNodeReset：'+isCurNodeReset)
                        if (isCurNodeReset) {
                            setTimeout(res => {
                                MeshController.disconnect().then(res => {
                                    getApp().switchMain()
                                }).catch(reason => {
                                })
                            }, 500)
                        } else {
                            getApp().switchMain()
                        }

                        break;
                    case CONFIG_COMPOSITION_DATA_STATUS:
                        setupNodeInfo(that)
                        break;
                    case CONFIG_APPKEY_STATUS:
                        nextMessage()
                        break;
                    case CONFIG_MODEL_APP_STATUS:
                        nextMessage()
                        break;
                    case CONFIG_MODEL_SUBSCRIPTION_STATUS:
                        nextMessage()
                        break;
                    case GENERIC_ON_OFF_STATUS:
                      //收到设备回复OnOff消息 刷新界面
                        that.updateOnOffModelState(res.statusMessage)
                        break;
 
                }


               })


          })();


            function sendMessage(message) {
             //判断设备是否已连接
                if (MeshController.connected()) {
                MeshController.sendMeshMessage(message).catch(reason => {
                    console.error('sendMessage error:' + reason)
                });
            } else {
               //连接设备
            }
        }
        // 亮灯
         sendMessage(new GenericOnOffSetAck(1, _seqNum(), this.curElementAddress))
         // 灭灯
          sendMessage(new GenericOnOffSetAck(0, _seqNum(), this.curElementAddress))







创建分组  
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

        //group.js 中部分代码 
       onAddGroupClick: function () {
        let that = this
        let isShow = that.data.showModals
        if (!isShow) {
            that.setData({showModals: true})
        }
    }


订阅分组
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 ::
   
   
   //ConfigModelSubscriptionAdd
   MeshController.sendMeshMessage(new ConfigModelSubscriptionAdd(dst, element.elementAddress, subscriptionAddress, modelId)).catch(res => {
            if (res.errCode === DEVICE_NOT_CONN) {
                getApp().showToast(res.reason)
                setTimeout(() => {
                    that.route()
                }, 500)
            }
        })
    




移除节点（使设备状态恢复初始状态，也就是未入网设备）
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
   
    //ConfigNodeReset
    MeshController.sendMeshMessage(new ConfigNodeReset(app.getSelectedNode().unicastAddress)).catch(res => {
            if (res.errCode === DEVICE_NOT_CONN) {
                getApp().showToast(res.reason)
                setTimeout(() => {
                    that.route()
                }, 500)
            }
        })
    
           
云函数使用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
   
    //例如 将入网设备存储至云端   CloudfuncController.js 部分代码如下显示
    let CloudController = require('./CloudfuncController').getInstance()
    //每一个用户拥有唯一的openid,小程序云自动生成的无需创建
    CloudController.insertNode(node,openid)
    //其它云函数具体使用请查看Demo中的使用
    


扫码共享当前已经入网数据
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
  
        
        // qrcode.js 中的部分代码
      wx.scanCode({
            onlyFromCamera: true,
            scanType: ['qrCode']
            , success(res) {
                let obj = JSON.parse(res.result)
                if (obj.openid) {
                   getApp().getMeshController().bindUser(obj.openid).then(res=>{
                       getApp().switchTab('network')
                    })
                } else {
                }

            }
        })
    }


log 输出
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
       
       MeshController.setLogger({
            DEBUG:function (tag,info) {
                console.debug(tag+'\n'+info)
            },
            ERROR:function (tag,info) {
                console.error(tag+'\n'+info)
            }
        });








关闭数据及时刷新监听   具体使用细节请查看小程序文档云函数使用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

    MeshController.CloudController.closeNodesWatch()
        MeshController. CloudController.closeGroupWatch()
        MeshController.CloudController.closeProCfgWatch() 

           
 
云函数定义
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
    
      //以下是插入节点到云端代码    路径wxapp_blemesh/cloud/insertNode.js
     // 云函数入口文件
    const cloud = require('wx-server-sdk')

    cloud.init()
    let db = cloud.database()
    // 云函数入口函数
    exports.main = async (event, context) => {
    const wxContext = cloud.getWXContext()
    event._openid = event.openid
    //插入数据之前查询数据库中是否存在该节点
    let rsl = await db.collection('provisioned_nodes').where({name: event.node.name}).get()
    if (rsl.data&&rsl.data.length>0) {
        let provisionedNode = rsl.data[0]
        //移除已存在节点
        await db.collection('provisioned_nodes').where({name: provisionedNode.name}).remove()
        //移除on_off_model_state 表中对应记录
        await  db.collection('on_off_model_state').where({name: provisionedNode.name}).remove()
    }

    return await db.collection('provisioned_nodes').add({
        data: event.node
    }) 
     }

    
创建云函数数据表
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 .. image:: ./img/create_colection_cloud.png
 

在云控制台添加集合
::
  
   groups
   on_off_model_state
   provision_config
   provisioned_nodes
   src
   user
   
   
   





导入默认的入网配置
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

   集合创建完毕后需要在集合“provision_config”导入一份默认的入网配置json文件
   步骤1：将以下json文本拷贝至文本文件中，然后重命名为xxx.json
   {"_id":"5ac1f101-8096-4e3d-a3ff-7d229f309ca6","unicastAddress":"0001","src":32767.0,"networKey":["8b19ac31d58b124c946209b5db1021b9"],"appKeys":["000102030405060708090A0B0C0D0E0F"],"keyIndex":"0000","flags":"00","ivIndex":"00000000","seq_num":0}

   步骤2 在云开发控制台中选中provision_config 集合,点击导入，选择之前重命名的xxx.json 这样默认的入网配置就算完成了











常见问题
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
1  初次使用时如果android手机版本大于等于5.0，请务必开启手机定位权限，也就是在手机设置中开启微信定位权限，不然会造成无法扫描到mesh 设备,

2  如发现扫描不到设备，请确认是否开启蓝牙

3  如果发现小程序显示一直在连接设备，或者连接设备失败，请尝试退出当前页面，再次扫描设备进行连接操作.
 
 * **[Q]** 入网过程中卡在以下界面?

 .. image:: ./img/wxapp_interrupt.jpg


 * **[A]** 请查看以下视频

 * https://www.ixigua.com/i6765790799360688648/

   



