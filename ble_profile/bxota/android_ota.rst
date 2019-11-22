
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
 
OTA opcode 说明 
*******************************************************

      #. ctrlChar>控制特征值

       #. opcode:00 ->BXOTA_CTRL_PKT_START_REQ  （开始请求升级）
          
       #. opcode:01 <- BXOTA_CTRL_PKT_START_RSP  （设备收到请求并且响应 ）
           
       #. opcode:02 -> BXOTA_CTRL_PKT_NEW_BLOCK_CMD  （image transport）
       #. opcode: 03 ->BXOTA_CTRL_PKT_IMAGE_TRANSFER_FINISH_CMD  （app 已经将image  全部写入成功）
       #. opcode:04 -> BXOTA_CTRL_PKT_SIGN_DATA_SEND（写入签名文件字节流，一共64bytes,一次写入16字节，一共写入四次 ）
       #. opcode:05 <- BXOTA_CTRL_PKT_SIGN_DATA_RSP （设备已经收到手机写入的image.bin 和signature.bin，并且验证合法，升级成功）


CRC32  ftp://bluexsh.22ip.net/misc/OTA/crc32/
**************************************************************************************************************


OTA 具体操作流程（以下步骤操作前提是设备已经连接ctrlChar，dataChar 已经获取到）
**************************************************************************************************************
    #.ctrlChar
      step 1 请求OTA
       mtu=20-1 , crc32:4byte 小端,  imagelength：4byte 小端
       
       格式：[opcode,mtu,00,crc32,imagelength]

       ->ctrlChar.write(BXOTA_CTRL_PKT_START_REQ-mtu-00)   不进行CRC32校验  

       ->ctrlChar.write(BXOTA_CTRL_PKT_START_REQ-mtu-00-crc32)  添加CRC32校验

      step 2 <- 回调 onCharacteristicIndicated:ctrlChar.getvalue()=>value[0]==BXOTA_CTRL_PKT_START_RSP?请求成功：失败
       
      step 3 signature.bin传输
       
        格式：[opcode,index,sigData]

        opcode=BXOTA_CTRL_PKT_SIGN_DATA_SEND

        index=0~3 signature.bin文件 64bytes 分成4段

        sigData=16bytes
    
        ctrlChar.write(BXOTA_CTRL_PKT_SIGN_DATA_SEND-index-sigData)

       每写一次都会回调 onCharacteristicWrite
       ::

             bytes data=characteristic.getValue()
              switch(data[0]){
              case:BXOTA_CTRL_PKT_SIGN_DATA_SEND
              {    
                index=txData[1]
                    if(index<3){
                     index++
                     ctrlChar.write(BXOTA_CTRL_PKT_SIGN_DATA_SEND-index-sigData)
                    }else{
                     //这里才开始image 传输
                     startOTATransfer()
                    }
                    
                  
              }
                   
              }



   #.image 传输
    #. ctrlChar

      格式：[opcode,blockiId,blockiId>>8]

       ctrlChar.write(BXOTA_CTRL_PKT_NEW_BLOCK_CMD-blockiId,blockiId>>8)
       ::
         
           回调>onCharacteristicWrite
           byte[] txData=ctrlChar.getvalue()
           opcode=txData[0]
           switch(opcode){
           case:BXOTA_CTRL_PKT_NEW_BLOCK_CMD:
             //第一个block 写入
              OTATransferContinue(true)
                break;
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
           }
           if (currentSegment == segmentNum) {
               
             mBluetoothGatt.readCharacteristic(dataChar) ;//当前block写入完毕 询问设备当前block 写入状态
              //接着回调
              @Override
             protected void onCharacteristicRead(@NonNull BluetoothGatt gatt, @NonNull BluetoothGattCharacteristic characteristic) {
            super.onCharacteristicRead(gatt, characteristic);
            final byte[] data = characteristic.getValue();
            log(Log.DEBUG, "dataReceived: " + ParserUtils.parse(data));
            if (characteristic.getUuid().compareTo(MESH_OTA_CHAR_DATA_UUID) == 0) {
                onAckRead(characteristic.getValue());
              }
              }

             } else {
             segmentTX();
             }

             }

              void onAckRead(byte[] rxBytes) {
        boolean allAcked = true;
        int segmentNum = getSegmentNumOfCurrentBlock();
        for (int i = 0; i < segmentNum; ++i) {
            //check  data send
            if ((rxBytes[i / 8] & (1 << i % 8)) != 0) {
                currentAck[i] = true;
             } else {
                // transfor filed
                allAcked = false;
                currentAck[i] = false;

             }
            }
          if (allAcked) {
            float progress = (float) (currentBlock + 1) / blockNum;
            mCallbacks.onProgress(progress);
            if (++currentBlock == blockNum) {
                log(Log.DEBUG, "OTA Complete");
                imageTXFinishCmd();
            } else {
                log(Log.DEBUG, " next block:" + currentBlock);
                newBlockCmd();
            }
        } else {
            //tranfor failed >>continue transfor
            OTATransferContinue(true);
        }

    }

