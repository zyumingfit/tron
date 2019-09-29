上一篇文章中，我们分析了DappChain是如何将一个主链TRC20资产质押到侧链,这篇文章主要介绍用户如何将侧链的资产从侧链转移到主链，以及DappChain底层是如何工作的。
下图是TRC20提款的操作，dappchain的整个实现流程：
![withdraw.png-5098.6kB][1]

 1. 用户调用sunweb.withdrawTrc20函数发起提款操作，这个函数会调用侧链TRC20合约的withdraw函数，这个函数会将资产transfer给SideChainGateway合约地址，然后调用SideChainGateway的onTRC20Received方法。
 2. onTRC20Received方法会将发送给自己的TRC20资产销毁，然后触发WithdrawTrc20事件，
    这个事件被oracle监听到并处理。
 3. 当oracle监听到WithdrawTrc20事件后，将提款的相关数据合并到一起（from,
        mainChainAddress, value,
        nonce）做一个签名，并将签名传递给SideChainGateWay合约的multiSignForWithdrawTRC20方法。
 4. 第3步会被4个oracle处理，每个oracle都会调用SideChainGateWay合约的multiSignForWithdrawTRC20方法，每一次调用multiSignForWithdrawTRC20方法中都会判断总数是否满足签名总数大于等于2/3的总oracle数，如果满足就进行多重签名验证，验证通过后触发一个MultiSignForWithdrawTRC20事件
 5. oracle监听到MultiSignForWithdrawTRC20事件后会调用SideChainGateWay的getWithdrawSigns方法将所有签名拿回来，然后传递给MainChainGateWay合约的withdrawTRC20函数
 6. withdrawTRC20函数也会验证多重签名的正确性，如果验证通过会将赎回的资产转给提款人。

上面的流程可能有点绕， 下面是每一个步骤的实现细节。


## sdk资产赎回函数
用户可以调用sunnetwork sdk将侧链的TRC20资产赎回到主链， 下面是一个调用sdk赎回资产的例子：
```javascript
//从侧链的contractAddress合约赎回num个最小单位的token到主链的trc20合约中，withdrawFee是本次赎回的费用, feelimit是本次调用允许最多消耗的费用
//sunWeb.withdrawTrc20(num, withdrawFee, feeLimit, contractAddress, options);
sunWeb.withdrawTrc20(10000, 10000, 10000000, 'TWzXQmDoASGodMss7uPD6vUgLHnkQFX7ok');
```
sdk的withdrawTrc20函数实际上是调用了侧链映射的trc20合约的withdrawal函数，withdrawal函数主要的作用就是将用户在侧链trc20合约的资产转给侧链的gateway合约，然后调用侧链gateway合约的相关函数进行资产赎回。下面是侧链trc20合约的withdrawal函数源码：
```java
//侧链trc20合约提款
    //1.如果赎回附带的费用大于预定义的费用， 将多余费用退回
    //2.将侧链trc20合约的资产转交给侧链gateway合约
    //3.调用侧脸的gateway合约，进行资产赎回
    function withdrawal(uint256 value) payable external returns (uint256 r) {
        //从侧链Gateway合约获取赎回费用信息
        uint256 withdrawFee = ITRC20Receiver(gateway).getWithdrawFee();
        //----------------------------------------------
        //1.如果赎回附带的费用大于预定义的费用， 将多余费用退回
        //----------------------------------------------
        require(msg.value >= withdrawFee, "value must be >= withdrawFee");
        if (msg.value > withdrawFee) {
            msg.sender.transfer(msg.value - withdrawFee);
        }
        //----------------------------------------------
        //2.将侧链trc20合约的资产转交给侧链gateway合约
        //----------------------------------------------
        transfer(gateway, value);
        //----------------------------------------------
        //3.调用侧脸的gateway合约，进行资产赎回
        //----------------------------------------------
        r = ITRC20Receiver(gateway).onTRC20Received.value(withdrawFee)(msg.sender, value);
    }
```
withdrawal首先是判断是调用侧链的Gateway合约获取赎回资产的费用，然后看用户的本次调用附带的费用是不是足够，并且将多余的费用返还给用户。第2步就是将用户的资产转移给gateway， 相当于将资产的控制权转交给gateway合约。最后调用侧链Gateway合约的onTRC20Received函数将资产赎回到主链。

