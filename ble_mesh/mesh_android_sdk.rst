在app 项目的 build.gradle 中引入一下依赖
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

    implementation 'no.nordicsemi.android:log:2.1.1'
    implementation 'com.madgag.spongycastle:core:1.56.0.0'
    implementation 'com.madgag.spongycastle:prov:1.56.0.0'
    implementation 'com.google.code.gson:gson:2.8.5'
    implementation 'no.nordicsemi.android.support.v18:scanner:1.1.0'
    implementation "android.arch.lifecycle:extensions:1.1.1"
    implementation 'android.arch.persistence.room:runtime:1.1.1'
    annotationProcessor "android.arch.persistence.room:compiler:1.1.1"
    androidTestImplementation 'android.arch.persistence.room:testing:1.1.1'


mesh sdk 初始化
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
:: 

    
   MeshManagerApi mMeshManagerApi = new MeshManagerApi(context);
                  mMeshManagerApi.setMeshManagerCallbacks(this);
                  mMeshManagerApi.setProvisioningStatusCallbacks(this);
                  mMeshManagerApi.setMeshStatusCallbacks(this);

   BleMeshManager bleMeshManager = new BleMeshManager(context)

   NetworkInformation networkInformation=new NetworkInformation(context)

   NrfMeshRepository mMeshRepository=NrfMeshRepository(mMeshManagerApi, networkInformation, bleMeshManager);

以上是初始化流程







扫描未入网设备
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
  
  ScannerRepository  mScannerRepository=new ScannerRepository(context,meshManagerApi)
  mScannerRepository.getScannerState().startScanning();
  //扫描前建议判断下蓝牙是否可用，定位是否打开（Android sdk>= 6.0）
  mScannerRepository.startScan(BleMeshManager.MESH_PROVISIONING_UUID)
  //扫描结果回调
  mScannerRepository.getScannerState().observe(this, state -> {
          mDevices = scannerLiveData.getDevices();
         final Integer i = devices.getUpdatedDeviceIndex();
            if (i != null)
                notifyItemChanged(i);
            else
                notifyDataSetChanged();
        });



扫描已经入网设备
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

  ScannerRepository  mScannerRepository=new ScannerRepository(context,meshManagerApi)
  mScannerRepository.getScannerState().startScanning();
  //扫描前建议判断下蓝牙是否可用，定位是否打开（Android sdk>= 6.0）
  mScannerRepository.startScan(BleMeshManager.MESH_PROXY_UUID)
  //扫描结果回调
  mScannerRepository.getScannerState().observe(this, state -> {
          mDevices = scannerLiveData.getDevices();
         final Integer i = devices.getUpdatedDeviceIndex();
            if (i != null)
                notifyItemChanged(i);
            else
                notifyDataSetChanged();
        });


