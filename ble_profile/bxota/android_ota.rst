
Android OTA 流程 分为两个阶段
#############################
  
 
第一阶段
***************
   #. 连接设备。
   #. 读取设备服务列表。
   #. 发现服务（gatt.discoverServices()）。
   #. 发现相关特征值（ctrlChar ，dataChar）。
   #. 写入ctrlChar request描述符。
   #. 设备响应ctrlChar response描述符。
   #. 手机发起OTA请求（wirite characteristic）。
   #. 设备响应OTA请求（到此OTA 开始进入准备升级状态）。
   #. 以上是OTA准备状态。
   
第一阶段流程图如下所示
************************
   
 .. image:: ../img/BXOTAAndroid_READY.png
   
  
   
第二阶段
***********
  #.  手机选择文件。
  #.  把选择的文件分成若干个block（一个block由若干个segment，手机一次写入一个segment）。
  #.  使用ctrlChar写入描述符开始请求当前block write 
 
  #.  设备回复onCharacteristicWrite 

      #. ctrlChar>

       #. 00> OTA request
       #. 02> new block start>OTATransferContinue(true)
       #. 03> OTA  complete

      #. dataChar>

       #. OTATransferContinue(false); 
   

  #.  当写入当前block的最后一个segment完成后，手机请求gatt.read(dataChar)当前block的segment写入状态。
  #.  设备响应当前block写入状态。

  #.  判断获取的byte[]数据,每个字节代表每个segment写入的状态。

    #. 如果全部写入成功。
    #. 判断当前的block是否是最后一个block。
    #. 如果是，说明所有的block写入完成，此时手机发送写入数据完成请求，设备回复写入数据完成后，说明整个OTA  image写入成功。
    #. 如果不是，说明还有block没有写入，此时开启新的block write 

  #. 如果部分segment写入失败，继续写入失败的 segment  直到全部写入成功
   
第二阶段流程图如下所示
*************************
   
 .. image:: ../img/OTA_tansport.png
   