需要说明的是用户需要确定自己赎回的额度要小于等于侧链的余额， 如果大于了会导致transfer函数执行失败，导致整个调用被回滚。也就是说如果用户从主链质押100个trc20token到侧链， 然后在侧链进行了一系列的转账，最终侧链的余额只有50个trc20token， 当用户调用sunweb.withdrawTrc20赎回资产的时候， 指定的赎回量只能是小于等于50，超过50会失败。


## SideChainGateway资产资产销毁
侧链的onTRC20Received函数实现的主要功能是将侧链的资产销毁，触发事件通知oracle资产赎回情况， 以便于oracle能够调用主链的GateWay合约进行资产归还。
```java
// 8. withdrawTRC20
    function onTRC20Received(address from, uint256 value) payable
    public goDelegateCall onlyNotPause onlyNotStop returns (uint256 r) {
        //1.根据侧链中保存的映射关系获取到主链的trc20合约地址,并判断地址的有效性
        address mainChainAddress = sideToMainContractMap[msg.sender];
        require(mainChainAddress != address(0), "mainChainAddress == address(0)");
        require(value >= withdrawMinTrc20, "value must be >= withdrawMinTrc20");
        if (msg.value > 0) {
            bonus += msg.value;
        }
        //2.将提款信息保存到userWithdrawList
        userWithdrawList.push(WithdrawMsg(from, mainChainAddress, 0, value, DataModel.TokenKind.TRC20, DataModel.Status.SUCCESS));

        // burn
        //3.将侧链trc20的相应资产销毁
        DAppTRC20(msg.sender).burn(value);
        //4.触发提款事件，通知oracle
        emit WithdrawTRC20(from, mainChainAddress, value, userWithdrawList.length - 1);
        r = userWithdrawList.length - 1;
    }
```
SideChainGateway的withdrawTRC20首先使用msg.sender从 sideToMainContractMap映射表中取出主链trc20的地址，msg.sender是侧链trc20的地址，sideToMainContractMap保存的是从侧链地址到主链地址的字典。第2步就是将信息保存到userWithdrawList列表中，然后调用侧链trc20合约的burn函数销毁资产，因为前面已经将资产所有权转移给了SideChainGateway，所有SideChainGateway完全有权限去销毁资产。最后发送WithdrawTRC20事件，oracle会监听这个事件，相当于告诉oracle， from从mainChainAddress对应的侧链合约中赎回value这么多的资产到主链，另外还传递了赎回事件的nonce值， 防止重复赎回。

## Oracle  WithdrawTRC20事件中继
Oracle对WithdrawTRC20事件的处理和其他的事件处理类似， 首先是processEvent从kafka中拿到事件，然后通过EventActuatorFactory的createSideChainActuator函数创建一个WithdrawTRC20Actuator， WithdrawTRC20Actuator类实现的 createTransactionExtensionCapsule方法是生成一个对SideChainGateWay合约的multiSignForWithdrawTRC20函数调用的交易，multiSignForWithdrawTRC20函数的任务是对多个oracle的赎回多签验证，如果验证成功，就会发送事件给oracle，让oracle去调用MainChainGateWay合约，进行资产赎回操作。
```java
//WithdrawTRC20Actuator.createTransactionExtensionCapsule方法
 public CreateRet createTransactionExtensionCapsule() {
    if (Objects.nonNull(transactionExtensionCapsule)) {
      return CreateRet.SUCCESS;
    }

    try {
      String fromStr = WalletUtil.encode58Check(event.getFrom().toByteArray());
      String mainChainAddressStr = WalletUtil
          .encode58Check(event.getMainchainAddress().toByteArray());
      String valueStr = event.getValue().toStringUtf8();
      String nonceStr = event.getNonce().toStringUtf8();

      logger
          .info("WithdrawTRC20Actuator, from: {}, mainChainAddress: {}, value: {}, nonce: {}",
              fromStr, mainChainAddressStr, valueStr, nonceStr);
      // 生成一个对SideChainGateway的 multiSignForWithdrawTRC20函数的调用
      Transaction tx = SideChainGatewayApi
          .withdrawTRC20Transaction(fromStr, mainChainAddressStr, valueStr, nonceStr);
      this.transactionExtensionCapsule = new TransactionExtensionCapsule(PREFIX + nonceStr, tx, 0);
      return CreateRet.SUCCESS;
    } catch (Exception e) {
      logger.error("when create transaction extension capsule", e);
      return CreateRet.FAIL;
    }
  }
```
SideChainGatewayApi.withdrawTRC20Transaction函数会对赎回操作的的数据进行签名，把签名后的结果传递给SideChainGateWay合约的multiSignForWithdrawTRC20方法进行验签。

