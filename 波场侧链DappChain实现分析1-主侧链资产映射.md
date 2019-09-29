
本文主要介绍的是波场的侧链技术DappChain，在介绍DappChain之前，首先要说一下波场的SunNetwork计划，Sun Network 计划是波场主网的扩容计划, 包含智能合约应用侧链(DAppChain), 跨链通讯等一些列扩容项目。DAppChain 为S的侧链项目，可以为波场主网提供无限扩容能力,并且可以大量削减能量消耗，提高运行效率。下面是官方资料的链接：

官方文档：https://tron.network/sunnetwork/doc/zh/guide/
github：https://github.com/tronprotocol/sun-network

本文主要的是介绍DappChain的技术实现细节,DappChain的基础介绍，可以先阅读官方文档。

### TRC20合约映射
我们以trc20合约映射为例介绍侧链的运作原理。下面是系统的整个架构图：
![image.png-163.9kB][1]

用户如果想要在侧链上流通某个trc20的资产的前提是这个trc20合约的owner事先已经将合约映射到侧链上。合约部署者（owner）可以使用sdk的mappingTrc20方法将合约映射到侧链上，所谓的映射到侧链就是在侧链自动创建一个trc20的合约。sdk的mappingTrc20方法调用的其实是主链的mainchain gateway合约提供了mappingTRC20方法，在这个方法中会触发一个TRC20Mapping事件，oracle模块会监听mainchain gateway的所有事件， 如果监听到TRC20Mapping事件，会通过交易方式调用sidechain的multiSignForDeployDAppTRC20AndMapping方法去在侧链上创建TRC20合约，当然这个新创建的合约代码和主链的TRC20不完全一样，在侧链上，他是使用一个标准的TRC20模版创建的，只要满足tc20的接口标准即可。另外mappingTRC20事件中会附带将主链的TRC20的地址传递到侧链的Gateway合约， 侧链的Gateway合约中会保存主链的TRC20的地址和侧链的TRC20地址的映射关系。
我们分析一下主链GateWay的mappingTRC20函数，代码如下：
```java
//映射trc20合约到侧链
//1.收取映射费用， 如果附带的费用过多， 则退还多余的费用
//2.计算需要被映射的trc20地址并判断合约是否真实存在
//3.保存映射
//4.发送事件
function mappingTRC20(bytes memory txId) payable public goDelegateCall onlyNotStop onlyNotPause isHuman returns (uint256) {
        //----------------------------------------------
        //1.收取映射费用， 如果附带的费用过多， 则退换还
        //----------------------------------------------
        require(msg.value >= mappingFee, "trc20MappingFee not enough");
        if (msg.value > mappingFee) {
            msg.sender.transfer(msg.value - mappingFee);
        }
        bonus += mappingFee;
        //----------------------------------------------
        //2.计算需要被映射的trc20地址并判断合约是否真实存在
        //----------------------------------------------
        address trc20Address = calcContractAddress(txId, msg.sender);
        require(trc20Address != sunTokenAddress, "mainChainAddress == sunTokenAddress");
        require(mainToSideContractMap[trc20Address] != 1, "trc20Address mapped");
        uint256 size;
        assembly {size := extcodesize(trc20Address)}
        require(size > 0);
        //----------------------------------------------
        //3.保存映射
        //----------------------------------------------
        userMappingList.push(MappingMsg(trc20Address, DataModel.TokenKind.TRC20, DataModel.Status.SUCCESS));
        mainToSideContractMap[trc20Address] = 1;
        //----------------------------------------------
        //4.发送事件
        //----------------------------------------------
        emit TRC20Mapping(trc20Address, userMappingList.length - 1);
        return userMappingList.length - 1;
    }
```
mappingTRC20函数有一个参数， 这个参数是主链TRC20合约部署的交易ID,映射这个合约时需要指定这个交易ID的， 结合这个ID和合约部署者地址能够计算出主链TRC20合约地址。另外合约映射是要收取一定的费用，目前收费是1000个TRX。
mappingTRC20函数第一步就是断言用户附带的TRX数量是不是大于等于预先设定的mappingFee（1000trx），如果大于了，将多余的退还给调用者。然后将费用记录到bonus变量中。
第2步通过 txId参数和调用者地址可以计算出主链TRC20地址，然后校验这个合约在主链上是不是真实存在的。这里需要注意，tron中合约地址是通过交易者地址和部署合约的参数生成，所以如果不是当初合约部署者来进行mapping是无法成功的。
第3步是把映射信息加入到userMappingList中，mainToSideContractMap[trc20Address] = 1标记这个trc20Address已经映射过了， 也就说DappChain不允许同一个TRC20合约多次映射。
第4步发送TRC20Mapping事件，这个事件包含两个参数需要映射的TRC20合约地址和映射信息在userMappingList中偏移（nonce），这个nonce是唯一的，也就是说每个映射都有自己独一无二的编号，这个很重要，后面很多操作都会用这个nonce进行判重，防止同一个事件多次处理。

