
上一篇文章中，我们分析了DappChain是如何将一个主链TRC20合约映射到侧链,这篇文章主要介绍用户如何将主链的资产从主链转移到侧链，以及DappChain底层是如何工作的。

### 资产质押
如果TRC20的部署者已经将合约映射到主网， 用户可以调用sunnetwork sdk将主网的TRC20资产质押到侧链， 下面是一个调用sdk质押主链资产到侧链的例子：
```javascript
/*第一步:授权资产给oracle*/
//授权1000000个最小单位的token给MainChainGateWay合约，本次授权最多消耗10000000sun用于资源消耗，contractAddress是trc20合约地址
//sunWeb.approveTrc20(num, feeLimit, contractAddress, options);
sunWeb.approveTrc20(1000000, 10000000, 'TGKuXDnvdHv9RNE6BPXUtNLK2FrVMBDAuA');  

/*第二步:质押资产到侧链*/
//质押1000000个最小单位的token到侧链，附带100000000sun（100trx）的质押费用，本次质押最多消耗1000000用于资源消耗
//sunWeb.depositTrc20(callValue, depositFee, feeLimit, options);
sunWeb.depositTrc20(1000000, 10000000, 1000000);
```
上面的第一步是将用户的trc20资产授权给MainChainGateWay合约，第二步是调用MainChainGateWay的depositTRC20方法，因为MainChainGateWay合约已经获得了用户的trc20使用权， 所以在depositTRC20方法会将用户的TRC20资产转移到给定的数量到MainChainGateWay合约owner地址上，需要说明的是depositTRC20的callValue参数指定的数量必须小于approveTrc20的num参数，也就是说质押的额度不能大于授权的额度。

我们看一下MainChainGateWay合约的depositTRC20代码：
```java
 //质押一定数量的token到侧链
    //1.收取映射费用， 如果附带的费用过多， 则退换还
    //2.从trc20合约中将资金转到mainchain-getway
    //3.发送TRC20Received事件
    function depositTRC20(address contractAddress, uint64 value) payable
    public goDelegateCall onlyNotStop onlyNotPause isHuman returns (uint256) {
        //----------------------------------------------
        //1.收取质押费用， 如果附带的费用过多， 则退换还
        //----------------------------------------------
        require(mainToSideContractMap[contractAddress] == 1, "not an allowed token");
        require(value >= depositMinTrc20, "value must be >= depositMinTrc20");
        require(msg.value >= depositFee, "msg.value need  >= depositFee");
        //退还多余的质押费用
        if (msg.value > depositFee) {
            msg.sender.transfer(msg.value - depositFee);
        }
        bonus += depositFee;

        //----------------------------------------------
        //2.从trc20合约中将资金转到mainchain-getway
        //----------------------------------------------
        if (!TRC20(contractAddress).transferFrom(msg.sender, address(this), value)) {
            revert("TRC20 transferFrom error");
        }
        userDepositList.push(DepositMsg(msg.sender, value, 2, contractAddress, 0, 0, 0));
        //----------------------------------------------
        //3.发送TRC20Received事件
        //----------------------------------------------
        emit TRC20Received(msg.sender, contractAddress, value, userDepositList.length - 1);
        return userDepositList.length - 1;

    }
```
首先depositTRC20函数使用require断言这个TRC2O合约是否已经被映射，没有被映射的合约，用户对其中的资产是不能够质押的。然后判断用户附带的手续费是否则足够， 目前质押资产需要100TRX的手续费，如果用户附带的转账额大于100TRX，会将多余的转账额退还给用户。

接着就是MainChainGateWay合约将用户的质押额转账到自己地址，由于前面用户已经调用了sunWeb.approveTrc20完成了资产授权，所以MainChainGateWay合约有权限对用户的资产进行转账操作。然后就是将deposit信息保存到userDepositList列表中。

最后发送TRC20Received事件，oracle模块会监听这个事件，TRC20Received事件包含的参数有发起Depoist操作的用户地址、TRC20地址、质押额度、TRC20Received的nonce值，相当于告诉了oracle，哪个用户质押了哪个TRC20合约多少资产，以及depoist事件的唯一编号nonce。

