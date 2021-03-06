<h1 align="center"> Java sdk 智能合约 </h1>


## 介绍

这个章节介绍如何通过Java SDK使用智能合约。

## 智能合约部署、调用、事件推送

> Note:目前java-sdk支持neo和WASM智能合约部署和调用，NEO和WASM合约部署和调用方法不同，见下面详解：


### 部署

通过[SmartX]()编译智能合约，可以在SmartX上直接部署合约，也可以通过java sdk部署合约。

```java
InputStream is = new FileInputStream("/Users/sss/dev/test/IdContract/IdContract.avm");
byte[] bys = new byte[is.available()];
is.read(bys);
is.close();
code = Helper.toHexString(bys);
dnaSdk.setCodeAddress(Address.AddressFromVmCode(code).toHexString());

//部署合约
Transaction tx = dnaSdk.vm().makeDeployCodeTransaction(code, true, "name",
                    "v1.0", "author", "email", "desp", account.getAddressU160().toBase58(),dnaSdk.DEFAULT_DEPLOY_GAS_LIMIT,500);
String txHex = Helper.toHexString(tx.toArray());
dnaSdk.getConnect().sendRawTransaction(txHex);
//等待出块
Thread.sleep(6000);
DeployCodeTransaction t = (DeployCodeTransaction) dnaSdk.getConnect().getTransaction(txHex);
```

**DeployCode**

```java
public DeployCode makeDeployCodeTransaction(String codeStr, boolean needStorage, String name, String codeVersion, String author, String email, String desp,String payer,long gaslimit,long gasprice) 

```

| 参数      | 字段   | 类型  | 描述 |             说明 |
| ----- | ------- | ------ | ------------- | ----------- |
| 输入参数 | codeHexStr| String | 合约code十六进制字符串 | 必选 |
|        | needStorage    | Boolean | 是否需要存储   | 必选 |
|        | name    | String  | 名字       | 必选|
|        | codeVersion   | String | 版本       |  必选 |
|        | author   | String | 作者     | 必选 |
|        | email   | String | emal     | 必选 |
|        | desp   | String | 描述信息     | 必选 |
|        | VmType   | byte | 虚拟机类型     | 必选 |
|        | payer   | String | 支付交易费用的账户地址     | 必选 |
|        | gaslimit   | long | gaslimit    | 必选 |
|        | gasprice   | long | gas价格   | 必选 |
| 输出参数 | tx   | Transaction  | 交易实例  |  |


**DeployWasmCode**

```java
DeployWasmCode tx = dnaSdk.wasmvm().makeDeployCodeTransaction(code, "helloWorld", "1.0", "NashMiao",
                "wdx7266@vip.qq.com", "wasm contract for java sdk test", payer.getAddressU160(),
                500, 25000000);

```

### 调用

#### NEO智能合约调用

* 基本流程：

 1. 构造调用智能合约调用参数；
 2. 构造交易；
 3. 交易签名(预执行不需要签名)；
 4. 发送交易。

* 示例

```java

List paramList = new ArrayList<>();
paramList.add("testHello".getBytes());

List args = new ArrayList();
args.add(true);
args.add(100);
args.add("test".getBytes());
args.add("test");
args.add(account.getAddressU160().toArray());

paramList.add(args);
byte[] params = BuildParams.createCodeParamsScript(paramList);

String result = invokeContract(params, account, 20000, 500,true);
System.out.println(result);

public static String invokeContract(byte[] params, Account payerAcct, long gaslimit, long gasprice, boolean preExec) throws Exception{
    if(payerAcct == null){
        throw new SDKException("params should not be null");
    }
    if(gaslimit < 0 || gasprice< 0){
        throw new SDKException("gaslimit or gasprice should not be less than 0");
    }
    Transaction tx = dnaSdk.vm().makeInvokeCodeTransaction(Helper.reverse(contractAddress),null,params,payerAcct.getAddressU160().toBase58(),gaslimit,gasprice);
    dnaSdk.addSign(tx, payerAcct);
    Object result = null;
    if(preExec) {
        result = dnaSdk.getConnect().sendRawTransactionPreExec(tx.toHexString());
    }else {
        result = dnaSdk.getConnect().sendRawTransaction(tx.toHexString());
        return tx.hash().toString();
    }
    return result.toString();
}
```



#### WASM智能合约调用