```
SideChainGatewayApi.withdrawTRC20Transaction方法
//对赎回数据的签名，并调用SideChainGateway进行验签
  //1.对赎回数据的签名
  //2.调用SideChainGageway的multiSignForWithdrawTRC20函数进行验签
  public static Transaction withdrawTRC20Transaction(String from, String mainChainAddress,
      String value, String nonce) throws RpcConnectException {

    //----------------------------------------------
    //1.对赎回数据的签名
    //----------------------------------------------
    String ownSign = getWithdrawTRCTokenSign(from, mainChainAddress, value, nonce);

    byte[] contractAddress = Args.getInstance().getSidechainGateway();
    String method = "multiSignForWithdrawTRC20(uint256,bytes)";
    List params = Arrays.asList(nonce, ownSign);
    //----------------------------------------------
    //2.调用SideChainGageway的multiSignForWithdrawTRC20函数进行验签
    //----------------------------------------------
    return GATEWAY_API.getInstance()
        .triggerContractTransaction(contractAddress, method, params, 0, 0, 0);
  }
```


## SideChainGateway多签确认实现
```java
    function multiSignForWithdrawTRC20(uint256 nonce, bytes memory oracleSign)
    public goDelegateCall onlyNotStop onlyOracle {
        //----------------------------------------------
        //1.判断多签是否完成
        //----------------------------------------------
        if (!countMultiSignForWithdraw(nonce, oracleSign)) {
            return;
        }

        //----------------------------------------------
        //2.多签验证
        //----------------------------------------------
        WithdrawMsg storage withdrawMsg = userWithdrawList[nonce];
        bytes32 dataHash = keccak256(abi.encodePacked(withdrawMsg.user, withdrawMsg.mainChainAddress, withdrawMsg.valueOrUid, nonce));
        if (countSuccessSignForWithdraw(nonce, dataHash)) {
            emit MultiSignForWithdrawTRC20(withdrawMsg.user, withdrawMsg.mainChainAddress, withdrawMsg.valueOrUid, nonce);
        }
    }
    
        //保存签名，并确定是否满足多重签名的个数
    //1.保存签名
    //2.如果签名总个数大于2/3的总oracle数，返回true
    function countMultiSignForWithdraw(uint256 nonce, bytes memory oracleSign) internal returns (bool){

        //----------------------------------------------
        //1.保存签名
        //----------------------------------------------
        SignMsg storage _signMsg = withdrawSigns[nonce];
        if (_signMsg.oracleSigned[msg.sender]) {
            return false;
        }
        _signMsg.oracleSigned[msg.sender] = true;
        _signMsg.signs.push(oracleSign);
        _signMsg.signOracles.push(msg.sender);
        _signMsg.signCnt += 1;
        //2.如果签名总个数大于2/3的总oracle数，返回true
        if (!_signMsg.success && _signMsg.signCnt > numOracles * 2 / 3) {
            return true;
        }
        return false;
    }
    
        //多重签名验证
    //1.获取出签名信息
    //2.多重签名验证
    function countSuccessSignForWithdraw(uint256 nonce, bytes32 dataHash) internal returns (bool) {
        //----------------------------------------------
        //1.获取出签名信息
        //----------------------------------------------
        SignMsg storage _signMsg = withdrawSigns[nonce];
        if (_signMsg.success) {
            return false;
        }
        //----------------------------------------------
        //2.多重签名验证
        //----------------------------------------------
        bytes32 ret = multivalidatesign(dataHash, _signMsg.signs, _signMsg.signOracles);
        uint256 count = countSuccess(ret);
        if (count > numOracles * 2 / 3) {
            _signMsg.success = true;
            return true;
        }
        return false;
    }
```
multiSignForWithdrawTRC20函数首先调用countMultiSignForWithdraw确认签名总数是否满足要求， countMultiSignForWithdraw首先将签名保存下来，然后看总数是否大于等于2/3个oracle总数， 如果满足则返回true。也就是当第2个oracle调用countMultiSignForWithdraw的时候即可满足要求。