### Oracle TRC20Received事件中继
Oracle对TRC20Received事件的处理和TRC20_MAPPING一样， 首先是processEvent从kafka中拿到事件，然后通过EventActuatorFactory创建一个Actuator，和TRC20_MAPPING不一样的是， 这里创建的是DepositTRC20Actuator对象,DepositTRC20Actuator类实现的 createTransactionExtensionCapsule方法是生成一个对SideChainGateWay合约的multiSignForDepositTRC20函数调用的交易，multiSignForDepositTRC20实际上就是在侧链的TRC20合约中增发相应数量的资产。
```java
@Slf4j(topic = "mainChainTask")
public class DepositTRC20Actuator extends Actuator {

  private static final String NONCE_TAG = "deposit_";

  private DepositTRC20Event event;
  @Getter
  private EventType type = EventType.DEPOSIT_TRC20_EVENT;
  @Getter
  private TaskEnum taskEnum = TaskEnum.SIDE_CHAIN;

  //生成交易
  @Override
  public CreateRet createTransactionExtensionCapsule() {
    if (Objects.nonNull(transactionExtensionCapsule)) {
      return CreateRet.SUCCESS;
    }
    try {
      String fromStr = WalletUtil.encode58Check(event.getFrom().toByteArray());
      String contractAddressStr = WalletUtil
          .encode58Check(event.getContractAddress().toByteArray());
      String valueStr = event.getValue().toStringUtf8();
      String nonceStr = event.getNonce().toStringUtf8();

      logger.info("DepositTRC20Actuator, from: {}, value: {}, contractAddress: {}, nonce: {}",
          fromStr, valueStr, contractAddressStr, nonceStr);

      //创建对侧链调用的交易
      Transaction tx = SideChainGatewayApi
          .mintToken20Transaction(fromStr, contractAddressStr, valueStr, nonceStr);
      this.transactionExtensionCapsule = new TransactionExtensionCapsule(NONCE_TAG + nonceStr, tx,
          0);
      return CreateRet.SUCCESS;
    } catch (Exception e) {
      logger.error("when create transaction extension capsule", e);
      return CreateRet.FAIL;
    }
  }
  
  
    public static Transaction mintToken20Transaction(String to, String mainAddress, String value,
      String nonce) throws RpcConnectException {
    byte[] contractAddress = Args.getInstance().getSidechainGateway();

    //请求侧链节点生成调用multiSignForDepositTRC20函数的交易
    String method = "multiSignForDepositTRC20(address,address,uint256,uint256)";
    List params = Arrays.asList(to, mainAddress, value, nonce);
    return GATEWAY_API.getInstance()
        .triggerContractTransaction(contractAddress, method, params, 0, 0, 0);
  }
```


### 侧链TRC20合约增发
```java
 //校验depoist多签是否完成
    //1.判断本次调用的Oracle是否已经完成多签
    //2.记录本次签名
    //3.如果签名的Oracle数量大于2/3，完成多签
    function multiSignForDeposit(uint256 nonce) internal returns (bool) {
        //----------------------------------------------
        //1.判断本次调用的Oracle是否已经完成多签
        //----------------------------------------------
        SignMsg storage _signMsg = depositSigns[nonce];
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
    
    
   // 4. depositTRC20
    function multiSignForDepositTRC20(address to, address mainChainAddress,
        uint256 value, uint256 nonce)
    public goDelegateCall onlyNotStop onlyOracle
    {
        address sideChainAddress = mainToSideContractMap[mainChainAddress];
        require(sideChainAddress != address(0), "the main chain address hasn't mapped");
        //校验多签是否完成
        bool needDeposit = multiSignForDeposit(nonce);
        if (needDeposit) {
            //质押资产
            depositTRC20(to, sideChainAddress, value, nonce);
        }
    }
```
multiSignForDepositTRC20的第1个参数表示的是发起质押的用户的地址，第2个是主链TRC20合约的地址，第3个是质押额， 第4个是质押事件的的编号。首先通过mainToSideContractMap字典找到侧链地址， 然后调用multiSignForDeposit确定多个Oracle是否完成多签确定，如果确定了就调用 depositTRC20进行质押操作。

```java
    function depositTRC20(address to, address sideChainAddress, uint256 value, uint256 nonce) internal {
        IDApp(sideChainAddress).mint(to, value);
        emit DepositTRC20(to, sideChainAddress, value, nonce);
    }
```
DepositTRC20事件相对比较简单就是调用了侧链的TRC20合约的mint接口进行增发，增发给to地址， 额度是value，最后发送DepositTRC20事件，当然侧链发出的DepositTRC20事件， oracle并不会去处理。

到这里，就完成了主链TRC20合约的资产质押到侧链， 用户可以将资产在侧链上进行了流通了， 下一篇文章，我们介绍资产是如何从侧链赎回到主链的。

