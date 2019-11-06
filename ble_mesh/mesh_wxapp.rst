

初始化Blemanager.js
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

    
      let blueStateCb = {

            onBlueOpened: function (res) {
                that.globalData.bluetoothEnable = true
            },
            onBlueOpenfailed: function (res) {
                that.globalData.bluetoothEnable = false
                that.showToast('请开启蓝牙')
            },
            onDeviceConnected: function (res) {
                that.showToast('已连接')
               connSateChangeCallback.forEach((value, key, map) => {
                    value.onDeviceConnected(res)
                })

            },
            onDeviceDisConnected: function (res) {
                that.showToast('连接断开')
               connSateChangeCallback.forEach((value, key, map) => {
                    value.onDeviceDisConnected(res)
                })
            },
            onDeviceReady: function () {
                    deviceReadyObserver.forEach((value, key, map) => {
                        value()
                    })
            },
            onBlueAdapterEnabled: function (res) {
                that.globalData.bluetoothEnable = true
                // that.showToast('蓝牙已开启')
            },
            onBlueAdapterdisabled: function (res) {
                that.globalData.bluetoothEnable = false
                that.showToast('请开启蓝牙')
            },
            onScanResults: function (res) {
             // 扫描到设备后回调
                scanResultCallback.forEach((value, key, map) => {
                    value(res)
                })
            },
            onDataReceived: function (res) {
                that.globalData.meshApi.parseNotification(res)
            }
        }
    let bleManager = new BleManager(blueStateCb);




开始扫描
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
        
      // 扫描设备 扫描设备前请确保蓝牙已经开启  具体细节请查看demo
      // 扫描未入网设备 uuid ：00001827-0000-1000-8000-00805F9B34FB
      // 扫描已经入网设备 uuid ：00001828-0000-1000-8000-00805F9B34FB
    startScan: function (uuid) {
        if (this.isBluetoothEnable()) {
            return bleManager.startScan(uuid)
        } else {
            this.showToast('请开启蓝牙')
            return new Promise((resolve, reject) => {
                reject(null)
            })
        }

    }
   
连接设备
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

    //这段代码在app.js 文件中
    connect: function (device) {
        function success(res) {
            return new Promise(resolve => {
                resolve(res)
            })
        }

        function fail(res) {
            return new Promise((resolve, reject) => {
                reject(res)
            })
        }

        return this.getbleManager().createBLEConnection(device).then(res => {
            //连接成功并发现服务此时小程序可以跟mesh 设备进行通信
            return success(res)
        }).catch(reason => {
            return fail(reason)
        })

    },


如果连接的是未入网设备（uuid:00001827-0000-1000-8000-00805F9B34FB）连接成功后，可以进行入网设备
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
    
    //例如provisioner.js 部分代码如下显示
    // 注册入网回调
     meshApi.setMeshProvisioningHandler({
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
                that.updateProvisionedInfo({
                    deviceKey: res.provisionBox.deviceKey,
                    unicastAddress: res.provisionBox.unicastAddress
                })
            },
            onReceivedProvisionComplete: function (res) {
               //入网完成
                that.setProvisionState(res)
                that.saveProvisionedNode(res)
                that.disconn()
            },
        })
    //调用如下代码就可以进行入网操作 
       meshApi.startInvite()