如果验证签名总数满足要求，就走到第2步，多重签名验证。这里是调用multivaliatesign函数进行多重签名验证， 这个函数是波场在原来solidity基础上增加一个新内置函数，专门进行多重签名校验。这个函数第1个参数是签名内容的hash， 第2个参数是签名列表，第3个参数是签名的oracle地址列表，返回值一个bytes32的类型， 每一位代表列表中的一个oracle是否签名正确，1表示验签正确， 0表示验签失败。

接着通过统计multivaliatesign的返回值中总的验证成功个数，判断这个个数是否大于等于2/3个oracle数，大于则多重签名成功，否则失败。

我们再回到multiSignForWithdrawTRC20函数， 如果countSuccessSignForWithdraw返回true，就会触发MultiSignForWithdrawTRC20事件通知oracle多重签名验证成功，以便以oracle可以通知MainChainGateWay可以将提款额转给提款人。

MultiSignForWithdrawTRC20事件的参数包括提款人地址、 主链TRC20合约地址、提款额、提款nonce，相当于告诉oracle哪个人在哪个trc20合约中提了多少钱到主链。

## Oracle  MultiSignForWithdrawTRC20事件中继

oracle接收到MultiSignForWithdrawTRC20事件后通过EventActuatorFactory的createSideChainActuator函数创建一个MultiSignForWithdrawTRC20Actuator对象。MultiSignForWithdrawTRC20Actuator的createTransactionExtensionCapsule方法是生成一个对MainChainGateWay合约的withdrawTRC20函数调用的交易。
```java
 public CreateRet createTransactionExtensionCapsule() {
    if (Objects.nonNull(transactionExtensionCapsule)) {
      return CreateRet.SUCCESS;
    }
    try {
      // ----------------------------------------------
      //1.从事件中解析出提款人
      // ----------------------------------------------
      String fromStr = WalletUtil.encode58Check(event.getFrom().toByteArray());
      // ----------------------------------------------
      //2.从事件中解析出主网trc20的地址
      // ----------------------------------------------
      String mainChainAddressStr = WalletUtil
          .encode58Check(event.getMainchainAddress().toByteArray());

      // ----------------------------------------------
      //3.从事件中解析出赎回额度
      // ----------------------------------------------
      String valueStr = event.getValue().toStringUtf8();
      //4.从事件中解析出nonce
      String nonceStr = event.getNonce().toStringUtf8();
      // ----------------------------------------------
      //5.从SidechainGateway合约中获取出所有签名数据
      // ----------------------------------------------
      SignListParam signParam = SideChainGatewayApi.getWithdrawOracleSigns(nonceStr);
      logger.info(
          "MultiSignForWithdrawTRC20Actuator, from: {}, mainChainAddress: {}, value: {}, nonce: {}",
          fromStr, mainChainAddressStr, valueStr, nonceStr);
      // ----------------------------------------------
      //6.创建一个调用MainChainGateway的withdrawTRC20交易
      // ----------------------------------------------
      Transaction tx = MainChainGatewayApi
          .multiSignForWithdrawTRC20Transaction(fromStr, mainChainAddressStr, valueStr, nonceStr,
              signParam);

      this.transactionExtensionCapsule = new TransactionExtensionCapsule(PREFIX + nonceStr, tx,
          getDelay(fromStr, mainChainAddressStr, valueStr, nonceStr, signParam.getOracleSigns()));
      return CreateRet.SUCCESS;
    } catch (Exception e) {
      logger.error("when create transaction extension capsule", e);
      return CreateRet.FAIL;
    }
  }
```
SideChainGatewayApi.getWithdrawOracleSigns函数
```
    public static SignListParam getWithdrawOracleSigns(String nonce) throws RpcConnectException {
    byte[] contractAddress = Args.getInstance().getSidechainGateway();
    String method = "getWithdrawSigns(uint256)";
    List params = Arrays.asList(nonce);
    byte[] ret = GATEWAY_API.getInstance()
        .triggerConstantContractAndReturn(contractAddress, method, params, 0, 0, 0);
    return AbiUtil.unpackSignListParam(ret);
  }
```
SideChainGateWay合约的getWithdrawOracleSigns方法:
```java
//SideChainGateWay的getWithdrawOracleSigns方法
 function getWithdrawSigns(uint256 nonce) view public returns (bytes[] memory, address[] memory) {
        return (withdrawSigns[nonce].signs, withdrawSigns[nonce].signOracles);
    }
```