#### Oracle事件中继
接着要考虑的就是如果让侧链知道这个映射，并在侧链上创建一个TRC20合约与之对应。这个就是Oracle的工作，下面是Oracle的架构图：
![image.png-193.2kB][2]

不管是主链GateWay合约还是侧链GateWay合约，他们产生的事件都会被发送kafka服务中， Oracle的processEvent会去消费kafka服务中的消息，事件分为很多种，每一种事件会被封装成不同的eventActuator对象。这里解释一下为什么要不同的事件封装成不同的eventActuator，因为oracle接受到不同事件，要做不同的处理， 所以每一种eventActuator的逻辑是不一样的， 比如接收到TRC20Mapping事件，需要做的事情是产生一个对侧链GateWay合约的multiSignForDeployDAppTRC20AndMapping方法的调用，如果是接收到TRC20Received事件，需要做的事情是对侧链GateWay合约的multiSignForDepositTRC20方法的调用。

processEvent如果接收到一个TRC20Mapping事件会将事件封装成一个MappingTRC20Actuator,MappingTRC20Actuator会将事件转换成一个对侧链GateWay合约的multiSignForDeployDAppTRC20AndMapping方法的调用的交易，然后调用SideChainGatewayAPI将交易发送到侧链，这里要说明一下Oracle的SideChainGatewayApi是封装了对SideChainGway的调用，相当于对SideChainGway的抽象。最后侧链GateWay合约的multiSignForDeployDAppTRC20AndMapping方法会在侧链创建合约并保存映射关系。
下面我们看一下processEvent函数：
代码路径：https://github.com/tronprotocol/sun-network/blob/version/SunNetwork-v1.0.0/dapp-chain/oracle/src/main/java/org/tron/service/task/EventTask.java
```java
//事件处理循环
  //1.从kafka中获取事件
  //2.将事件转换成eventActuator
  //3.判断数据库中是否存在这个事件
  //  3.1第一次接受到这个事件，直接处理
  //  3.2之前已经接收到过,根据当前事件的状态进行不同处理
  public void processEvent() {
    while (true) {
      try {
        //----------------------------------------------
        //1.从kafka中获取事件
        //----------------------------------------------
        ConsumerRecords<String, String> record = this.kfkConsumer.getRecord();
        for (ConsumerRecord<String, String> key : record) {
        //----------------------------------------------
        //2.将事件转换成eventActuator
        //----------------------------------------------
          Actuator eventActuator = EventActuatorFactory.CreateActuator(key.value());
          if (Objects.isNull(eventActuator)) {
            //Unrelated contract or event
            this.kfkConsumer.commit();
            continue;
          }
        //----------------------------------------------
        //3.判断数据库中是否存在这个事件
        //  3.1第一次接受到这个事件，直接处理
        //  3.2之前已经接收到过,根据当前事件的状态进行不同处理
        //----------------------------------------------
          byte[] nonceMsgBytes = NonceStore.getInstance().getData(eventActuator.getNonceKey());
          if (nonceMsgBytes == null) {
            // receive this nonce firstly
            //第一次接收到的处理
            processAndSubmit(eventActuator);
          } else {
            //
            try {
              //从数据库中获取出NonceMsg
              NonceMsg nonceMsg = NonceMsg.parseFrom(nonceMsgBytes);
              String chain = eventActuator.getTaskEnum().name();
              if (nonceMsg.getStatus() == NonceStatus.SUCCESS) {
                //如果事件的状态是成功,输出一个日志
                if (logger.isInfoEnabled()) {
                  String msg = MessageCode.NONCE_HAS_BE_SUCCEED
                      .getMsg(chain, ByteArray.toStr(eventActuator.getNonceKey()));
                  logger.info(msg);
                }
              } else if (nonceMsg.getStatus() == NonceStatus.FAIL) {
                //如果事件的状态是失败的， 重新尝试一次
                setRetryTimesForUserRetry(eventActuator);
                processAndSubmit(eventActuator);
              } else {
                //如果事件的状态是正在处理或者已经广播了，但是没有确定是否成功
                // processing or broadcasted
                if (System.currentTimeMillis() / 1000 >= nonceMsg.getNextProcessTimestamp()) {
                //如果当前时间已经大于了NextProcessTimestamp，重新再试一次
                  setRetryTimesForUserRetry(eventActuator);
                  processAndSubmit(eventActuator);
                } else {
                  if (logger.isInfoEnabled()) {
                    String msg = MessageCode.NONCE_IS_PROCESSING
                        .getMsg(chain, ByteArray.toStr(eventActuator.getNonceKey()));
                    logger.info(msg);
                  }
                }
              }
            } catch (InvalidProtocolBufferException e) {
              logger.error("retry fail: {}", e.getMessage(), e);
            }
          }
          this.kfkConsumer.commit();
        }
      } catch (Exception e) {
        logger.error("in main loop: {}", e.getMessage(), e);
      }
    }
  }
```
processEvent函数是oracle对事件处理的主循环，首先从kafka中循环获取事件， 当获取到一个事件后， 将事件转换成eventActuator，我们看一下eventActuator是如何被创建的：
代码路径：https://github.com/tronprotocol/sun-network/blob/version/SunNetwork-v1.0.0/dapp-chain/oracle/src/main/java/org/tron/service/eventactuator/EventActuatorFactory.java
```java
public class EventActuatorFactory {

  public static Actuator CreateActuator(String eventStr) {
    try {

      JSONObject obj = (JSONObject) JSONValue.parse(eventStr);
      Args args = Args.getInstance();
      if (Objects.isNull(obj) || Objects.isNull(obj.get("contractAddress"))) {
        return null;
      }
      //根据发出事件的合约地址来确定，应该是转换成MainChain相关的Actuator还是SideChain相关的Actuator
      if (obj.get("contractAddress").equals(args.getMainchainGatewayStr())) {
        return createMainChainActuator(obj);
      } else if (obj.get("contractAddress").equals(args.getSidechainGatewayStr())) {
        return createSideChainActuator(obj);
      }
      logger.debug("unknown contract address:{}", obj.get("contractAddress"));
    } catch (Exception e) {
      logger.info("{} create actuator err", eventStr);
      logger.error("{}", e);
      return null;
    }
    return null;
  }
```
从上面的代码可以看出来，主要是根据事件的地址来确定是要创建mainchain相关的actuator还是sidechain相关的actuator。**也就是说如果是主链产生的事件，就创建MainChainActuator，MainChainActuator封装的交易是要发送到侧链。如果是侧链产生的事件，就创建SideChainActuator，SideChainActuator封装的交易是要发送到主链**。
代码路径：https://github.com/tronprotocol/sun-network/blob/version/SunNetwork-v1.0.0/dapp-chain/oracle/src/main/java/org/tron/service/eventactuator/EventActuatorFactory.java
```java
  private static Actuator createMainChainActuator(JSONObject obj) {
    Actuator task;
    MainEventType eventSignature = MainEventType
        .fromSignature(obj.get("eventSignature").toString());
    JSONObject dataMap = (JSONObject) obj.get("dataMap");

    switch (eventSignature) {
      //mainchain gateway合约发出的trx deposit事件
      case TRX_RECEIVED: {
        task = new DepositTRXActuator(dataMap.get("from").toString(),
            dataMap.get("value").toString(), dataMap.get("nonce").toString());
        return task;
      }
      //mainchain gateway合约发出的trc10 deposit事件
      case TRC10_RECEIVED: {
        task = new DepositTRC10Actuator(dataMap.get("from").toString(),
            dataMap.get("tokenId").toString(), dataMap.get("tokenValue").toString(),
            dataMap.get("nonce").toString());
        return task;
      }
      //mainchain gateway合约发出的trc20 deposit事件
      case TRC20_RECEIVED: {
        task = new DepositTRC20Actuator(dataMap.get("from").toString(),
            dataMap.get("contractAddress").toString(), dataMap.get("value").toString(),
            dataMap.get("nonce").toString());
        return task;
      }
      //mainchain gateway合约发出的trc721 deposit事件
      case TRC721_RECEIVED: {
        task = new DepositTRC721Actuator(dataMap.get("from").toString(),
            dataMap.get("contractAddress").toString(), dataMap.get("uid").toString(),
            dataMap.get("nonce").toString());
        return task;
      }
      //mainchain gateway合约发出的 trc20 mapping事件
      case TRC20_MAPPING: {
        task = new MappingTRC20Actuator(dataMap.get("contractAddress").toString(),
            dataMap.get("nonce").toString());
        return task;
      }
      //mainchain gateway合约发出的 trc721 mapping事件
      case TRC721_MAPPING: {
        task = new MappingTRC721Actuator(dataMap.get("contractAddress").toString(),
            dataMap.get("nonce").toString());
        return task;
      }
      default: {
        if (logger.isInfoEnabled()) {
          logger.info("main chain event:{},signature:{}.", obj.get("eventSignature").toString(),
              eventSignature.getSignature());
        }
      }
    }
    return null;
  }
```
createMainChainActuator函数解析事件，根据事件的名称，来创建相应的Actuator。我们这里主要看TRC20_MAPPING事件生成的MappingTRC20Actuator对象。
代码路径：https://github.com/tronprotocol/sun-network/blob/version/SunNetwork-v1.0.0/dapp-chain/oracle/src/main/java/org/tron/service/eventactuator/mainchain/MappingTRC20Actuator.java
```java
@Slf4j(topic = "mainChainTask")
public class MappingTRC20Actuator extends Actuator {

  private static final String NONCE_TAG = "mapping_";

  private MappingTRC20Event event;
  @Getter
  private EventType type = EventType.MAPPING_TRC20;
  //将taskenum设置为TaskEnum.SIDE_CHAIN;
  @Getter
  private TaskEnum taskEnum = TaskEnum.SIDE_CHAIN;

  public MappingTRC20Actuator(String contractAddress, String nonce) {
    ByteString contractAddressBS = ByteString
        .copyFrom(WalletUtil.decodeFromBase58Check(contractAddress));
    ByteString nonceBS = ByteString.copyFrom(ByteArray.fromString(nonce));
    this.event = MappingTRC20Event.newBuilder().setContractAddress(contractAddressBS)
        .setNonce(nonceBS).build();
  }

  public MappingTRC20Actuator(EventMsg eventMsg) throws InvalidProtocolBufferException {
    this.event = eventMsg.getParameter().unpack(MappingTRC20Event.class);
  }

  //创建一个对sidechain的multiSignForDeployDAppTRC20AndMapping函数的调用的交易
  //1.从主网上读取trc20合约的信息
  //2.生成一个对侧链multiSignForDeployDAppTRC20AndMapping函数调用的交易
  @Override
  public CreateRet createTransactionExtensionCapsule() {
    if (Objects.nonNull(transactionExtensionCapsule)) {
      return CreateRet.SUCCESS;
    }
    try {
      String contractAddressStr = WalletUtil
          .encode58Check(event.getContractAddress().toByteArray());
      String nonceStr = event.getNonce().toStringUtf8();

      //----------------------------------------------
      //1.从主网上读取trc20合约的信息
      //----------------------------------------------
      long trcDecimals = MainChainGatewayApi.getTRCDecimals(contractAddressStr);
      String trcName = MainChainGatewayApi.getTRCName(contractAddressStr);
      String trcSymbol = MainChainGatewayApi.getTRCSymbol(contractAddressStr);
      logger.info(
          "MappingTRC20Event, contractAddress: {}, trcName: {}, trcSymbol: {}, trcDecimals: {}, nonce: {}.",
          contractAddressStr, trcName, trcSymbol, trcDecimals, nonceStr);

      //----------------------------------------------
      //2.生成一个对侧链multiSignForDeployDAppTRC20AndMapping函数调用的交易
      //----------------------------------------------
      Transaction tx = SideChainGatewayApi
          .multiSignForMappingTRC20(contractAddressStr, trcName, trcSymbol, trcDecimals, nonceStr);
      this.transactionExtensionCapsule = new TransactionExtensionCapsule(NONCE_TAG + nonceStr, tx,
          0);
      return CreateRet.SUCCESS;
    } catch (Exception e) {
      logger.error("when create transaction extension capsule", e);
      return CreateRet.FAIL;
    }
  }

  @Override
  public EventMsg getMessage() {
    return EventMsg.newBuilder().setParameter(Any.pack(this.event)).setType(getType())
        .setTaskEnum(getTaskEnum()).build();
  }

  @Override
  public byte[] getNonceKey() {
    return ByteArray.fromString(NONCE_TAG + event.getNonce().toStringUtf8());
  }

  @Override
  public byte[] getNonce() {
    return event.getNonce().toByteArray();
  }

```
上面的代码就是MappingTRC20Actuator的实现， 不同的actuator都是继承于Actuator类， 我们比较关心MappingTRC20Actuator实现的下面几个方法：
```java
//将事件转换成对另一条链的调用交易
public abstract CreateRet createTransactionExtensionCapsule();
//将交易广播到另一条链上（父类实现）
public BroadcastRet broadcastTransactionExtensionCapsule() 
//检查广播后的交易是否被确认（父类实现）
public CheckTxRet checkTxInfo() 
```
MappingTRC20Actuator实现的createTransactionExtensionCapsule我们可以看到其实就是通过从事件中获取到主链TRC20的地址，然后根据这个地址从主链上获取trc20的合约的一些属性比如合约名称、合约精度、合约符号，然后根据这些参数创建调用侧链GateWay的multiSignForDeployDAppTRC20AndMapping方法的交易。交易创建的过程不是本地创建，而是调用了侧链节点的GRPC接口创建，是用签名Oracle的私钥进行签名，需要说明的是不管是MainChainGateWayAPI向主链发送交易，还是SideChainGatewayAPI向侧链发送交易，都是使用Oracle的私钥进行签名，我们可以把Orcale看做一个用户在主链和侧链之前交互数据。