OTA涉及相关代码
*******************
.. code:: java

    <?java   
   //连接设备
        public void connect(Context context) {
          if (mDecvice != null) {
       mDecvice.connectGatt(context, false, this);}
       }
  
 
  
   //连接设备后回调 
     @Override
    public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
        super.onConnectionStateChange(gatt, status, newState);
        if (newState == BluetoothProfile.STATE_CONNECTED) {
            Log.d(BXOTA_CLIENT_CALLBACK_TAG, "onConnectionStateChange->STATE_CONNECTED");
            gatt.discoverServices();//发现服务
          
        } 
    }
    
    
    
  // 读取设备服务列表，
   
    public void onServicesDiscovered(BluetoothGatt gatt, int status) {
        super.onServicesDiscovered(gatt, status);
        Log.d(BXOTA_CLIENT_CALLBACK_TAG, "onServicesDiscovered:" + status);
        bindGattBXOTAService(gatt);//绑定服务
       
    }
     
  // 绑定服务 
     void bindGattBXOTAService(BluetoothGatt gatt) {
        Log.d(TAG, "bindGattBXOTAService: ");
        if (gatt != null) {
            mBXOTAService = gatt.getService(BXOTA_SERVICE_UUID);
            if (mBXOTAService != null) {
                mBluetoothGatt = gatt;
                mCharsList = mBXOTAService.getCharacteristics();
                ctrlChar = mCharsList.get(BXOTA_CHAR_CTRL_INDEX);
                ctrlChar.setWriteType(BluetoothGattCharacteristic.WRITE_TYPE_DEFAULT);
                mBluetoothGatt.setCharacteristicNotification(ctrlChar, true);
                dataChar = mCharsList.get(BXOTA_CHAR_DATA_INDEX);
                dataChar.setWriteType(BluetoothGattCharacteristic.WRITE_TYPE_NO_RESPONSE);
                mBluetoothGatt.setCharacteristicNotification(dataChar, true);
                enableCtrlIndication();
              


            }
        }
    }
    
    //写入描述符
     void enableCtrlIndication() {
        BluetoothGattDescriptor ctrlDesc = ctrlChar.getDescriptor(CCCD);
        ctrlDesc.setValue(BluetoothGattDescriptor.ENABLE_INDICATION_VALUE);
        mBluetoothGatt.writeDescriptor(ctrlDesc);
        Log.d(TAG, "bindGattBXOTAService: writeDescriptor..");
    }
    
   // 设备响应描述符写入
    
     public void onDescriptorWrite(BluetoothGatt gatt, BluetoothGattDescriptor descriptor, int status) {
        super.onDescriptorWrite(gatt, descriptor, status);
        if (status == gatt.GATT_SUCCESS) {
            BluetoothGattDescriptor ctrlDesc = ctrlChar.getDescriptor(CCCD);
            if (descriptor.getUuid().compareTo(ctrlDesc.getUuid()) == 0) {
                isBound = true;
                //ctrlDesc 描述符写入成功 也就是服务绑定成功
               //startOtaRequest();
            }
        }

    }
    
    
 // 发起OTA升级请求
    
     public void startOtaRequest() {
        if (isBound) {
            byte[] maxBlockDataSizeArray = new byte[]{(byte) maxSegmentDataSize, (byte) (maxSegmentDataSize >> 8)};
            ctrlPktTX(BXOTA_CTRL_PKT_START_REQ, maxBlockDataSizeArray);
        } else {
            Log.w(TAG, "startOtaRequest:  service not bound please bound first");
        }
    }
    
    
    
    //设备响应OTA请求
    
    
    void ctrlPktIndicationRX(byte[] rxData) {
        switch (rxData[0]) {
            case BXOTA_CTRL_PKT_START_RSP:
                int status = rxData[1];
                maxSegmentNumInBlock = rxData[2] * 8;
                currentAck = new boolean[maxSegmentNumInBlock];
                if (status == 0) {
                    boolOtaReady = true;
                    mIbxotaListener.onOtaRequeSuccess();
                }
                break;
        }
    }
    
  
    





        
    //把选择的文件分成若干个block
    
    /**
     * 选择image开始升级
     *
     * @param data  文件转成的byte
     */
    public void startOTATransfer(byte[] data) {
        if (isOtaReady()) {
            OTAData = data;
            blockNum = (int) Math.ceil((double) OTAData.length / (maxSegmentDataSize * maxSegmentNumInBlock));
            int lastBlockDataLength = OTAData.length % (maxSegmentDataSize * maxSegmentNumInBlock);
            lastBlockSegmentNum = (int) Math.ceil((double) lastBlockDataLength / maxSegmentDataSize);
            currentBlock = 0;
            newBlockCmd();
        } else {
            Log.w(TAG, "startOTATransfer:  ota is not  ready");
        }
    }






    
     //使用ctrlChar开始新的segment write。
     
      private void newBlockCmd() {
        byte[] blockIDArray = new byte[]{(byte) currentBlock, (byte) (currentBlock >> 8)};
        Log.d(TAG, "newBlockCmd: " + Utils.bytesToHex(blockIDArray, false));
        ctrlPktTX(BXOTA_CTRL_PKT_NEW_BLOCK_CMD, blockIDArray);
        }

      private void ctrlPktTX(int type, byte[] param) {
        byte[] ctrl;
        if (param != null) {
            ctrl = new byte[param.length + 1];
            System.arraycopy(param, 0, ctrl, 1, param.length);
        } else {
            ctrl = new byte[1];
        }
        ctrl[0] = (byte) type;
        ctrlChar.setValue(ctrl);
        Log.d(TAG, "ctrlPktTX: "+Utils.bytesToHex(ctrl,false));
        mBluetoothGatt.writeCharacteristic(ctrlChar);

    }

   // 设备回复ctrlChar,dataChar write
      public void onCharacteristicWrite(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic, int status) {
        super.onCharacteristicWrite(gatt, characteristic, status);
       
        if (status == gatt.GATT_SUCCESS) {
            if (characteristic.getUuid().compareTo(BXOTAClient.BXOTA_CHAR_CTRL_UUID) == 0) {
                byte[] txBytes = characteristic.getValue();
                ctrlPktSent(txBytes);
            }
            if (characteristic.getUuid().compareTo(BXOTAClient.BXOTA_CHAR_DATA_UUID) == 0) {
                OTATransferContinue(false);
            }
        }
      

    }

    void ctrlPktSent(byte[] txData) {
        switch (txData[0]) {
            case BXOTA_CTRL_PKT_START_REQ:
                Log.d(TAG, "ota ready: ");
                break;
            case BXOTA_CTRL_PKT_NEW_BLOCK_CMD:
                Arrays.fill(currentAck, false);
                OTATransferContinue(true);
                break;
            case BXOTA_CTRL_PKT_IMAGE_TRANSFER_FINISH_CMD:
                Log.d(TAG, "ota  complete....: ");
                mIbxotaListener.onComplete();
                break;
        }
    }
    
        
      
    
    
    
     void OTATransferContinue(boolean newBlock) {
        int segmentNum = getSegmentNumOfCurrentBlock();
        if (newBlock) {
            currentSegment = 0;
        } else {
            ++currentSegment;
        }
        while (currentSegment < segmentNum && currentAck[currentSegment]) {
            ++currentSegment;
            Log.d(TAG, "OTATransferContinue while " + currentSegment);
        }
        // dataChar 写完block最后一个segment
        if (currentSegment == segmentNum) {
            Log.d(TAG, "OTATransferContinue: current block write complete >>>>>read current block transport state");
            readAck();//读取当前block 写入情况

        } else {
            segmentTX();
        }

    }
    
    void readAck()
    {
        mBluetoothGatt.readCharacteristic(dataChar);
    }
    
   // 响应当前block readAck
    void onAckRead(byte[] rxBytes) {
        boolean allAcked = true;
        int segmentNum = getSegmentNumOfCurrentBlock();
        for (int i = 0; i < segmentNum; ++i) {
            //check all data send
            if ((rxBytes[i / 8] & (1 << i % 8)) != 0) {
                currentAck[i] = true;
            } else {
                // some pack data transfor filed
                allAcked = false;
                currentAck[i] = false;

            }
        }
        if (allAcked) {
            double progress = (double) (currentBlock + 1) / blockNum;
            mIbxotaListener.onUpdataProgress((float) (progress * 100));
            if (++currentBlock == blockNum) {
                Log.d(TAG, "all image trans success");//所有数据传输成功
                imageTXFinishCmd();
            } else {
                Log.d(TAG, "begining transport next block");
                newBlockCmd();//开启新的block
            }
        } else {
            //tranfor failed >>continue transfor
            Log.d(TAG, "some segment transport failed trying retransport ");
            OTATransferContinue(true);//传输failed segment  
        }

    }
   // image 写入完成 write request
      private void imageTXFinishCmd() {
        ctrlPktTX(BXOTA_CTRL_PKT_IMAGE_TRANSFER_FINISH_CMD, null);
    }
  ?>

关于BXOTAClient使用
************************

.. code:: java

  <?java
   //在Activity 中初始化
    protected void onCreate(final Bundle savedInstanceState) {
      BluetoothDevice mDevice;
        mBxotaClient = new BXOTAClient(this,mDevice);
      
     }
    ?>

BleCallback
*******************
 * 这个类可以实现OTA 状态监听外，还能自由处理蓝牙回调数据，进行其它操作

IBxotaListener
************************
 * 这个类主要实现监听OTA状态,