MainChainGatewayApi.multiSignForWithdrawTRC20Transaction方法：
```
public static Transaction multiSignForWithdrawTRC20Transaction(String from,
      String mainChainAddress, String value, String nonce, SignListParam signParam)
      throws RpcConnectException {

    byte[] contractAddress = Args.getInstance().getMainchainGateway();
    String method = "withdrawTRC20(address,address,uint256,uint256,bytes[],address[])";
    //将被签名内容和签名后的数据组成一个参数列表
    List params = Arrays.asList(from, mainChainAddress, value, nonce, signParam.getOracleSigns(),
        signParam.getOracleAddresses());
    //调用主链Gateway合约的withdrawTRC20方法
    return GATEWAY_API.getInstance()
        .triggerContractTransaction(contractAddress, method, params, 0, 0, 0);

  }
```
上面代码的前4步就是从事件中获取出提款的所有信息，第5步是调用SideChainGateWay的getWithdrawOracleSigns方法将签名列表和oracle地址列表获取到，然后调用MainChainGateway的withdrawTRC20函数。

通过之前的分析我们知道， oracle实际上就是对前4步中的提款信息进行的签名，所以这里拿到提款相关的信息、签名数据列表、签名地址列表这3部分之后，就可以对这些签名数据列表进行多重签名验签。所以MainChainGateway的withdrawTRC20函数就是进行这个验签以确定赎回提款的准确无误。

## MainChainGateway资产归还
 MainChainGateway的withdrawTRC20函数就是对多个oracle对赎回数据的确认签名进行多重签名验签，如果验签成功，说明2/3个oracle对赎回操作都进行了确认，可以将用户的提款额归还给用户了。
```java
//trc20资产赎回
    //1.多重签名验签
    //2.如果验签成功，将资产transfer给提款人
    function withdrawTRC20(address _to, address contractAddress, uint256 value,
        uint256 nonce, bytes[] memory oracleSigns, address[] memory signOracles)
    public goDelegateCall onlyNotStop onlyOracle
    {
        require(oracleSigns.length <= numOracles, "withdraw TRC20 signs num > oracles num");
        //-------------------------
        //1.获取被签名的原始数据的hash值
        //-------------------------
        bytes32 dataHash = keccak256(abi.encodePacked(_to, contractAddress, value, nonce));
        //-------------------------
        //2.防止重复处理
        //-------------------------
        if (withdrawMultiSignList[nonce][dataHash].success) {
            return;
        }
        //-------------------------
        //3.多签验证
        //-------------------------
        bool needWithdraw = checkOracles(dataHash, nonce, oracleSigns, signOracles);
        //-------------------------
        //4.如果验签成功，将资产transfer给提款人
        //-------------------------
        if (needWithdraw) {
            TRC20(contractAddress).transfer(_to, value);
            emit TRC20Withdraw(_to, contractAddress, value, nonce);
        }
    }

```
withdrawTRC20的第1步就是将原始被签名的数据打包并做hash，用作验签，第2步很重要，我们前面说4个oracle每一个oracle都会对同一个事件做处理，所以MultiSignForWithdrawTRC20事件也会被多次处理， 也就是说withdrawTRC20会被多次调用，但是只能执行一次， 如果执行多次会就会发生重复提款。所以这里的意思就是如果已经有一次多签确认了，就不需要再处理了。