#### 侧链新合约创建
在分析侧链新合约创建之前，首先要说一下，目前上线的侧链网络，是由四个oracle参与主侧链交互，比如主链的到侧链的TRC20Mapping事件，会被4个oracle处理，4个oracle都会发送交易调用侧链的multiSignForDeployDAppTRC20AndMapping方法进行新合约创建， 但是并不会真的创建4次， DappChain使用多重签名的方式实现新合约创建，只有2/3*总oracle数量的调用，multiSignForDeployDAppTRC20AndMapping函数才会正真的去创建合约。当然其他的操作（资产抵押、资产赎回等操作）也是如此。


前面我们分析到Oracle从kafka接收到主链MainChainGateWay合约产生TRC20Mapping事件后封装了一个对SideChainGateWay的multiSignForDeployDAppTRC20AndMapping函数的调用的交易， 接下来，我们看看multiSignForDeployDAppTRC20AndMapping的实现细节。
代码路径：https://github.com/tronprotocol/sun-network/blob/version/SunNetwork-v1.0.0/dapp-chain/contract/sideChain/SideChainGateway.sol
```java
// 1. deployDAppTRC20AndMapping
    function multiSignForDeployDAppTRC20AndMapping(address mainChainAddress, string memory name,
        string memory symbol, uint8 decimals, uint256 nonce)
    public goDelegateCall onlyNotStop onlyOracle
    {
        require(mainChainAddress != sunTokenAddress, "mainChainAddress == sunTokenAddress");
        //判断是否完完成多重签名
        bool needMapping = multiSignForMapping(nonce);
        if (needMapping) {
            //如果完成多重签名，部署新合约
            deployDAppTRC20AndMapping(mainChainAddress, name, symbol, decimals, nonce);
        }
    }

    //在侧链上部署新合约
    function deployDAppTRC20AndMapping(address mainChainAddress, string memory name,
        string memory symbol, uint8 decimals, uint256 nonce) internal
    {
        //确保主链的TRC20在侧链没有映射
        require(mainToSideContractMap[mainChainAddress] == address(0), "TRC20 contract is mapped");
        //实例化一个TRC20合约
        address sideChainAddress = address(new DAppTRC20(address(this), name, symbol, decimals));
        //建立主链地址到侧链地址的索引
        mainToSideContractMap[mainChainAddress] = sideChainAddress;
        //建立侧链地址到主链地址的索引
        sideToMainContractMap[sideChainAddress] = mainChainAddress;
        //发送部署完成事件
        emit DeployDAppTRC20AndMapping(mainChainAddress, sideChainAddress, nonce);
        //将主链的trc20合约地址追加到列表中
        mainContractList.push(mainChainAddress);
    }
    //校验多钱是否完成
    //1.判断本次调用的Oracle是否已经完成多签
    //2.记录本次签名
    //3.如果签名的Oracle数量大于2/3，完成多签
    function multiSignForMapping(uint256 nonce) internal returns (bool) {
        //----------------------------------------------
        //1.判断本次调用的Oracle是否已经完成多签
        //----------------------------------------------
        SignMsg storage _signMsg = mappingSigns[nonce];
        if (_signMsg.oracleSigned[msg.sender]) {
            return false;
        }
        //----------------------------------------------
        //2.记录本次签名
        //----------------------------------------------
        _signMsg.oracleSigned[msg.sender] = true;
        _signMsg.signCnt += 1;

        //----------------------------------------------
        //3.如果签名的Oracle数量大于2/3，完成多签
        //----------------------------------------------
        if (!_signMsg.success && _signMsg.signCnt > numOracles * 2 / 3) {
            _signMsg.success = true;
            return true;
        }
        return false;
    }
```
multiSignForDeployDAppTRC20AndMapping函数的参数第一个参数是主链TRC20合约的地址， 最后是事件的编号，前面我们分析这个编号是唯一的，不会产生重复。multiSignForDeployDAppTRC20AndMapping一进来就调用multiSignForMapping函数来进行校验多重签名是否完成， mappingSigns字典保存的不同nonce事件的签名情况，如果本次调用的oracle已经签过了， 直接返回false，如果没有签过，则记录本次签名，然后判断签名总数是不是大于2/3个orace总数，大于返回true，小于返回false。需要说明的是侧链的Gateway合约是如何知道总共有多少个oracle呢？是因为oracle合约提供的增加和删除oracle合约的接口，addOracle(address _oracle) 和 delOracle(address _oracle)方法，也就是说侧链gateway合约部署后，需要将所有oracle合约的地址全部更新到侧链的gateway合约中。