* 基本流程：
  1. 构造调用合约中的方法需要的参数；
  2. 构造交易；
  3. 交易签名(如果是预执行不需要签名)；
  4. 发送交易。

* 示例：

```java
String contractHash = "bf8ee176c360f7a77b9c45b6faab213bc50eaf5d";
List<Object> params = new ArrayList<>(Arrays.asList(1, 2));
InvokeWasmCode tx = dnaSdk.wasmvm().makeInvokeCodeTransaction(contractHash, "add", params, payer.getAddressU160(), 500, 25000000);

dnaSdk.signTx(tx, new Account[][]{{payer}});
JSONObject result = (JSONObject) dnaSdk.getRestful().sendRawTransactionPreExec(tx.toHexString());

params = new ArrayList<>(Arrays.asList(-2, 3));
tx = dnaSdk.wasmvm().makeInvokeCodeTransaction(contractHash, "add", params, payer.getAddressU160(), 500, 25000000);

dnaSdk.signTx(tx, new Account[][]{{payer}});
result = (JSONObject) dnaSdk.getRestful().sendRawTransactionPreExec(tx.toHexString());
```

#### 智能合约调用例子

合约中的方法
```c#

public static bool Transfer(byte[] from, byte[] to, object[] param)
{
    StorageContext context = Storage.CurrentContext;

    if (from.Length != 20 || to.Length != 20) return false;

    for (int i = 0; i < param.Length; i++)
    {

        TransferPair transfer = (TransferPair)param[i];
        byte[] hash = GetContractHash(transfer.Key);
        if (hash.Length != 20 || transfer.Amount < 0) throw new Exception();
        if (!TransferNEP5(from, to, hash, transfer.Amount)) throw new Exception();

    }
    return true;
}
struct TransferPair
{
     public string Key;
     public ulong Amount;
}
```

Java-SDK 调用Transfer函数的方法

分析：合约中的Transfer方法需要三个参数，前两个参数都是字节数组类型的参数，最后一个参数是对象数组，数组中的每个元素的结构可以通过TransferPair知道各个属性数据类型。

```java
String functionName = "Transfer";
//构造Transfer方法需要的param 数组
List list = new ArrayList();
List list2 = new ArrayList();
list2.add("Atoken");
list2.add(100);
list.add(list2);
List list3 = new ArrayList();
list3.add("Btoken");
list3.add(100);
list.add(list3);
//设置函数需要的参数
func.setParamsValue(account999.getAddressU160().toArray(),Address.decodeBase58("AacHGsQVbTtbvSWkqZfvdKePLS6K659dgp").toArray(),list);
String txhash = dnaSdk.neovm().sendTransaction(Helper.reverse("44f1f4ee6940b4f162d857411842f2d533892084"),acct,acct,20000,500,func,false);
Thread.sleep(6000);
System.out.println(dnaSdk.getConnect().getSmartCodeEvent(tx.hash().toHexString()));
```



> 如果需要监控推送结果，可以了解下面章节。

### 智能合约事件推送

创建websocket线程，解析推送结果。


#### 1. 设置websocket链接


```java

String ip = "http://127.0.0.1";
String wsUrl = ip + ":" + "20335";
DnaSdk wm = DnaSdk.getInstance();
wm.setWesocket(wsUrl, lock);
wm.setDefaultConnect(wm.getWebSocket());
wm.openWalletFile("AssetDemo.json");

```


#### 2. 启动websocket线程


```java
//false 表示不打印回调函数信息
dnaSdk.getWebSocket().startWebsocketThread(false);

```


#### 3. 启动结果处理线程


```java
Thread thread = new Thread(
                    new Runnable() {
                        @Override
                        public void run() {
                            waitResult(lock);
                        }
                    });
            thread.start();
            //将MsgQueue中的数据取出打印
            public static void waitResult(Object lock) {
                    try {
                        synchronized (lock) {
                            while (true) {
                                lock.wait();
                                for (String e : MsgQueue.getResultSet()) {
                                    System.out.println("RECV: " + e);
                                    Result rt = JSON.parseObject(e, Result.class);
                                    //TODO
                                    MsgQueue.removeResult(e);
                                    if (rt.Action.equals("getblockbyheight")) {
                                        Block bb = Serializable.from(Helper.hexToBytes((String) rt.Result), Block.class);
                                        //System.out.println(bb.json());
                                    }
                                }
                            }
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
```