#. dataChar>

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
   .. image:: ../img/Android_App_UI1.png





android  OTA 参考代码
************************


::

            package com.lianrui.mesh_ota;

       import android.bluetooth.BluetoothGatt;
       import android.bluetooth.BluetoothGattCharacteristic;
       import android.bluetooth.BluetoothGattDescriptor;
       import android.bluetooth.BluetoothGattService;
       import android.content.Context;
       import android.os.Build;
       import android.support.annotation.NonNull;
       import android.util.Log;

       import java.nio.ByteBuffer;
       import java.nio.ByteOrder;
       import java.util.Arrays;
       import java.util.Deque;
       import java.util.LinkedList;
       import java.util.UUID;

       import no.nordicsemi.android.ble.BleManager;
       import no.nordicsemi.android.ble.Request;
       import no.nordicsemi.android.ble.utils.ParserUtils;

       import static com.lianrui.mesh_ota.Crc32.CRC32_INIT_VAL;

       public class MeshOTAManager extends BleManager<OTAManagerCallbacks> {
       public final static String ERROR_CONNECTION_STATE_CHANGE = "Error on connection state change";
       public final static String ERROR_DISCOVERY_SERVICE = "Error on discovering services";
       public final static String ERROR_AUTH_ERROR_WHILE_BONDED = "Phone has lost bonding information";
       public final static String ERROR_READ_CHARACTERISTIC = "Error on reading characteristic";
       public final static String ERROR_WRITE_CHARACTERISTIC = "Error on writing characteristic";
       public final static String ERROR_READ_DESCRIPTOR = "Error on reading descriptor";
       public final static String ERROR_WRITE_DESCRIPTOR = "Error on writing descriptor";
       public final static String ERROR_MTU_REQUEST = "Error on mtu request";
       public final static String ERROR_CONNECTION_PRIORITY_REQUEST = "Error on connection priority request";
       public final static String ERROR_READ_RSSI = "Error on RSSI read";
       public final static String ERROR_READ_PHY = "Error on PHY read";
       public final static String ERROR_PHY_UPDATE = "Error on PHY update";
       public final static String ERROR_RELIABLE_WRITE = "Error on Execute Reliable Write";

    /**
     * The maximum packet size is 20 bytes.
     */
    private static final int MAX_PACKET_SIZE = 20;
    public static final int MTU_SIZE_MIN = 23;
    private static final int MTU_SIZE_MAX = 40;
    /**
     * Mesh provisioning data in characteristic UUID
     */

    /**
     * Mesh OTA service UUID
     */
    public final static UUID MESH_OTA_UUID = UUID.fromString("00002600-0000-1000-8000-00805F9B34FB");

    private final static UUID MESH_OTA_CHAR_CTRL_UUID = UUID.fromString("00007000-0000-1000-8000-00805F9B34FB");

    private final static UUID MESH_OTA_CHAR_DATA_UUID = UUID.fromString("00007001-0000-1000-8000-00805F9B34FB");

    public static final UUID CCCD = UUID.fromString("00002902-0000-1000-8000-00805f9b34fb");

    private final String TAG = MeshOTAManager.class.getSimpleName();
    private BluetoothGattCharacteristic ctrlChar;
    private BluetoothGattCharacteristic dataChar;

    private BluetoothGatt mBluetoothGatt;
    public final static int BXOTA_CTRL_PKT_START_REQ = 0;
    public final static int BXOTA_CTRL_PKT_START_RSP = 1;
    public final static int BXOTA_CTRL_PKT_NEW_BLOCK_CMD = 2;
    public final static int BXOTA_CTRL_PKT_IMAGE_TRANSFER_FINISH_CMD = 3;
    public final static int BXOTA_CTRL_PKT_SIGN_DATA_SEND = 4;
    public final static int BXOTA_CTRL_PKT_SIGN_DATA_RSP = 5;

    private byte[] OTAData;
    private boolean[] currentAck;
    private int blockNum;
    private int maxSegmentNumInBlock;
    private int lastBlockSegmentNum;
    private short currentBlock;
    private short currentSegment;
    private short maxSegmentDataSize = 19;
    private final short blockHeaderSize = 1;
    private final static int MAX_SIGNATURE_SEGMENT_COUNT = 4 - 1;//index from 0 so max segindex=size-1

    public void setNeedCheckSign(boolean mNeedCheckSign) {
        this.mNeedCheckSign = mNeedCheckSign;
    }

    private boolean mNeedCheckSign;
    private boolean mNeedCrc32 = false;
    private static final int MAX_SIGN_SEG_SIZE = 16;

    public void setNeedCrc32(boolean mNeedCrc32) {
        this.mNeedCrc32 = mNeedCrc32;
    }

    public void setSignData(byte[] signData) {
        this.signData = signData;
    }

    private byte[] signData;

    public MeshOTAManager(Context context, byte[] otaData) {
        super(context);
        this.OTAData = otaData;

    }


    BleManagerGattCallback bleManagerGattCallback = new BleManagerGattCallback() {
        @Override
        protected Deque<Request> initGatt(@NonNull BluetoothGatt gatt) {
            isOtaAReady = false;
            final LinkedList<Request> requests = new LinkedList<>();
            mBluetoothGatt = gatt;
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                mBluetoothGatt.requestConnectionPriority(BluetoothGatt.CONNECTION_PRIORITY_HIGH);
            }

            ctrlChar.setWriteType(BluetoothGattCharacteristic.WRITE_TYPE_DEFAULT);
            mBluetoothGatt.setCharacteristicNotification(ctrlChar, true);
            dataChar.setWriteType(BluetoothGattCharacteristic.WRITE_TYPE_NO_RESPONSE);
            mBluetoothGatt.setCharacteristicNotification(dataChar, true);
            BluetoothGattDescriptor ctrlDesc = ctrlChar.getDescriptor(CCCD);
            ctrlDesc.setValue(BluetoothGattDescriptor.ENABLE_INDICATION_VALUE);
            mBluetoothGatt.writeDescriptor(ctrlDesc);

            return null;
        }

        @Override
        protected boolean isRequiredServiceSupported(@NonNull BluetoothGatt gatt) {
            for (BluetoothGattService service : gatt.getServices()) {
                if (service.getUuid().toString().equals(MESH_OTA_UUID.toString())) {
                    log(Log.DEBUG, "Service:" + service.getUuid().toString());
                    for (BluetoothGattCharacteristic characteristic : service.getCharacteristics()) {
                        log(Log.DEBUG, "characteristic:" + characteristic.getUuid().toString());
                    }
                } else {
                }
            }
            boolean writeRequest;
            BluetoothGattService meshService = gatt.getService(MESH_OTA_UUID);
            if (meshService != null) {
                log(Log.DEBUG, "found OTA services  ");
                ctrlChar = meshService.getCharacteristic(MESH_OTA_CHAR_CTRL_UUID);
                dataChar = meshService.getCharacteristic(MESH_OTA_CHAR_DATA_UUID);
                writeRequest = false;
                if (dataChar != null) {
                    final int rxProperties = dataChar.getProperties();
                    writeRequest = (rxProperties & BluetoothGattCharacteristic.PROPERTY_WRITE_NO_RESPONSE) > 0;
                }
                return (ctrlChar != null && dataChar != null && writeRequest);
            }

            log(Log.DEBUG, "OTA service not support");
            return false;
        }

        @Override
        protected void onDeviceDisconnected() {

        }

        @Override
        protected void onCharacteristicIndicated(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic) {
            super.onCharacteristicIndicated(gatt, characteristic);
            log(Log.DEBUG, "onCharacteristicIndicated: " + characteristic.getUuid().toString());
            if (characteristic.getUuid().compareTo(MESH_OTA_CHAR_CTRL_UUID) == 0) {
                ctrlPktIndicationRX(characteristic.getValue());
            }

        }

        @Override
        public void onCharacteristicWrite(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic, int status) {

            if (status == 0) {
                super.onCharacteristicWrite(gatt, characteristic, status);
            } else {
                if (!isOtaAReady) {
                    startOtaRequest();
                }
            }
        }

        @Override
        protected void onCharacteristicWrite(@NonNull BluetoothGatt gatt, @NonNull BluetoothGattCharacteristic characteristic) {
            super.onCharacteristicWrite(gatt, characteristic);
            if (characteristic.getUuid().compareTo(MESH_OTA_CHAR_CTRL_UUID) == 0) {
                byte[] txBytes = characteristic.getValue();
                ctrlPktSent(txBytes);
            } else if (characteristic.getUuid().compareTo(MESH_OTA_CHAR_DATA_UUID) == 0) {
                OTATransferContinue(false);
            }

        }

        @Override
        public void onDescriptorWrite(BluetoothGatt gatt, BluetoothGattDescriptor descriptor, int status) {
         //  super.onDescriptorWrite(gatt, descriptor, status);
            log(Log.INFO, "onDescriptorWrite:" + descriptor.getUuid().toString());
            if (descriptor.getUuid().compareTo(CCCD) == 0) {
                startOtaRequest();
            }
        }


        @Override
        protected void onCharacteristicRead(@NonNull BluetoothGatt gatt, @NonNull BluetoothGattCharacteristic characteristic) {
            super.onCharacteristicRead(gatt, characteristic);
            final byte[] data = characteristic.getValue();
            log(Log.DEBUG, "dataReceived: " + ParserUtils.parse(data));
            if (characteristic.getUuid().compareTo(MESH_OTA_CHAR_DATA_UUID) == 0) {
                onAckRead(characteristic.getValue());
            }
        }

        @Override
        public void onCharacteristicNotified(final BluetoothGatt gatt, final BluetoothGattCharacteristic characteristic) {
            super.onCharacteristicNotified(gatt, characteristic);
        }


        @Override
        protected void onMtuChanged(@NonNull int mtu) {
            super.onMtuChanged(mtu);
            maxSegmentDataSize = (short) (mtu - 3 - blockHeaderSize);
            log(Log.DEBUG, "onMtuChanged: " + maxSegmentDataSize);
        }
    };


    @Override
    public void log(int priority, @NonNull String message) {
        super.log(priority, message);
        if (priority==Log.DEBUG)
        mCallbacks.print(message);
        Log.d(TAG, message);

    }


    @NonNull
    @Override
    protected BleManagerGattCallback getGattCallback() {
        return bleManagerGattCallback;
    }

    private void segmentTX() {
        int length = getSegmentLength(currentBlock, currentSegment);
        byte[] data = new byte[blockHeaderSize + length];
        data[0] = (byte) currentSegment;
        data[1] = (byte) (currentSegment >> 8);
        System.arraycopy(OTAData, (currentBlock * maxSegmentNumInBlock + currentSegment) * maxSegmentDataSize
                , data, blockHeaderSize, length);
        dataChar.setValue(data);
        writeCharacteristic(dataChar, data);
    }


    private int getSegmentLength(int blockID, int segmentID) {
        if (blockID == blockNum - 1 && segmentID == lastBlockSegmentNum - 1) {
            return OTAData.length - maxSegmentDataSize *
                    ((blockNum - 1) * maxSegmentNumInBlock + (lastBlockSegmentNum - 1));
        } else {
            return maxSegmentDataSize;
        }
    }

    public void readAck() {
        mBluetoothGatt.readCharacteristic(dataChar);
    }

    public void writeDescriptor() {

    }

    void OTATransferContinue(boolean newBlock) {
        int segmentNum = getSegmentNumOfCurrentBlock();
        if (newBlock) {
            currentSegment = 0;
            log(Log.DEBUG, "newBlock start..");

        } else {
            ++currentSegment;
        }
        while (currentSegment < segmentNum && currentAck[currentSegment]) {
            ++currentSegment;
        }
        if (currentSegment == segmentNum) {
            log(Log.DEBUG, "Seg 1 ~ seg " + segmentNum + " write complete then read ack");
            readAck();

        } else {
            segmentTX();
        }

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
         //ctrlChar.setValue(ctrl);
        if (type == BXOTA_CTRL_PKT_START_REQ) {
            log(Log.DEBUG, "send start reuest:" + Arrays.toString(ctrl));
        } else if (type == BXOTA_CTRL_PKT_NEW_BLOCK_CMD) {
            log(Log.DEBUG, "send new block cmd:" + Arrays.toString(ctrl));
        } else if (type == BXOTA_CTRL_PKT_IMAGE_TRANSFER_FINISH_CMD) {
            log(Log.DEBUG, "send transfer comolete cmd:" + Arrays.toString(ctrl));
        } else {

        }

        writeCharacteristic(ctrlChar, ctrl);

    }


    private void newBlockCmd() {
        mCallbacks.print("currentBlock:" + currentBlock);
        byte[] blockIDArray = new byte[]{(byte) currentBlock, (byte) (currentBlock >> 8)};
        ctrlPktTX(BXOTA_CTRL_PKT_NEW_BLOCK_CMD, blockIDArray);

    }

    private void imageTXFinishCmd() {
        ctrlPktTX(BXOTA_CTRL_PKT_IMAGE_TRANSFER_FINISH_CMD, null);
    }

    private boolean isOtaAReady;

    void ctrlPktSent(byte[] txData) {
        switch (txData[0]) {
            case BXOTA_CTRL_PKT_START_REQ:
                isOtaAReady = true;
                log(Log.DEBUG, "ota ready: ");
                break;
            case BXOTA_CTRL_PKT_NEW_BLOCK_CMD:
                Arrays.fill(currentAck, false);
                OTATransferContinue(true);
                break;
            case BXOTA_CTRL_PKT_IMAGE_TRANSFER_FINISH_CMD:
                log(Log.DEBUG, "ota  complete....: ");
                break;
            case BXOTA_CTRL_PKT_SIGN_DATA_SEND:
                mCallbacks.print(String.format("sign data:%d of 4 send complete", +txData[1]));
                if (txData[1] < MAX_SIGNATURE_SEGMENT_COUNT) {
                    transSignDataCmd(txData[1] + 1);
                } else {
                    startOTATransfer();
                }

                break;
            default:

                break;

        }
    }

    void transSignDataCmd(int index) {
        int type = BXOTA_CTRL_PKT_SIGN_DATA_SEND;
        int nextIndex = index;
        byte[] data = new byte[17];
        data[0] = (byte) index;
        System.arraycopy(signData, nextIndex * MAX_SIGN_SEG_SIZE, data, 1, MAX_SIGN_SEG_SIZE);
        log(Log.DEBUG, ParserUtils.parse(data));
        ctrlPktTX(type, data);
    }


    private void startOtaRequest() {
        mCallbacks.onOTARequestStart();
        int lenth = mNeedCrc32 ? 10 : 2;
        ByteBuffer buffer = ByteBuffer.allocate(lenth).order(ByteOrder.LITTLE_ENDIAN);
        byte[] maxBlockDataSizeArray = new byte[]{(byte) maxSegmentDataSize, (byte) (maxSegmentDataSize >> 8)};
        buffer.put(maxBlockDataSizeArray);
        if (mNeedCrc32) {
            buffer.putInt(crc32());
            buffer.putInt(OTAData.length);
        }
        log(Log.DEBUG, "startOtaRequest: " + maxSegmentDataSize);
        ctrlPktTX(BXOTA_CTRL_PKT_START_REQ, buffer.array());

    }


    private int crc32() {
        Crc32 crc32 = new Crc32();
        long crc = crc32.crc32_calc(CRC32_INIT_VAL, OTAData, OTAData.length);
        return (int) crc;
    }

    void ctrlPktIndicationRX(byte[] rxData) {
        int status = rxData[1];
        switch (rxData[0]) {
            case BXOTA_CTRL_PKT_START_RSP:
                log(Log.DEBUG, "received OTA start Resp");
                maxSegmentNumInBlock = rxData[2] * 8;
                log(Log.DEBUG, "total segments size(): " + maxSegmentNumInBlock);
                currentAck = new boolean[maxSegmentNumInBlock];
                if (status == 0) {
                    if (mNeedCheckSign) {
                        transSignDataCmd(0);
                    } else {
                        startOTATransfer();
                    }

                }
                break;
            case BXOTA_CTRL_PKT_SIGN_DATA_RSP:
                String mOtaStatus = status == 1 ? "OTA Status:" + "success" : "failed";
                log(Log.DEBUG, mOtaStatus);
                break;
            default:

                break;

        }
    }


    private void startOTATransfer() {
        mCallbacks.onOTAStart();
        new Thread(new Runnable() {
            @Override
            public void run() {
                blockNum = (int) Math.ceil((double) OTAData.length / (maxSegmentDataSize * maxSegmentNumInBlock));
                int lastBlockDataLength = OTAData.length % (maxSegmentDataSize * maxSegmentNumInBlock);
                lastBlockSegmentNum = (int) Math.ceil((double) lastBlockDataLength / maxSegmentDataSize);
                currentBlock = 0;
                newBlockCmd();
            }
        }).start();
    }


    private int getSegmentNumOfCurrentBlock() {
        if (currentBlock == blockNum - 1) {
            return lastBlockSegmentNum;
        } else {
            return maxSegmentNumInBlock;
        }
    }

    void onAckRead(byte[] rxBytes) {
        boolean allAcked = true;
        int segmentNum = getSegmentNumOfCurrentBlock();
        for (int i = 0; i < segmentNum; ++i) {
            //check  data send
            if ((rxBytes[i / 8] & (1 << i % 8)) != 0) {
                currentAck[i] = true;
            } else {
                // transfor filed
                allAcked = false;
                currentAck[i] = false;

            }
        }
        if (allAcked) {
            float progress = (float) (currentBlock + 1) / blockNum;
            mCallbacks.onProgress(progress);
            if (++currentBlock == blockNum) {
                log(Log.DEBUG, "OTA Complete");
                imageTXFinishCmd();
            } else {
                log(Log.DEBUG, " next block:" + currentBlock);
                newBlockCmd();
            }
        } else {
            //tranfor failed >>continue transfor
            OTATransferContinue(true);
        }

    }


}


 

  