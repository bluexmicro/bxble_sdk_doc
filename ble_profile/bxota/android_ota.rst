
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

       #. 00 -> write OTA request
       #. 01 <- received OTA response
       #. 02 -> write new block start>OTATransferContinue(true)
       #. 03 -> write OTA  complete
       #. 04 -> write signature block send(divide to 4 blocks)
       #. 05 <- received OTA verify success 

    #. dataChar>

       #. OTATransferContinue(false); 
   

  #.  当写入当前block的最后一个segment完成后，手机请求gatt.read(dataChar)当前block的segment写入状态。
  #.  设备响应当前block写入状态。

  #.  判断获取的byte[]数据,每个字节代表每个segment写入的状态。

    #. 如果全部写入成功。
    #. 判断当前的block是否是最后一个block。
    #. 如果是，说明所有的block写入完成，此时手机发送写入数据完成请求，设备回复写入数据完成后，说明整个OTA  image写入成功。
    #. 如果不是，说明还有block没有写入，此时开启新的block write 

  #. 如果部分segment写入失败，重新写入blocks data
     #. 例如一个block 有128个segments,其中下标为0,2,58,127的segment 传输失败（lost in air）
     
     那么必须将当前0~127的segment 从新写入一遍。
   
第二阶段流程图如下所示
*************************
   
 .. image:: ../img/OTA_tansport.png
   
 .. image:: ../img/AndroidOTASequenceChart.png
 

App 效果图
*************************************************

关于MeshOTAManager使用
************************

 .. image:: ../img/Android_App_UI1.png
 .. image:: ../img/Android_App_UI2.png



.. code:: java

  <?java
   //在Activity 中初始化
    protected void onCreate(final Bundle savedInstanceState) {
    MeshOTAManager manager=new MeshOTAManager(this, ota_data);
        manager.setSignData(sign_data);
        manager.setNeedCrc32(mNeedCrc32);//SDK version>=2.0
        manager.setNeedCheckSign(mNeedCheckSign);//checkSign
         manager.setGattCallbacks(this)
     }
    ?>