#### 4. 每6秒发送一次心跳程序，维持socket链接


```java
for (;;){
                    Map map = new HashMap();
                    if(i >0) {
                        map.put("SubscribeEvent", true);
                        map.put("SubscribeRawBlock", false);
                    }else{
                        map.put("SubscribeJsonBlock", false);
                        map.put("SubscribeRawBlock", true);
                    }
                    //System.out.println(map);
                    dnaSdk.getWebSocket().setReqId(i);
                    dnaSdk.getWebSocket().sendSubscribe(map);     
                Thread.sleep(6000);
            }
```


#### 5. 推送事例详解


以调用存证合约的put函数为例，

//存证合约abi.json文件部分内容如下

```json
{
    "hash":"0x27f5ae9dd51499e7ac4fe6a5cc44526aff909669",
    "entrypoint":"Main",
    "functions":
    [

    ],
    "events":
    [
        {
            "name":"putRecord",
            "parameters":
            [
                {
                    "name":"arg1",
                    "type":"String"
                },
                {
                    "name":"arg2",
                    "type":"ByteArray"
                },
                {
                    "name":"arg3",
                    "type":"ByteArray"
                }
            ],
            "returntype":"Void"
        }
    ]
}
```


当调用put函数保存数据时，触发putRecord事件，websocket 推送的结果是{"putRecord", "arg1", "arg2", "arg3"}的十六进制字符串

例子如下：

```json
RECV: 
{
	"Action": "Log",
	"Desc": "SUCCESS",
	"Error": 0,
	"Result": {
		"Message": "Put",
		"TxHash": "8cb32f3a1817d88d8562fdc0097a0f9aa75a926625c6644dfc5417273ca7ed71",
		"ContractAddress": "80f6bff7645a84298a1a52aa3745f84dba6615cf"
	},
	"Version": "1.0.0"
}
RECV: {
      	"Action": "Notify",
      	"Desc": "SUCCESS",
      	"Error": 0,
      	"Result": [{
      		"States": ["7075745265636f7264", "507574", "6b6579", "7b2244617461223a7b22416c6772697468656d223a22534d32222c2248617368223a22222c2254657874223a2276616c75652d7465737431222c225369676e6174757265223a22227d2c2243416b6579223a22222c225365714e6f223a22222c2254696d657374616d70223a307d"],
      		"TxHash": "8cb32f3a1817d88d8562fdc0097a0f9aa75a926625c6644dfc5417273ca7ed71",
      		"ContractAddress": "80f6bff7645a84298a1a52aa3745f84dba6615cf"
      	}],
      	"Version": "1.0.0"
      }
```


## FAQ


* contractAddress是什么

contractAddress是智能合约的唯一标识。

* 如何获得contractAddress ？

```java
InputStream is = new FileInputStream("IdContract.avm");
byte[] bys = new byte[is.available()];
is.read(bys);
is.close();
code = Helper.toHexString(bys);
System.out.println("Code:" + Helper.toHexString(bys));
System.out.println("CodeAddress:" + Address.AddressFromVmCode(code).toHexString());
```

> Note: 在获得codeAddress的时候，需要设置该合约需要运行在什么虚拟机上，目前支持的虚拟机是NEO和WASM。

* 调用智能合约invokeTransaction的过程，sdk中具体做了什么

```java
//step1：构造交易
//需先将智能合约参数转换成vm可识别的opcode
Transaction tx = dnaSdk.vm().makeInvokeCodeTransaction(contractAddr, null, contract.toArray(), VmType.Native.value(), sender.toBase58(),gaslimit，gasprice);

//step2：对交易签名
dnaSdk.signTx(tx, info1.address, password);

//step3：发送交易
dnaSdk.getConnectMgr().sendRawTransaction(tx.toHexString());
```

* invoke时为什么要传入账号和密码

调用智能合约时需要用户签名，如果是预执行不需要签名，钱包中保存的是加密后的用户私钥，需要密码才能解密获取私钥。


* 查询资产操作时，智能合约预执行是怎么回事，如何使用？

如智能合约get相关操作，从智能合约存储空间里读取数据，无需走节点共识，只在该节点执行即可返回结果。发送交易时调用预执行接口。
```java
String result = (String) sdk.getConnect().sendRawTransactionPreExec(txHex);
```
