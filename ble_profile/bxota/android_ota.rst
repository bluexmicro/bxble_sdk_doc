=======================
BX2400 OTA  on Android
=======================
  
 
第一阶段
==================================================================================================================================================================================================================================================================================================================================
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
==================================================================================================================================================================================================================================================================================================================================
   
 .. image:: ../img/BXOTAAndroid_READY.png
   
  
第二阶段流程图如下所示
==================================================================================================================================================================================================================================================================================================================================
   
 .. image:: ../img/OTA_tansport.png

 
    
OTA 升级的三种情况流程图
==================================================================================================================================================================================================================================================================================================================================
 
 .. image:: ../img/OTA.png
 .. image:: ../img/OTA_with_Crc32.png
 .. image:: ../img/OTA_with_crc32_signature.png



OTA opcode 说明 
==================================================================================================================================================================================================================================================================================================================================

      #. ctrlChar>控制特征值

       #. opcode:00 ->BXOTA_CTRL_PKT_START_REQ  （开始请求升级）
          
       #. opcode:01 <- BXOTA_CTRL_PKT_START_RSP  （设备收到请求并且响应 ）
           
       #. opcode:02 -> BXOTA_CTRL_PKT_NEW_BLOCK_CMD  （image transport）
       #. opcode: 03 ->BXOTA_CTRL_PKT_IMAGE_TRANSFER_FINISH_CMD  （app 已经将image  全部写入成功）
       #. opcode:04 -> BXOTA_CTRL_PKT_SIGN_DATA_SEND（写入签名文件字节流，一共64bytes,一次写入16字节，一共写入四次 ）
       #. opcode:05 <- BXOTA_CTRL_PKT_SIGN_DATA_RSP （设备已经收到手机写入的image.bin 和signature.bin，并且验证合法，升级成功）


CRC32  ftp://bluexsh.22ip.net/misc/OTA/crc32/
==================================================================================================================================================================================================================================================================================================================================




App 效果图
==================================================================================================================================================================================================================================================================================================================================
   .. image:: ../img/Android_App_UI1.png





android 代码执行步骤仅供参考
==================================================================================================================================================================================================================================================================================================================================
    #.请查看参考代码中 setp1~11 的执行步骤
      
   


android  OTA 参考代码
==================================================================================================================================================================================================================================================================================================================================


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
               
                // step 8 每调用一次 segmentTX 就会回调这个函数
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
     //step 7 传输image
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
         // 当开启新的block 会执行这里
            currentSegment = 0;
            log(Log.DEBUG, "newBlock start..");

        } else {
            // 写入一次就++
            ++currentSegment;
        }
        
        while (currentSegment < segmentNum && currentAck[currentSegment]) {
            ++currentSegment;
        }
        //step  9 判断当前block 是否写入完毕
        if (currentSegment == segmentNum) {
            log(Log.DEBUG, "Seg 1 ~ seg " + segmentNum + " write complete then read ack");
             //step  10  手机请求读取当前block 写入状态
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
       //step 5  传输前先用ctrlChar(控制特征值)发送新的block 开始cmd
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
                //step 6 
                OTATransferContinue(true);
                break;
            case BXOTA_CTRL_PKT_IMAGE_TRANSFER_FINISH_CMD:
                log(Log.DEBUG, "ota  complete....: ");
                break;
            case BXOTA_CTRL_PKT_SIGN_DATA_SEND:
                mCallbacks.print(String.format("sign data:%d of 4 send complete", +txData[1]));
                //step 3-2 // 这里会连续回调4次  也就是写入四次 
                
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
      //step 1 
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
             //step 2
                log(Log.DEBUG, "received OTA start Resp");
                maxSegmentNumInBlock = rxData[2] * 8;
                log(Log.DEBUG, "total segments size(): " + maxSegmentNumInBlock);
                currentAck = new boolean[maxSegmentNumInBlock];
                if (status == 0) {
                //step 3 判断是否需要验证Signature data
                    if (mNeedCheckSign) {
                       //step 3-1  传输signature.bin 中的0~16 bytes
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
     //step 4  开始传输image
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
    // step 11 接收到设备返回的block 写入状态
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
            //判断所有的block 是否全部写入完毕
            if (++currentBlock == blockNum) {
                log(Log.DEBUG, "OTA Complete");
                //发送结束命令
                imageTXFinishCmd();
            } else {
                log(Log.DEBUG, " next block:" + currentBlock);
               // 下一个block 传输
                newBlockCmd();
            }
        } else {
            //tranfor failed >>continue transfor
            //走到这里说明传输过程丢失了一部分数据，那么将当前block 重新传输一次
            OTATransferContinue(true);
        }

    }


}


 

  