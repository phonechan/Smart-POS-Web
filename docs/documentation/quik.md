## 快速接入
为了方便快速接入，提供了demo供接入参考
[demo下载](http://hd2-prod-smartpos.oss-cn-shanghai.aliyuncs.com/apkMgt/2020-01-07/pos-demo.zip)

* 本 SDK 已使用 mavenCentral 托管，配置如下

```gradle
gradle:
    implementation 'com.cardinfolink.smart.pos:PosSDK:2.6.3'
```

* 如果想使用讯联集成的结算UI和逻辑，请配置

```gradle
gradle:
    implementation 'com.cardinfolink.smart.pos:SDK-ZaiHui:1.1.1'
```
## 初始化（连接POS机硬件）

使用 N900 智能 POS 机器, 首先需要连接 POS 机硬件, 使用 SDK 提供的 `connect` 方法连接。
建议在 `Application` 的 `onCreate` 方法中进行连接。

```
    @Override
    public void onCreate() {
        super.onCreate();
        
        ...
        CILSDK.setDebug(true);//调试阶段使用测试环境，发版时请改为 false
        CILSDK.connect(this);
        
        ...
    }
```
使用联迪A8设备的，联迪设备连接需要传入activity的context,
除了在 `Application` 的 `onCreate` 方法中进行连接，建议在每个可能调用设备刷卡、加密、打印的activity onCreat方法中再调用 CILSDK.connect(this)进行连接，
保证使用过程中设备连接不断开；另外推荐在ActivityLifecycleCallbacks中处理设备连接。

## 激活POS机 
完整的激活环节分为三步，分别是：激活，终端参数下载，终端密钥下载，三步都成功，表示激活成功，激活成功之后才能正常使用后面的交易流程。  

![](../img/activeprocess.png)  

**1、激活**  
* a、新激活流程（推荐）

根据用户反馈，为了简化激活流程，我们新增了激活码的方式激活，当你拿到 POS 机器之后，我们会为这个商户发放激活码，一个激活码只可以激活一台 POS 机。
建议先调用[根据激活码获取商户信息接口:](#load_merInfo)（返回商户名商户号终端号等信息），确认信息无误后，在调用激活码激活接口激活。  

![](../img/newloginprocess.png) 
 
> 注意:一台终端能且仅能成功激活一次,无法重复进行激活操作。若有多次激活的需求(如debug阶段),请联系讯联客服。

```
    //authCode 激活码（建议用扫码的方式得到激活码，避免让用户手输）
    CILSDK.activeWithCode(authCode, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse cilResponse) {
            if (cilResponse.getStatus() == 0) {
            //激活成功
            } else {
            //激活失败，失败原因见 cilResponse.getMessage()
            }
        }

        @Override
        public void onError(Parcelable cilRequest, Exception e) {
            //激活出错
        }
    });
```


* b、旧激活流程

第一次使用智能 POS 终端的时候,需要使用讯联下发的商户号（merCode） 和终端号（termCode）激活  

![](../img/activeprocess.png) 

> 注意:一台终端能且仅能成功激活一次,无法重复进行激活操作。若有多次激活的需求(如debug阶段),请联系讯联客服。推荐APP本身能够保持设备是否已经激活标志

```
    //merCode  讯联下发的商户号
    //termCode 讯联下发的终端号
    CILSDK.active(merCode, termCode, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse cilResponse) {
            
            if (cilResponse.getStatus() == 0) {
                //激活成功
            } else {
                //激活失败
            }
        }

        @Override
        public void onError(Parcelable p, Exception e) {
            //激活出错
        }
    });
```

<span id="load_merInfo" style="font-size:20px;color:red" >注意:</span><br/>
>根据激活码获取商户详情接口：
```
CILSDK.getMerchantInfo(HashUtils.encryptActiveCode(authCode), true, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                //success
                //you will get merName,merCode,termCode
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                //failed
            }
        });

```
  
  
**2、终端参数下载**

激活成功之后,你的应用还需要下载一些交易时使用的参数,比如`交易地址和端口`、`交易超时时间`、`终端支持的功能`、`TPDU` 等,
在你拿到 POS 终端之前,这些参数都会在讯联后台已经配置好,全部参数见 `CILResponse.Info` 返回值。下载成功之后 SKD 会
以json字符串的形式保存这些参数到 `SharedPreference` (请不要擅自改动这些参数值,以免导致交易失败) 。当然,在 `onResult`
中也会返回,你也可以自己选择性保存一些参数。另外, SDK 保存在 SharedPreference 里的值会提供接口 `CILSDK.getSystemParams()` 获取。

> 注意：正常情况下，此方法只需执行成功一次，但是后台参数配置可能会有改动，APP本身需要调用预留功能调用此方法

```
    //merCode  讯联下发的商户号 激活成功之后可以确定商户号
    //termCode 讯联下发的终端号 激活成功之后可以确定终端号
    CILSDK.downloadParams(merCode, termCode, new com.cardinfolink.pos.listener.Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse response) {
            if (0 == response.getStatus()) {
                //参数下载成功,具体返回的参数见 response.Info
            } else {
                //参数下载错误
            }
        }

        @Override
        public void onError(Parcelable p, Exception e) {
                //下载出错
        }
    });
```

**3、终端密钥下载**

终端密钥下载只需要成功执行一次就可以,成功下载的密钥会被转载到POS硬件模块里面,后面就不需要再次调用了,建议你的应用可以在成功下载密钥之后持久化一个标志位,
下次进入应用就不再去下载密钥了。整个过程可能会需要1~2分钟左右(依赖当前的网络状况),会经历以下步骤:

> 请求讯联网关 RSA -> 装载 RSA -> 请求主密钥 -> 装载主密钥 -> 启用主密钥 -> 请求工作密钥(签到) -> 装载工作密钥 -> 下载 AID -> 装载 AID -> 下载 IC 公钥 -> 装载 IC 公钥。
建议APP本身存储设备是否已经初始化方法标志。

>注意：此方法一般只需要安装后成功调用一次即可。但银行交互密钥有可能会更新，APP本身需要调用预留功能调用此方法

```
    CILSDK.downloadParamsWithProgress(new ProgressCallback<CILResponse>() {
        @Override
        public void onProgressUpdate(int progress) {
            //progress下载密钥的进度
        }

        @Override
        public void onResult(CILResponse response) {
            
            if (0 == response.getStatus())
                //密钥下载成功。在这里可以持久化一个标志位
        }

        @Override
        public void onError(Parcelable p, Exception e) {
            //下载密钥出错
        }
    });
```  

## 签到
签到其实也就是更新工作密钥的一个过程 (`下载工作密钥+装载工作密钥` ),讯联网关平台要求应用需要每天签到一次。

```
    CILSDK.signIn(new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse cilResponse) {
            //签到成功
        }

        @Override
        public void onError(Parcelable cilRequest, Exception e) {
            //签到出错
        }
    });
```    
## 银行卡交易
由于银行卡交易逻辑有点复杂,讯联提供了一个 `BaseCardActivity` 基础类,你只需要继承这个类便可以做银行卡类的交易了。具体使用方法可以见 demo 
里的 `CommonCardHandlerActivity` 类。下面给个大概说明:

```java
    public class CommonCardHandlerActivity extends BaseCardActivity {
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            
            //...
            
        }
        
        /**
         * 必须传入金额
         *
         * @return
         */
        public String getAmount() {
            //在这里传入金额
        }
		
		
		/**
		 * 此方法为控制sdk内部是否开启DCC交易逻辑（sdk2.4.1版本及以上支持该方法）
		 * 
		 *@return 需要进行DCC交易时，返回true，否则返回false
		 */
		public boolean isOpenDcc() {
			//如有DCC需求，因只有刷卡消费和刷卡预授权才支持DCC交易，此方法只需要在这两种交易类型返回true
			//如果接入方无需支持DCC，则此方法返回false即可
			return false;
		}

        /**
        * 提供给持卡人选择交易币种
        *
        * @param isSuccess
        * @param cardInfo
        * @param rateInfo
        */
        @Override
        public void selectCurrency(boolean isSuccess, CardInfo cardInfo, RateInfo rateInfo) {
            // 1、提供 UI 界面给持卡人选择交易币种

            // 2、选择本币调用
            // cardReaderHandler(isSuccess, cardInfo.getCardType(), cardInfo, null); //EDC

            // 3、选择外币调用
            // cardReaderHandler(isSuccess, cardInfo.getCardType(), cardInfo, rateInfo); //DCC
        }
		
    
        /**
         * 读卡的结果（sdk2.4.1版本及以上新增RateInfo参数，返回DCC读卡流程中进行汇率查询的结果）
         *
         * @param isSuccess 是否成功
         * @param cardType  卡片种类(-1(unknow) 1(msc) 2(ic) 3(nfc) 4(scancode) 5(other))
         * @param cardInfo  读取卡片信息
		 * @param rateInfo  汇率信息
         */
        public void cardReaderHandler(boolean isSuccess, @CardType.Type int cardType, CardInfo cardInfo, RateInfo rateInfo){    
            //读卡成功后才发起交易
            if (!isSuccess || cardInfo == null) {
                Toast.makeText(getApplicationContext(), "读卡失败",  Toast.LENGTH_SHORT).show();
                initCardEvent();
                return;
            }
        
            //1. 根据银联85号文规定，智能终端在消费和预授权完成交易需上送经度，纬度，坐标系信息到卡组织
			CILRequest request = new CILRequest();
			
			request.setLongitude(121.600228);//设置经度
			request.setLatitude(31.180606);//设置纬度
			request.setCoordinates("GCJ02");//设置坐标系
			//关于坐标系，国内一些常用第三方取值：百度（BD09），高德、腾讯（GCJ02），GPS（WGS84）。
			//一般第三方定位SDK都能从定位后返回的位置信息类中取到，详细可查看各第三方接入文档。
			
			
			//2.如果接入方需要进行DCC交易，在消费和预授权完成交易中需将汇率信息填入request中，否则无需处理
			CILRequest request = new CILRequest();
			...
			if (rateInfo != null) {
			    request.setBillingAmt(rateInfo.getBillingAmt());//设置扣账金额
			    request.setBillingCurr(rateInfo.getBillingCurr());//设置扣账币种
			    request.setTransRate(rateInfo.getTransRate());//设置交易汇率
			    request.setBatchNum(rateInfo.getBatchNum());//设置汇率请求批次号
			    request.setTraceNum(rateInfo.getTraceNum());//设置汇率请求流水号
			}
			
			//3. 在这里面发送银行卡相关的交易(如消费、消费撤销、退货、预授权、预授权撤销、预授权完成、预授权完成撤销、余额查询)
            //CILSDK.consume(request, cardType, new Callback<CILResponse>()) //消费
		}
   
        /**
         * 自定义读卡时的缓冲页面
         */
        public void waitLoadingShow(){
        
        }
    
        /**
         * 取消读卡时的缓冲页面
         */
        public void waitLoadingDismiss(){
        
        }
    
        /**
         * 读卡失败
         */
        public void cardHandlerError(Exception e){
        
        }
            
    }
```
<span id="base_request" style="font-size:20px;color:red" >注意:</span><br/>
>调用银行卡类交易接口时，需要传入CILRequest以及CardType，且所有接口中CILRequest均需要传入以下信息：
```
    CILRequest request = new CILRequest();
    request.setAmount(amount);//消费金额
    /**
    * 复写的刷卡回调方法中获取到的cardInfo信息
    */
    request.setCardNumb(cardInfo.getCardNumber());//卡号
    request.setCardExpirationDate(cardInfo.getCardExpirationDate());//卡片有效期
    request.setPinEmv(cardInfo.getPinBins());//卡bin
    request.setCardSequenceNumber(cardInfo.getSequenceSerialNum());//卡片序列号
    request.setField55(cardInfo.getField55());//55域信息
    request.setSecondTrack(cardInfo.getTrack2());//二磁道信息
    request.setOrderId(orderId);//可选参数。（消费、退货、预授权、预授权完成、扫码下单、扫码退货）
    request.setLocation(location)//有终端具备获取位置信息能力时必选上送（ 用于消费 预授权完成）
```

**1、消费**

消费接口[request必须包含参数:](#base_request)
```
    //除了request必须包含的参数外，还需要以下参数
    //复写的刷卡回调的方法中会返回汇率信息
    if (rateInfo != null) {
            request.setBillingAmt(rateInfo.getBillingAmt());//持卡人扣账金额
            request.setBillingCurr(rateInfo.getBillingCurr());//持卡人扣账货币代码
            request.setTransRate(rateInfo.getTransRate());//交易汇率
            request.setBatchNum(rateInfo.getBatchNum());//汇率查询批次号
            request.setTraceNum(rateInfo.getTraceNum());//汇率查询流水号
        }
    //地理位置信息，需要调用方自行获取传入sdk
            request.setLongitude(location.getLongitude());//经度
            request.setLatitude(location.getLatitude());//纬度
            request.setCoordinates(location.getCoordinates());//坐标系
    
    /**
     * 刷卡消费
     *
     * @param request  请求参数
     * @param cardType 卡片类型 CardType.MSC_CARD, CardType.IC_CARD, CardType.NFC_CARD...
     */
    CILSDK.consume(request, cardType, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });

```

**2、撤销**  

撤销接口[request必须包含参数:](#base_request)
```

    /**
     *  撤销接口除基础信息外，
     *  还需要原交易信息
     */
    request.setReferenceNumber(transaction.getRefNum());//原交易参考号
    request.setRevAuthCode(transaction.getRevAuthCode());//原交易授权码
    request.setBatchNum(transaction.getBatchNum());//原交易批次号
    request.setTraceNum(transaction.getTraceNum());//原交易凭证号

    CILSDK.revokeConsume(request, cardType, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });

```

**3、退货**  

退货接口[request必须包含参数:](#base_request)
```

    /**
     *  退货接口除基础信息外
     *  还需要原交易信息
     */
    request.setReferenceNumber(referenceNumber);
    request.setTransDatetime(tradeDate);
    CILSDK.returnConsume(request, cardType, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });

```
**4、余额查询**  

余额查询接口[request必须包含参数:](#base_request)
```
    CILSDK.checkBalance(request, cardType, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });

```

**5、卡预授权**  

预授权接口[request必须包含参数:](#base_request)
```
    
    /**
     * 刷卡预授权
     *
     * @param request  请求参数
     * @param cardType 卡片类型 CardType.MSC_CARD, CardType.IC_CARD, CardType.NFC_CARD...
     */
    CILSDK.preAuth(request, cardType, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
               ...
            }
        });

```

**6、卡预授权撤销**  

预授权撤销接口[request必须包含参数:](#base_request)
```
    /**
     *  预授权撤销接口除基础信息外
     *  还需要原预授权交易信息（需要输入的信息）
     */
    request.setAmount(amount);//交易金额
    request.setRevAuthCode(authCode);//原预授权交易授权码/参考号
    request.setTransDatetime(originalTradeDate);//原预授权交易日期
    CILSDK.revokePreAuth(request, cardType, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });

```

**7、卡预授权完成**  

预授权完成接口[request必须包含参数:](#base_request)  
>注意：预授权完成属于消费类型的交易，默认是做DCC的，在调用接口之前一定要将刷卡回调中的汇率信息设置给request，确保预授权完成的DCC交易能够顺利进行。
```
//除了request必须包含的参数外，还需要以下参数
    //复写的刷卡回调的方法中会返回汇率信息
    if (rateInfo != null) {
            request.setBillingAmt(rateInfo.getBillingAmt());//持卡人扣账金额
            request.setBillingCurr(rateInfo.getBillingCurr());//持卡人扣账货币代码
            request.setTransRate(rateInfo.getTransRate());//交易汇率
            request.setBatchNum(rateInfo.getBatchNum());//汇率查询批次号
            request.setTraceNum(rateInfo.getTraceNum());//汇率查询流水号
        }
    //地理位置信息，需要调用方自行获取传入sdk
            request.setLongitude(location.getLongitude());//经度
            request.setLatitude(location.getLatitude());//纬度
            request.setCoordinates(location.getCoordinates());//坐标系
            
            request.setAmount(amount);//交易金额
            request.setRevAuthCode(authCode);//原预授权交易授权码
            request.setTransDatetime(originalTradeDate);//原预授权交易日期
    CILSDK.preAuthComplete(request, cardType, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
               ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });
```

**8、卡预授权完成撤销**  

预授权完成撤销接口[request必须包含参数:](#base_request)
```
    /**
     *  预授权完成撤销接口除基础信息外
     *  还需要原预授权完成交易信息
     */
    request.setReferenceNumber(refNum);//原预授权完成交易参考号
    request.setRevAuthCode(authCode);//原预授权完成交易授权码
    request.setTraceNum(traceNum);//原预授权完成交易凭证号
    request.setBatchNum(batchNum);//原预授权完成交易批次号
    request.setTransDatetime(originalTradeDate);//原预授权完成交易日期

    CILSDK.revokePreAuthComplete(request, cardType, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });
```

**9、DCC转EDC**  

当交易卡片为外卡时，消费类（消费、预授权完成）交易可选择进行DCC转EDC。  
>注意1：只有做了DCC的交易后才可能DCC转EDC成功，调用消费接口时如果持卡人的卡支持DCC，并且在刷卡页面复写的isOpenDcc方法中也返回true，sdk会优先执行DCC的逻辑，就是默认会走DCC的逻辑。  

>注意2：DCC转EDC无需刷卡，只需要使用消费成功的返回信息就可发起。  

>注意3：DCC转EDC成功后打印时，入参formatTransCode传PER,isForeignTrans传false
```
    CILRequest request = new CILRequest();
    //以下参数取自原交易
    request.setCardNum(cardNum);//卡号
    request.setTransDatetime(datetime);//原交易时间
    request.setAmount(amount);//原交易金额
    request.setBillingAmt(biilingAmt);//原扣币金额
    request.setReferenceNumber(refNum);//原交易参考号
    request.setRevAuthCode(revAuthCode);//原交易授权码
    request.setBatchNum(batchNum);//原交易批次号
    request.setTraceNum(traceNum);//原交易凭证号
    request.setTransCurr(transCurr);//原交易币种
    request.setBillingCurr(billingCurr);//原交易扣款币种
    CILSDK.dccToEdc(request, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse cilResponse) {
            ...
        }

        @Override
        public void onError(Parcelable p, Exception e) {
            ...
        }
    });

```    

**10、预授权手输卡号**  
    ```
    //需要手输的卡信息有卡号和卡有效期两个字段
    
    CILRequest request = new CILRequest();
     request.setAmount(amount);
     request.setCardNumb(cardInfo.getCardNumber());//卡号
     request.setCardExpirationDate(cardInfo.getCardExpirationDate());//卡片有效期
     request.setPosInputStyle("012");
     
    CILSDK.preAuth(request, CardType.MSC_CARD, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
               ...
            }
        });
    ```  
    
**11、预授权撤销手输卡号**  
```
    CILRequest request = new CILRequest();
     
     //卡信息两个字段通过手输代替刷卡
     request.setCardNumb(cardInfo.getCardNumber());//卡号
     request.setCardExpirationDate(cardInfo.getCardExpirationDate());//卡片有效期
     request.setPosInputStyle("012");
     
     //下面三个字段可通过外界手输录入
     request.setAmount(amount);
     request.setTransDatetime(originalTradeDate);//原始交易日期
     request.setRevAuthCode(authCode);//授权码
     
    CILSDK.revokePreAuth(request, CardType.MSC_CARD, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
               ...
            }
        });
    
```  
    
**12、预授权完成手输卡号**   
>注意1：这个地方一定要进行DCC汇率查询，预授权完成属于消费类接口，默认是做Dcc的，同样如果需要做DCC转EDC,在预授权交易成功页面调用Dcc转EDC接口即可。  

>注意2：汇率查询一般sdk在刷卡的流程中自动完成，但是手输类的交易不走sdk的刷卡流程，需要调用方自行调用汇率查询api进行汇率查询，流程如下：
```
graph LR
输入金额/原交易日期/授权码-->手输卡号和卡有效期
    手输卡号和卡有效期--success-->DCC汇率查询
        DCC汇率查询--succ or fail-->预授权完成
```
 
```
    //1.进行DCC汇率查询
    RateQueryListener listener = new RateQueryListener() {
                @Override
                public void onResult(boolean isSupport, RateInfo rateInfo) {
                    //发预授权完成交易
                }
            };
    sendQueryRate(cardInfo.getCardNumber(), listener);//调用父类BaseCardActivity
    
    //2.发预授权交易
    CILRequest request = new CILRequest();
     request.setCardNumb(cardInfo.getCardNumber());//卡号
     request.setCardExpirationDate(cardInfo.getCardExpirationDate());//卡片有效期
     request.setPosInputStyle("012");
    //2.1 汇率 
    if (rateInfo != null) {
            request.setBillingAmt(rateInfo.getBillingAmt());
            request.setBillingCurr(rateInfo.getBillingCurr());
            request.setTransRate(rateInfo.getTransRate());
            request.setBatchNum(rateInfo.getBatchNum());
            request.setTraceNum(rateInfo.getTraceNum());
        }
    //2.2地理位置信息，需要调用者自己获取传入    
    if (mLocationManager != null) {
            request.setLongitude(location.getLongitude());
            request.setLatitude(location.getLatitude());
            request.setCoordinates(location.getCoordinates());
        }
        
            //以下信息可通过外界输入
            request.setAmount(amount);
            request.setTransDatetime(originalTradeDate);//原始交易日期
            request.setRevAuthCode(authCode);//授权码
            
    CILSDK.preAuthComplete(request, CardType.MSC_CARD, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
               ...
            }
        });
```

**13、预授权完成撤销手输卡号**  
```
    CILRequest request = new CILRequest();
     
     //卡信息两个字段通过手输代替刷卡
     request.setCardNumb(cardInfo.getCardNumber());//卡号
     request.setCardExpirationDate(cardInfo.getCardExpirationDate());//卡片有效期
     request.setPosInputStyle("012");
     
     //下面三个字段可通过外界手输录入,也可通过本地数据库保存的交易信息查找
     request.setAmount(amount);
     request.setReferenceNumber(refnum);
     request.setRevAuthCode(authcode;//授权码
     request.setTraceNum(tracenum);
     request.setBatchNum(batchnum);
     request.setTransDatetime(originalTradeDate);//原始交易日期
     
    CILSDK.revokePreAuthComplete(request, CardType.MSC_CARD, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
               ...
            }
        });
    
```



**14、交易结果字段说明**  
  
   字段 | 类型 | 含义 | 备注 | 备注
---|---|---|---|---
additionalResData  |String |受理方标识码 |无|
afterTransCode  |String |原交易类型 |无|
batchNum   |String |批次号 |无|
billingAmt   |String |持卡人扣帐金额 |无|
billingCurr   |String |持卡人扣帐货币代码符号，三位字母 |例如USD|
billingCurrNum   |String |持卡人扣帐货币代码,三位数字 |无|
cardBrand   |String |国际信用卡公司代码 |无|
cardNo   |String |银行卡号 |无|
cardType |String |刷卡方式 |无|
cashierName |String |收银员 |无|
cashierNum |String |收银员号 |无|
clearingDate |String |清算日期 |无|
compInfoA1 |String |签购单收单行 |无|
compInfoA2 |String |签购单商户号 |无|
compInfoA3 |String |签购单终端号 |无|
compInfoA4 |String |markup |无|
compInfoA6 |String |借贷记标识 |无|
compInfoA7 |String |营销信息 |无|
compInfoA8 |String |二维码信息 |无|
coupon |String |支付宝/微信优惠金额 |无|
field55 |String |IC卡交易的TAG信息 |无|
insCode |String |受理方标识码 |无|
localTransDate |String |受卡方所在地日期 |无|
localTransTime |String |受卡方所在地时间 |无|
merCode |String |受卡方标识码（商户号） |无|
merDiscount |String |商家优惠金额 |无|
originTraceNum |String |原交易凭证号 |无|
outOrderNum |String |外部订单号 |无|
posInputStyle |String |服务点输入方式码 |无|
processflag |String |扫码支付09状态的交易是否成功 |附件表1|
refNum |String |检索参考号 |无|
respCode |String |应答码 |"00"表示成功|
revAuthCode |String |授权标识应答码 |无|
revInsCode |String |附加响应数据 |无|
revOrderNum |String |自定义域，用于扫码支付业务。 |无|
scanCodeId |String |扫码号 |无|
termCode |String |终端号 |无|
traceNum |String |受卡方系统跟踪号 |合作方交易流水|
transAmt |String |交易金额 |无|
transCode |String |交易类型码 |无|
transCurr |String |交易货币代码 |无|
transDate |String |原交易日期 |无|
transDatetime |String |受卡方所在地日期＋受卡方所在地时 |无|
transRate |String |持卡人扣帐汇率 |无|
  
    
## 扫码交易

扫码相关的交易则是不依赖 POS 机器的读卡模块的,但是你需要将`微信`或者`支付宝`的二维码读出来传给扫码消费接口,扫码可以使用第三方库,如 `zxing`。
  
**1、扫码消费**
```
    CILRequest request = new CILRequest();
    request.setAmount(amount);//交易金额
    request.setScanCodeId(result);//二维码code
    request.setOrderId(orderId);//外部订单号（可选参数）
    
    CILSDK.consumeQr(request, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse response) {
//        处理扫码消费结果，字段说明见底部
//        如果结果返回`09`，需要查询该订单获取最终结果，如下：
//        resultCode = response.getTrans().getRespCode();
//        if ("09".equals(resultCode)) {
//            CILSDK.queryQr(); // `queryQr` 方法见下文
//        }

        }

        @Override
        public void onError(Parcelable cilRequest, Exception e) {
            // 扫码消费出错
        }
    });  
```  

>注意,如果应答码返回09或98，需要调用讯联扫码查询接口，查询该笔订单的实际状态。

    
**2、扫码撤销**  
```
    CILRequest request = new CILRequest();
    request.setAmount(amount);//原交易金额
    request.setBatchNum(batchNum);//原交易批次号
    request.setTraceNum(traceNum);//原交易凭证(流水)号
    request.setReferenceNumber(refNum);  //原交易参考号

    CILSDK.revokeConsumeQr(request, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
               ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
              ...
            }
        });  
```
    

**3、扫码退货**  

```   
    CILRequest request = new CILRequest();  
    request.setAmount(amount);//退款金额  
    request.setReferenceNumber(serialNum);//原交易参考号  
    request.setTransDatetime(tradeDate);//原交易时间

    CILSDK.returnConsumeQr(request, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
               ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
               ...
            }
        });
```

**4、扫码查询**
```
    CILRequest request = new CILRequest();
    request.setBatchNum(response.getTrans().getBatchNum());//批次号
    request.setTraceNum(response.getTrans().getTraceNum());//凭证号
    request.setPeriod(10000L);//10s
    request.setLimitTime(6);//6 次
    request.setReferenceNumber(response.getTrans().getRefNum());//参考号
    request.setPosInputStyle(response.getTrans().getPosInputStyle());//pos输入服务方式码
    request.setScanCodeId(response.getTrans().getScanCodeId());//扫码号
    request.setAmount(response.getTrans().getTransAmt());//交易金额
    
    CILSDK.queryQr(request, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });
```

**5、扫码消费查询,只查询一次，不含取消接口**  
```
    CILRequest request = new CILRequest();
    request.setBatchNum(response.getTrans().getBatchNum());//批次号
    request.setTraceNum(response.getTrans().getTraceNum());//凭证号
    request.setReferenceNumber(response.getTrans().getRefNum());//参考号
    request.setPosInputStyle(response.getTrans().getPosInputStyle());//pos输入服务方式码
    request.setScanCodeId(response.getTrans().getScanCodeId());//扫码号  
    
    CILSDK.queryQrJustOnce(request, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
               ...
            }
        });
```  
**6、扫码取消**  
```
    /**
     * 扫码取消
     * 对于09状态的消费订单，最终需要取消、关单
     * @param request
     * @param listener
     */  

    CILRequest cilRequest = new CILRequest();
    cilRequest.setOriginalTradeDate(response.getTrans().getTransDate());//原交易日期
    cilRequest.setAmount(response.getTrans().getTransAmt());//交易金额
    cilRequest.setBatchNum(response.getTrans().getBatchNum());//批次号
    cilRequest.setTraceNum(response.getTrans().getTraceNum());//凭证号
    cilRequest.setPosInputStyle(response.getTrans().getPosInputStyle());//pos输入服务方式码
    cilRequest.setReferenceNumber(response.getTrans().getRefNum());//参考号  
    
    CILSDK.voidQr(request, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
               ...
            }
        });  
```	
	

**7、扫码预授权**  
```
    CILRequest request = new CILRequest();
    request.setAmount(amount);
    request.setScanCodeId(result);//二维码code
    request.setOrderId(orderId);//外部订单号（可选参数）
	
	CILSDK.preAuthQr(request, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
            // 处理扫码预授权结果
            // 如果结果返回`09`，需要查询该订单获取最终结果，如下：
            /**
             *      resultCode = response.getTrans().getRespCode();
             *      if ("09".equals(resultCode)
             ) {
             *          CILSDK.queryQr(); // `queryQr` 方法见下文
             *      }
             */
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                //扫码预授权出错
            }
        });  
```		
	

**8、扫码预授权撤销**
```
	CILRequest request = new CILRequest();
    request.setAmount(transAmt);//原预授权交易金额
    request.setReferenceNumber(revAuthCode);//原预授权交易参考号
    request.setTransDatetime(transDate);//原预授权交易时间
    request.setBatchNum(curTrans.getBatchNum());//原预授权交易批次号
    request.setTraceNum(curTrans.getTraceNum());//原预授权交易凭证（流水）号
	
	CILSDK.revokePreAuthQr(request, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
               ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
               ...
            }
        });  
		
```	

**9、扫码预授权完成**
```
		CILRequest request = new CILRequest();
        request.setAmount(transAmt);//原预授权交易金额
        request.setReferenceNumber(revAuthCode);//原预授权交易参考号
        request.setTransDatetime(transDate);//原预授权交易时间
        CILSDK.preAuthCompleteQr(request, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });  
```		

**10、扫码预授权完成撤销**
```
	CILRequest request = new CILRequest();
    request.setAmount(transaction.getTransAmt());//原预授权完成交易金额
    request.setBatchNum(transaction.getBatchNum());//原预授权完成交易批次号
    request.setTraceNum(transaction.getTraceNum());//原预授权完成交易凭证（流水）号
    request.setReferenceNumber(transaction.getRefNum());//原预授权完成交易参考号

    CILSDK.revokePreAuthCompleteQr(request, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse cilResponse) {
            ...
        }

        @Override
        public void onError(Parcelable cilRequest, Exception e) {
            ...
        }
    });
```




**11、单品券功能接入指引**   
> 注意：sdk v2.5.3及之后版本支持该功能
1. 扫码消费接口增加单品券核销功能，扫码消费接口增加入参两个参数，订单优惠标记（为配券时候的填的优惠标记）,商品列表；
```
    /**
     * 扫码消费
     */
    CILRequest request = new CILRequest();
    request.setAmount(amount);
    request.setScanCodeId(result);//二维码code
    request.setOrderId(orderId);//外部订单号（可选参数）
    
    request.setOrderPromotionMark(orderPromotionMark);//订单优惠标记（类型String可选参数）
    request.setGoodsList(goodsList);//商品列表（类型String可选参数）详见下新增字段说明

    CILSDK.consumeQr(request, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse response) {
            // 处理扫码消费结果
            // 如果结果返回`09`，需要查询该订单获取最终结果，如下：
            /**
             *      resultCode = response.getTrans().getRespCode();
             *      if ("09".equals(resultCode)
             ) {
             *          CILSDK.queryQr(); // `queryQr` 方法见下文
             *      }
             */

        }

        @Override
        public void onError(Parcelable cilRequest, Exception e) {
            // 扫码消费出错
        }
    });
```  
* 新增字段说明  

字段 | 类型 | 含义 | 是否可选 | 备注1 | 备注2
---|---|---|---|---|---
orderPromotionMark   |String |订单优惠标记 |Y|ans32|来源于配券时选填字段
goodsList |String |商品列表 |Y|最多9个商品|按照以下样例格式传输  

报文样例：
```
"orderPromotionMark":"1111",
"[
    {
        "goodsName":"小面包",
        "price":"1",
        "goodsNum":"1",
        "goodsId":"1111"
    },
    {
        "goodsName":"棒棒糖",
        "price":"1",
        "goodsNum":"1",
        "goodsId":"2222"
    },
    {
        "goodsName":"彩虹糖",
        "price":"1",
        "goodsNum":"1",
        "goodsId":"3333"
    },
    {
        "goodsName":"矿泉水",
        "price":"1",
        "goodsNum":"1",
        "goodsId":"4444"
    }
]"
```
* 响应报文新增一个CouponInfo的实例,内部包含属性字段有：

字段 | 类型 | 含义 | 是否可选 | 备注
---|---|---|---|---
couponId   |String |优惠券id |Y|
couponName   |String |优惠券名称 |Y|
channelContribution    |String |渠道出资 |Y| 
merchantContribution    |String |商家出资 |Y| 
otherContribution    |String |其它出资 |Y| 
discountType    |String |优惠类型 |Y| 
discountRange    |String |优惠范围 |Y| 
discountbatchaId    |String |优惠活动批次ID |Y| 
goodsList |String |单品优惠商品列表 |Y|  


* 单品优惠列表商品字段  
 
字段 | 类型 | 含义 | 是否可选 | 备注
---|---|---|---|---
goodsBarCode   |String |商品条码号 |Y|
goodsDiscount   |String |单品优惠金 |Y| 

  
CouponInfo响应报文样例：
```
{
    "MerchantContribution":"000000000100",
    "couponId":"9026256969",
    "couponName":"讯联满1.1减1 tag",
    "discountRange":"SINGLE",
    "discountType":"DISCOUNT",
    "discountbatchaId":"9803978",
    "goodsList":"[{"goodsBarCode":"1111","goodsDiscount":"000000000034"}, {"goodsBarCode":"2222","goodsDiscount":"000000000034"}, {"goodsBarCode":"3333","goodsDiscount":"000000000032"}]",
    "otherContribution":"000000000000"
}
```  
附：最多支持传入9个商品    


## 账单查询

智能 POS SDK 分别提供了最多30天的`账单列表查询`和`账单统计接口`接口,接口会根据 type 值确定返回`银行卡账单`或`扫码账单`。
交易成功还是失败最终以返回账单中应答码为准，见[应答码表](https://gongluis.github.io/Smart-POS/attached/sdkAnserCode/) 。


> 交互设计建议：交易中，具体来说，调用CILSDK.consumeQr()或是CILSDK.consume()方法时，当因为网络中断进入onError callback时，建议在交互中加入`查询账单列表`的逻辑，这样交易失败后可方便收银员通过账单来确认这笔订单的实际状态。
 
> 注意：对于 `09`（请求正在处理中）状态的交易账单数据，还需要看 `处理标志位` 才能判定此次交易成功与否，使用 `getProcessFlag()` 获取。
当 `processFlag 为 '0'` 时，此次交易成功；当 `processFlag 为非'0'` 时，此次交易失败，见[处理标志表](/user-guide/attachments/#_1)。
  
  
  
**1、根据外部订单号获取该笔订单信息（同步）**  
```
//outOrderNum为传入数据
CILSDK.getBillsAsync(outOrderNum, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        })
```  
**2、根据凭证号获取当前批次号下该笔订单详情(异步)**    

```
/**
* 依据凭证号，获取当前批次下的订单详情 回调在主线程
* 异步操作
* @param traceNum 凭证号
* @param listener
*/
CILSDK.getBillByTraceNum(traceNum, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });
```  
**3、根据参考号获取当前批次下的订单详情（异步操作）**
```
/**
* @param refNum 参考号
* @param callBackIsOnMainThread 是否在主线程中回调
*/
CILSDK.getBillByRefNumAsync(refNum, callBackIsOnMainThread, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                    ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                   ...
            }
        });
```

**4、根据批次号获取当前批次下的订单详情（异步操作）**  
```

/**
* @param refNum 批次号
* @param callBackIsOnMainThread 是否在主线程中回调
*/
CILSDK.getBillByBatchNumAsync(batchNum, callBackIsOnMainThread, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                ...
            }
        });
```

**5、获取账单列表**
```
 1. 默认获取三十天的账单列表
    /**
     * 获取账单列表 异步
     *
     * @param int page 从0开始
     * @param int size 每页返回的条数
     * @param int type 账单类型(TransConstants.CARD_BILL, TransConstants.QR_BILL, TransConstants.ALL_BILL)
     *
     */
    CILSDK.getBillsAsync(page, size, @BillType int type, new Callback<CILResponse>() {
        @Override
        public void onResult(final CILResponse response) {
            if (null != response && 0 == response.getStatus()) {
                //账单获取成功
                Trans[] trans = response.getTxn(); //账单数据,字段详情见 Trans
            }
        }

        @Override
        public void onError(Parcelable p, Exception ex) {
            //账单获取出错
        }
    });
    
    2. 获取指定区间的账单列表
        /**
     * 获取账单列表 异步
     *
     * @param page
     * @param size
     * @param startTime 查询订单开始时间，格式：yyyyMMdd
     * @param endTime 查询订单结束时间，格式：yyyyMMdd
     * @param type
     * @param callBackIsOnMainThread
     * @param listener
     * @return
     */
    CILSDK.getBillsAsync(page, size, startTime, endTime, type, true, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                   ...
            }

            @Override
            public void onError(Parcelable cilRequest, Exception e) {
                   ...
            }
        });
    
```  


**6、获取今日交易统计**      
```
    /**
     * 获取今日交易统计 异步
     * @param type 账单类型
     * <ul>
     *      <li>TransConstants.ALL_BILL:所有账单</li>
     *      <li>TransConstants.CARD_BILL:银行卡账单</li>
     *      <li>TransConstants.QR_BILL:扫码账单</li>
     * </ul>
     *
     */
    CILSDK.getBillStatAsync(@BillType int type, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse response) {
            if (null != response && 0 == response.getStatus()) {
                //今日交易统计获取成功
                CILResponse.Info data = response.getData();
                String transCount = data.getTransCount();//今日银行卡正向交易数量
                String transAmtSum = data.getTransAmtSum();//今日银行卡正向总交易金额
                String backTransCount = data.getBackTransCount();//今日银行卡负向交易数量
                String backTransAmtSum = data.getBackTransAmtSum();//今日银行卡负向总交易金额
                String sctTransCount = data.getSctTransCount();//今日扫码正向交易笔数
                String sctTransAmtSum = data.getSctTransAmtSum();//今日扫码正向交易金额
                String sctBackTransCount = data.getSctBackTransCount();//今日扫码负向交易笔数
                String sctBackTransAmtSum = data.getSctBackTransAmtSum();//今日扫码负向交易金额
                String tipsTransCount = data.getTipsTransCount();//今日收取小费笔数
                String tipsTransAmtSum = data.getTipsTransAmtSum();//今日收取小费金额
            }

        }

        @Override
        public void onError(Parcelable p, Exception ex) {
            //今日交易统计获取出错
        }
    });
    
```

> SDK 的网络部分使用的是第三方库 `okhttp`,以上账单接口分别还提供了相对应的同步接口 `getBills` 和 `getBillStat` 。
对于异步接口来说,都会返回一个 `Call` 对象,你可以在应用出错的时候调用 `call.cancel()` 取消这次请求,以免造成内存泄露。  

## 小费 
> 注意:
通过对一笔交易收取小费。  
小费最多收取交易金额的20%  
小费只能成功收取一次
消费撤销后不能再次收取  

**1、小费**   

```  

        CILRequest request = new CILRequest();
        request.setCardNum(cardNumber);
        request.setReferenceNumber(trans.getRefNum());
        request.setBatchNum(trans.getBatchNum());
        request.setTraceNum(trans.getTraceNum());
        request.setTransDatetime(trans.getTransDatetime());
        request.setRevAuthCode(trans.getRevAuthCode());
        
        request.setAmount(amount);//小费金额

        CILSDK.takeTip(request, new     Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable parcelable, Exception e) {
                ...
            }
        });
```
**2、小费撤销**  

```
        CILRequest request = new CILRequest();
        request.setCardNum(cardNo);
        request.setAmount(trans.getTransAmt());
        request.setReferenceNumber(trans.getRefNum());
        request.setRevAuthCode(trans.getRevAuthCode());
        request.setBatchNum(trans.getBatchNum());
        request.setTraceNum(trans.getTraceNum());
        request.setTransDatetime(trans.getTransDatetime());  
        
        CILSDK.revokeTip(request, new Callback<CILResponse>() {
            @Override
            public void onResult(CILResponse cilResponse) {
                ...
            }

            @Override
            public void onError(Parcelable parcelable, Exception e) {
                ...
            }
        });
```  

## 结算

结算需求主要用于每日交易结束时或收银员交接班时,对某段时间内的账款核对。商户每日交易结束后,收银员需要统计并核对所有的交易,核对交易统计准确后结算，打印出结算单。
结算会涉及到一个概念--`批次号`,我们在前面的交易都会传入一个批次号给 request ,调用结算之后,后续的交易需要将这个批次号加1,因为此批次已经打包结算掉了。

```
    //batchNum 批次号
    CILSDK.transSettleAsync(batchNum, new Callback<CILResponse>() {
        @Override
        public void onResult(final CILResponse response) {
            //打印结算单
        }

        @Override
        public void onError(Parcelable cilRequest, Exception e) {
            //结算出错
        }
    });
```

如果使用讯联样式结算UI、逻辑直接使用

>注意,使用SettleDaoUtil工具类是必须先初始化，SettleDaoUtil.getInstance().init();

```

    SettleDaoUtil.getInstance().gotoLiquidation(context)
```    

## 打印


本模块可用于根据交易信息打印所需的消费票据。接口不仅提供了一套固定格式的小票样式，而且还可以根据需要自定义打印样式。
主要功能包括小票打印、二维码打印、条形码打印以及图片打印。

<p style="color:red; font-size:18">特别注意: 中国人民银行和中国银联为了规范市场上的POS机终端，要求终端打印的签购单必须合乎规范，规范内容包括必须打印的字段与正确的字段内容。</p>

签购单规范详情见 [签购单规范](https://gongluis.github.io/Smart-POS/attached/ticket)


* 打印银行卡类交易、扫码类交易小票
```
    /** trans 交易信息,Trans类型
     *  lineBreak 小票结尾需要走纸换行的行数，int类型
     *  formatTransCode @FormatTransCode String类型，小票的交易类型
     *  kind @ReceiptSubtitle int类型，小票的子标题，判断是商户联或者是客户联
     *  isForeignTrans 是否是外卡类交易
     *  bitmap logo图标，没有直接传null
     */
    CILSDK.printKindsReceipts(trans,lineBreak,formatTransCode, kind, isForeignTrans,  bitmap, new Callback<PrinterResult>(){
         @Override
         public void onResult(PrinterResult response) {
             if (null ！= printerResult && !"打印成功".equals(printerResult.toString())) {
                 //打印成功
             }
         }
           
          @Override
          public void onError(Parcelable cilRequest, Exception e) {
                 //打印失败
          }
    }); 
```

* 打印结算小票

```
    /**
     * 打印结算小票
     *
     * transSettles    结算信息List
     * transDatetime   结算时间
     * lineBreak       打印结尾换行数
     * formatTransCode 结算类型；TransConstants.TRANS_SETTLE_DETAILS：结算详情小票；TransConstants.TRANS_SETTLE_TOTAL：结算统计小票
     * callback 回调
     */
     CILSDK.printSettleReceipts(transSettles, transDatetime,batchNum, lineBreak, formatTransCode, new Callback<PrinterResult>() {
          @Override
          public void onResult(PrinterResult result) {
              if (null != result && !"打印成功".equals(result.toString())){
                                           
               }
          }    
          @Override
          public void onError(Parcelable cilRequest, Exception e) {
                                       
          }
     });

```

* 自定义打印

> 注意：使用自定义打印方法时，若打印内容超过2000个字符，请使用分段打印方式，否则可能出现DeviceRTException

```
    /**
     * 根据二进制数据打印(根据打印规范用户自定义打印小票样式)
     *
     * buffer    打印内容
     * lineBreak 换行数
     * callback  回调
     */
   CILSDK.printBufferReceipt(buffer, lineBreak,new Callback<PrinterResult>() {
        @Override
        public void onResult(PrinterResult result) {
            if (null != result && !"打印成功".equals(result.toString())){
                                                                                                                                         
             }
        }    
        @Override
        public void onError(Parcelable cilRequest, Exception e) {
                                                                                                                                     
        }
    });

```

* 打印二维码

```
    /**
     * 打印二维码
     *
     * qrCode   二维码内容
     * position 打印位置  0:左对齐；1居中；2：右对齐
     * width    二维码宽度
     * callback 回调
     *
     */
    CILSDK.printQRCode(qrCode,position,width,lineBreak,new Callback<PrinterResult>() {
         @Override
         public void onResult(PrinterResult result) {
             if (null != result && !"打印成功".equals(result.toString())){
                                                                                                  
             }
         }    
         @Override
         public void onError(Parcelable cilRequest, Exception e) {
                                                                                              
         }
    });
```

* 打印条形码

```
    /**
     * 打印条形码
     *
     * barCode   条形码数字
     * position  条形码位置 0:左对齐；1居中；2：右对齐
     * lineBreak 换行数
     * callback  回调
     */
    CILSDK.printBarCode(String barCode, int position, int lineBreak, new Callback<PrinterResult>() {
         @Override
         public void onResult(PrinterResult result) {
             if (null != result && !"打印成功".equals(result.toString())){
                                                                                                                                                                                                                        
              }
         }    
         @Override
         public void onError(Parcelable cilRequest, Exception e) {
                                                                                                                                                                                                                    
         }
    });
```

* 打印图片

```
    /**
     * 打印图片
     *
     * bitmap    图片Bitmap
     * lineBreak 换行数
     * offset    偏移量
     * callback  回调
     *
     */
    CILSDK.printImage(bitmap, lineBreak, offset, new Callback<PrinterResult>() {
         @Override
         public void onResult(PrinterResult result) {
             if (null != result && !"打印成功".equals(result.toString())){
                                                                                                                                                   
             }
         }    
         @Override
         public void onError(Parcelable cilRequest, Exception e) {
                                                                                                                                               
         }
    });
```    


## 其他设置

考虑到使用 SDK 的时候可能还会有其他需求,比如`获取 POS 机的 SN 号`、`设置密钥索引`等,在这里,我们也提供了一部分接口。

* 获取 SDK 版本

```
    //版本名
    String versionName = CILSDK.VERSION_NAME;
    //版本号
    int versionCode = CILSDK.VERSION_CODE;
```

* 获取 SN 号

```
    //SN号
    String snCode = CILSDK.getDeviceSN();
```

* 设置流水号

```
    //serialNum范围：1~999999
    boolean isSuccess = CILSDK.setSerialNum(int serialNum);
```

* 获取流水号

```
    //序列号
    int serialNum = CILSDK.getSerialNum();
```

* 设置批次号

```
    //batchNum范围：1~999999
    boolean isSuccess = CILSDK.setBatchNum(int batchNum);
```

* 获取批次号

```
    //批次号
    int batchNum = CILSDK.getBatchNum();
```

<!--* 设置密钥索引-->

* 设置联迪密钥区
```
    //1-15的设值范围
    CILSDK.setTingA8KeyIndex(2);
```
* 设置密钥索引  
```
    //分别对应MAIN MAC PIN MES
    //1-255的设值范围
    //可以使用下方提供数值,也可以根据自身程序设值
     CILSDK.setTingKeyIndex(4,101,10,150);
```
以上两个方法请在连接刷卡器(CILSDK.connect)之前使用  

**工具类**

* CILPayUtil

```

    /**
     * 将respCode翻译成对应中文解释
     */
    CILPayUtil.translate(context, respCode));

    /**
     * 将Trans类中的transCode翻译成打印所需的对象
     */
    CILPayUtil.getFormatTransCode(transCode);

    /**
     * 根据billingCurr判断交易是否为外卡类的DCC交易
     */
    CILPayUtil.isDCCPay(billingCurr);
    /**
     * 判断交易是否成功
     */
    CILPayUtil.isTransSuccess(trans);
    /**
     * 根据Trans类中的transCode判断交易是否属于扫码类交易
     */
    CILPayUtil.isQrPay(transCode);

```

* ReceiptFormatUtils

```
    /**
     * 根据Trans类中的TransCode翻译成对应的中文解释
     */
    ReceiptFormatUtils.getTransType(transCode);
    /**
     * 将明文的卡号修改为"前六后四中间为四个*"的样式
     */
    ReceiptFormatUtils.handleCardNum(cardNum);

```

**许可证**

Copyright (c) 2016 cardinfolink.com

**JAVADOC**

java document 详情见 [javadoc](sdkapi/javadoc/)  