入网成功后小程序会自动断开连接并且再次去连接该设备并且获取节点数据信息代码如下
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
:: 

   //以下下步骤都是自动完成的
   onReceivedProvisionComplete: function (res) {
               //入网完成
                that.setProvisionState(res)
                that.saveProvisionedNode(res)
                that.disconn()
            },



                // 入网完毕断开连接并且重新连接
    disconn: function () {
        let that = this
        this.setProvisionState({type: TYPE.OTHER, status: '断开连接中'})
        getApp().disconnect().then(res => {
                that.setProvisionState({type: TYPE.OTHER, status: '连接断开'})
                setTimeout(function () {
                    that.reconnectDevice()
                }, 2500)
            },
        )
    },

    //重新连接设备，
    reconnectDevice: function () {
        let that = this
        this.setProvisionState({type: TYPE.OTHER, status: '连接中'})
        getApp().connect(that.data.curDevice).then(res => {
            getApp().setSelectNode(that.provisonedNode)
            that.setProvisionState({type: TYPE.OTHER, status: '已连接'})
            //获取节点信息数据
            that.sendingComposeDataGet()
        })
    },


  
    sendingComposeDataGet: function () {
        sendMessage(new ConfigCompositionDataGet(this.getCurrentUnicastAddress()))
    },
  
    //注册mesh 消息回调  provisioner.js
    
        meshApi.registerMeshMessageHandler(KEY, function (res) {
                let state
                switch (res.opcode) {
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
                        //收到设备回复的节点信息
                        state = {type: TYPE.RECEIVED, status: 'Receiving CompositionDataStatus'}
                        //发送加解密appkey
                        that.sendingConfigAppKeyAdd();

                        break;
                    case OPCODE.CONFIG_APPKEY_STATUS:
                        //收到设备回复appkey Status 
                        state = {type: TYPE.RECEIVED, status: 'Receive ConfigAppkeyStatus'};
                        if (res.statusMessage.StatusCode == 0) {
                           // 此时设备已经存储appkey 便于后续消息加解密
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
            //配置消息发送完毕 退出当前界面，当前使用到的配置消息如下
          
               // ConfigCompositionDataGet 获取节点信息消息
               // ConfigAddKeyAdd 添加发送开关灯消息加解密appkey
               // ConfigModelAddKeyBind  绑定添加appkey
               // ConfigModelSubscriptionAdd 将灯添加分组（可以将不同的设备加入相同分组）这样就可以实现一个按键控制多个灯
                
                getApp().switchTab('network')
            // 此时已经可以通过小程序来控制设备了
            }

        }

控制设备
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 ::
    
   function initMeshMsgHandler() {
            meshApi.registerMeshMessageHandler(getPageKey(), function (res) {
                switch (res.opcode) {
                    case CONFIG_COMPOSITION_DATA_GET:
                        app.showLoading('loading')
                        break;

                    case CONFIG_NODE_RESET_STATUS:
                        // if connected device  is reset  should  disconnect
                        if (app.getSelectedNode().name === app.getConnDevice().name) {
                            app.disconnect().then(res => {
                                LOG('onConfigNodeResetStatusReceived')('disconnect:' + JSON.stringify(res))
                                app.switchMain()
                            })
                        } else {
                            app.switchMain()
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
                      //收到设备回复OnOffStatus 表示On/off
                      //刷新界面 显示灯亮灭
                        that.updateOnOffModelState(res.statusMessage)
                        break;

                }


            })


        }

    //在nodeConfig.js 中 可以看到如下代码
    //GenericOnOffSetAck 带回复的消息，发送后如果设备收到，设备会回复GENERIC_ON_OFF_STATUS
      genericOnSet: function (e) {
        //打开灯
        this.curElementAddress = e.currentTarget.dataset.data.elementAddress
        sendMessage(new GenericOnOffSetAck(1, app.getMeshConfig().seq_num, this.curElementAddress))
        // this.updateOnOffModelState({mTargetOn:true})
    },
    genericOffSet: function (e) {
      //关闭灯
        this.curElementAddress = e.currentTarget.dataset.data.elementAddress
        sendMessage(new GenericOnOffSetAck(0, app.getMeshConfig().seq_num, this.curElementAddress))
        // this.updateOnOffModelState({mTargetOn:false})
    },


   // 再看一个无回复的消息   group.js 中的部分代码
   //GenericOnOffSetUnAck 代表无回复消息，一般用于控制一系列设备，前提是订阅了该消息的地址，具体的地址可查看创建分组
       groupSendOn: function (e) {
        let that = this
        let address = e.currentTarget.dataset.dst.address
        // this.setOnOffImage(1, address)
        getApp().getMeshApi().sendMeshMessage(new GenericOnOffSetUnAck(1, getApp().getMeshConfig().seq_num, address)).catch(res => {
            if (res.errCode === DEVICE_NOT_CONN) {
                getApp().showToast(res.reason)
                setTimeout(() => {
                    that.route()
                }, 500)
            }
        })
    },
    groupSendOff: function (e) {
        let that = this
        let address = e.currentTarget.dataset.dst.address
        // this.setOnOffImage(0, address)
        getApp().getMeshApi().sendMeshMessage(new GenericOnOffSetUnAck(0, getApp().getMeshConfig().seq_num, address)).catch(res => {
            if (res.errCode === DEVICE_NOT_CONN) {
                getApp().showToast(res.reason)
                setTimeout(() => {
                    that.route()
                }, 500)
            }

        })

    },


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
   getApp().getMeshApi().sendMeshMessage(new ConfigModelSubscriptionAdd(dst, element.elementAddress, subscriptionAddress, modelId)).catch(res => {
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
     getApp().getMeshApi().sendMeshMessage(new ConfigNodeReset(app.getSelectedNode().unicastAddress)).catch(res => {
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
        scanQrcode: function (e) {
        wx.scanCode({
            onlyFromCamera: true,
            scanType: ['qrCode']
            , success(res) {
                let obj = JSON.parse(res.result)
                if (obj.openid) {
                    getApp().updateUserBindInfo(obj.openid, function () {
                    // 扫码成功
                        getApp().switchTab('network')
                    })
                } else {
                    getApp().showToast('数据非法')
                }

            }
        })

    }


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
 .. image:: ./img/create_colection_cloud.png
 