点击设备开始连接,入网开始
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
1  设备连接成功后，mesh SDK 进行发现服务，使能通知(接收硬件端发送的数据)
:: 

   final UnprovisionedBeacon beacon = (UnprovisionedBeacon) device.getBeacon();
        //private static final int ATTENTION_TIMER = 5;
        if (beacon != null) {
            mMeshManagerApi.identifyNode(beacon.getUuid(), device.getName(), device.getAddress(), ATTENTION_TIMER);
        } else {
            final byte[] serviceData = getServiceData(device.getScanResult(), BleMeshManager.MESH_PROVISIONING_UUID);
            if (serviceData != null) {
                final UUID uuid = mMeshManagerApi.getDeviceUuid(serviceData);
                //第一步  发送invite  消息
                mMeshManagerApi.identifyNode(uuid, device.getName(), device.getAddress(), ATTENTION_TIMER);
            }
        }



   // 入网回调
   mNrfMeshRepository.getUnprovisionedMeshNode().observe(this, meshNode -> {
            if (meshNode != null) {
                if (meshNode.getDeviceAddress() == null) {
                    meshNode.setDeviceAddress(device.getAddress());
                }
                  // 第二步  收到设备端回复Capabilities 
                if (meshNode.getProvisioningCapabilities() != null) { 
                 //此处可以进行界面刷新
                 // 第三步  发送provision start 消息  有三种方式
                  final UnprovisionedMeshNode node = mMeshRepository.getUnProvisionedMeshNode().getValue();
                 //mMeshRepository.getMeshManagerApi().startProvisioning(node);
                //mMeshRepository.getMeshManagerApi().startProvisioningWithStaticOOB(node);
                 //mMeshRepository.getMeshManagerApi().startProvisioningWithOutputOOB(node, action);
               //调用 startProvision 后,sdk会自动完成入网
                  
                }
            }
        });
     // provision state callback
   mNrfMeshRepository.getProvisioningStatus().observe(this, provisioningStateLiveData -> {
            if (provisioningStateLiveData != null) {
                final ProvisionerProgress provisionerProgress = provisioningStateLiveData.getProvisionerProgress();
         
                if (provisionerProgress != null) {
                    final ProvisioningStatusLiveData.ProvisioningLiveDataState state = provisionerProgress.getState();                
                    switch (state) {
                        case PROVISIONING_INVITE:
                            // step 1
                            break;
                        case PROVISIONING_CAPABILITIES:
                            // step 2
                            
                            break;
                        case PROVISIONING_START:
                            //step 3
                             break;
                        case PROVISIONING_FAILED:
                        // 入网失败
                           
                            break;
                        case PROVISIONING_AUTHENTICATION_STATIC_OOB_WAITING:
                         //step 4
                         break;
                        case PROVISIONING_AUTHENTICATION_OUTPUT_OOB_WAITING:
                        //step 4
                          
                            break;
                        case PROVISIONING_AUTHENTICATION_INPUT_OOB_WAITING:
                        //step 4
                          
                            break;
                        case PROVISIONING_AUTHENTICATION_INPUT_ENTERED:
                           //step 4
                            break;
                        case COMPOSITION_DATA_STATUS_RECEIVED:
                          
                            break;
                        case PROVISIONING_PUBLIC_KEY_SENT:
                            //step 5
                            break;
                        case PROVISIONING_PUBLIC_KEY_RECEIVED:
                            //step 6
                            break;   
                        case PROVISIONING_RANDOM_SENT:
                            //step 7
                            break;
                        case PROVISIONING_RANDOM_RECEIVED:
                            //step 8
                            break;
                        case PROVISIONING_DATA_SENT:
                            //step 9
                            break;
                        case PROVISIONING_COMPLETE:
                        // 入网完成 
                           //received
                            break      
                        case APP_KEY_STATUS_RECEIVED:
                          
                            break;
                    }

                }
                
            }
        });

入网成功后手机数据库中会插入一条该设备数据 在MeshManagerApi.java中
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
 ::

    private final InternalMeshManagerCallbacks internalMeshMgrCallbacks = new InternalMeshManagerCallbacks() {
        @Override
        public void onNodeProvisioned(final ProvisionedMeshNode meshNode) {
            updateProvisionedNodeList(meshNode);
            incrementUnicastAddress(meshNode.getUnicastAddress(), meshNode.getNumberOfElements());
            //Set the mesh network uuid to the node so we can identify nodes belonging to a network
            meshNode.setMeshUuid(mMeshNetwork.getMeshUUID());
            //插入节点数据
            mMeshNetworkDb.insertNode(mProvisionedNodeDao, meshNode);
            mMeshNetworkDb.updateProvisioner(mProvisionerDao,
                    mMeshNetwork.getSelectedProvisioner());
            mTransportCallbacks.onNetworkUpdated(mMeshNetwork);
        }

        private void updateProvisionedNodeList(final ProvisionedMeshNode meshNode) {
            Log.d(TAG, "updateProvisionedNodeList: " + meshNode.getUuid());
            for (int i = 0; i < mMeshNetwork.nodes.size(); i++) {
                final ProvisionedMeshNode node = mMeshNetwork.nodes.get(i);
                Log.d(TAG, "updateProvisionedNodeList: " + node.getUuid());
                if (meshNode.getUuid().equals(node.getUuid())) {
                    mMeshNetwork.nodes.remove(i);
                    break;
                }
            }
            mMeshNetwork.nodes.add(meshNode);
        }
    };       



发送消息注意事项
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
ConfigMessage 地址必须是：unicastAddress（mMeshRepository.getSelectedMeshNode().value().getUnicastAddress()）
GenericMessage 地址必须是：节点元素地址 elementAddress 