我们再回到multiSignForDeployDAppTRC20AndMapping方法中，如果multiSignForMapping方法没有返回true，也就是没有完成多签，则直接返回，不会进行新合约创建。如果multiSignForMapping方法返回true，则调用deployDAppTRC20AndMapping函数进行新合约部署。

deployDAppTRC20AndMapping函数很简单，就是根据oracle传递过来的参数进行新合约的创建，直接使用一个预定义好的TRC20模版类进行实例化。将新创建的合约地址和主链TRC20的地址进行相互关联，也即是分别建立两个字典，通过主链合约地址能够立刻索引到新合约地址，通过新合约地址也能立刻索引到主链合约地址。最后产生一个DeployDAppTRC20AndMapping事件，需要说明一下，DeployDAppTRC20AndMapping事件也会被发送到kafka，但是oracle并不会对这个事件做任何处理。

至次，就完成了主链的合约到侧链的映射，大家发现这个新创建的合约是空的，没有发行量也没有任何balance记录。后面我们要分析的deposit操作，将主链的资产质押到侧链，主要原理就是将主链普通用户在TRC20的余额转给主链的MainChainGateWay合约，发出相应的事件，然后通过oracle事件中继（过程跟本文描述的mapping事件中继类似），在侧链的新创建的合约中增加相应数量的余额。相当于用户失去了主链的TRC20的资产控制权， 但是获得了侧链资产的控制权。然后用户就可以在侧链使用自己的资产，最后用户可以将剩余的余额重新赎回到主链，只要调用侧链的GateWay合约进行资产提取，销毁侧链TRC20的资产，再通过oracle事件中继，使得主链的Gateway合约将剩余的质押余额返回给用户。

  [1]: http://static.zybuluo.com/zyumingfit/sf3m7kukzeo4an2er2iq81cy/image.png
  [2]: http://static.zybuluo.com/zyumingfit/lcht5xaeiedcbvovq0zg7sec/image.png