如果第一次调用就能走到第3步进行多签的验证了，这里是调用checkOracle函数进行的。
```java
    function checkOracles(bytes32 dataHash, uint256 nonce, bytes[] memory sigList, address[] memory signOracles) internal returns (bool) {
        require(sigList.length == signOracles.length, "error sigList.length or signOracles.length");
        //如果multivalidatesignSwitch开关打开了，就使用multivalidatesign内置函数进行多重签名验证
        if (multivalidatesignSwitch) {
            uint256 signFlag = 0;
            for (uint256 i = 0; i < sigList.length; i++) {
                uint256 _oracleIndex = oracleIndex[signOracles[i]];
                if (_oracleIndex == 0) {
                    signOracles[i] = address(0);
                    continue;
                }
                uint256 thisFlag = (1 << (_oracleIndex - 1));
                if ((thisFlag & signFlag) == 0) {
                    // not signed
                    signFlag = thisFlag | signFlag;
                } else {
                    signOracles[i] = address(0);
                }
            }
            return checkOraclesWithMultiValidate(dataHash, nonce, sigList, signOracles);
        }
        //如果multivalidatesignSwitch关闭了，就是用ecrecover进行逐个验签

        SignMsg storage signMsg = withdrawMultiSignList[nonce][dataHash];
        uint256 _signedOracleFlag = signMsg.signedOracleFlag;
        uint64 _countSign = signMsg.countSign;

        for (uint256 i = 0; i < sigList.length; i++) {
            address _oracle = dataHash.recover(sigList[i]);
            uint256 _oracleIndex = oracleIndex[_oracle];
            if (_oracleIndex == 0) {// not oracle
                continue;
            }
            uint256 thisFlag = (1 << (_oracleIndex - 1));
            if ((thisFlag & _signedOracleFlag) == 0) {
                // not signed
                _signedOracleFlag = thisFlag | _signedOracleFlag;
                _countSign++;
            }
        }
        signMsg.signedOracleFlag = _signedOracleFlag;
        signMsg.countSign = _countSign;

        if (!signMsg.success && signMsg.countSign > numOracles * 2 / 3) {
            signMsg.success = true;
            return true;
        }
        return false;
    }
```
这里根据multivalidatesignSwitch开关来决定使用什么方式多重签名验签是有点历史原因的，在DappChain发布时， 波场的主链还不支持multivalidatesign内置函数， 所以使用了一个开关来手动控制， 当主网支持multivalidatesign之后， MainChainGateway的部署者就可以打开这个开关了。multivalidatesign函数比原始的ecrecover的费用低很多。

我们再回到withdrawTRC20函数，如果checkOracles验签成功之后，就将调用主链TRC20合约的transfer方法将相应的资产归还给提款人， 相当于在主链的TRC20合约中， 用户又重新拿回了资产，另外在侧链中TRC20合约中的资产已经被销毁不存在了。

至此TRC20合约的赎回操作就全部完成了， 通过我们对TRC20合约映射、质押资产、赎回的分析，相信大家可以更容易的理解TRC10、TRC721、TRX的流程了。

  [1]: http://static.zybuluo.com/zyumingfit/g5qveff83vc5qw1lmfxq0g3r/withdraw.png