ConfigMessage 属于配置信息主要目的是给相应的model 添加通信能力的字段信息
例如：ConfigAppkeyAdd 添加节点通信加解密appkey（16byte）
例如：ConfigModelAppBind 用于model通信加解密绑定
例如：ConfigModelSubscriptionAdd  订阅分组
::

    public ConfigModelSubscriptionAdd(@NonNull final byte[] elementAddress,
                                      @NonNull final byte[] subscriptionAddress,
                                      final int modelIdentifier) throws IllegalArgumentException {
                                      //subscriptionAddress 分组地址 0xc000~0xFFEF
        this(AddressUtils.getUnicastAddressInt(elementAddress), AddressUtils.getUnicastAddressInt(subscriptionAddress), modelIdentifier);
    }



发送消息
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
 
 //example 入网完成后需要获取设备配置信息
  final ConfigCompositionDataGet compositionDataGet = new ConfigCompositionDataGet();
  ProvisionedMeshNode provisionedMeshNode =  mMeshRepository.getProvisionedMeshNode().getValue();
  int dst = provisionedMeshNode.getUnicastAddress();//代表目的地址(elementAddress,groupAddress) 手机发送消息用到的地址
  mMeshRepository.getMeshManagerApi().sendMeshMessage(dst, meshMessage);
  
  mMeshRepository.getNrfMeshRepository().getMeshMessageLiveData().observe(this, new Observer<MeshMessage>() {
                @Override
                public void onChanged(@Nullable MeshMessage meshMessage) {

                   if(meshMessage instanceof ConfigCompositionDataStatus){
                        //收到设备端回复的配置信息 
                        //ConfigCompositionDataStatus 解析得到List :elements ,list:models
                         }else{
                   // more status need to catch
                   }
                  
            });

移除节点
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
  
    final ConfigNodeReset configNodeReset = new ConfigNodeReset();
  ProvisionedMeshNode provisionedMeshNode =  mMeshRepository.getSelectedMeshNode().getValue();//get之前先set  不然会报错
  int dst = provisionedMeshNode.getUnicastAddress();
  mMeshRepository.getMeshManagerApi().sendMeshMessage(dst, meshMessage);
  

  mMeshRepository.getNrfMeshRepository().getMeshMessageLiveData().observe(this, new Observer<MeshMessage>() {
                @Override
                public void onChanged(@Nullable MeshMessage meshMessage) {
                   if(meshMessage instanceof ConfigNodeResetStatus){
                        
                        //节点已经移除，该设备已经变成未入网设备，可以进行入网操作

                   }else{

                   // more status need to catch
                   }
                  
            });


在手机数据库中添加分组信息
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::
  
  //地址范围 0xc000~0xFEFF
  //(例如手机中现有两个节点，并且都订阅了改地址(以下代码中的地址是0xc000),那么现在可以向0xc000 发送一条分组消息，那么这两个节点都会处理该消息
  mMeshRepository.getMeshNetworkLiveData().getMeshNetwork().addGroup(0xc000,"randomName")


从手机数据库中获取分组列表
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

  mMeshRepository.getGroups().observe(this, groups -> {
           //refresh  UI
        });
 

导出手机入网数据
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

   final File f = new File(EXPORT_PATH);

            if (!f.exists()) {

                f.mkdirs();
            }
            mMeshRepository.getMeshManagerApi().exportMeshNetwork(EXPORT_PATH);




导入数据到手机
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
::

   final Intent intent;
        if (Utils.isKitkatOrAbove()) {
            intent = new Intent(Intent.ACTION_OPEN_DOCUMENT);
        } else {
            intent = new Intent(Intent.ACTION_GET_CONTENT);
        }
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        intent.setType("*/*");
        startActivityForResult(intent, READ_FILE_REQUEST_CODE);


         public void onActivityResult(final int requestCode, final int resultCode, final Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        switch (requestCode) {
            case READ_FILE_REQUEST_CODE:
                if (resultCode == RESULT_OK) {
                    if (data != null) {
                        final Uri uri = data.getData();
                        mMeshRepository.importMeshNetwork(uri);
                    }
                } else {
                    Log.e(TAG, "Error while opening file browser");
                }
                break;
        }
    }
 

   