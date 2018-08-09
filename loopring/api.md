### 参考文档
官方文档地址 `https://loopring.github.io/relay-cluster/relay_api_spec_v2.html#loopring_notifytransactionsubmitted`。
### 接口
`loopring_notifyTransactionSubmitted`
### 参数含义
- hash - The txHash.
- nonce - The owner newest nonce.
- to - The target address to send.
- value - The value in transaction.
- gasPrice.
- gas.
- input - The value input in transaction.
- from - The transaction sender.
- v - ECDSA signature parameter v.
- r - ECDSA signature parameter r.
- s - ECDSA signature parameter s.

其中，`input`, `v`, `r`, `s`含义如下：

#### `input`

`input`是交易时的`data`（详见 https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethsendtransaction)。
普通交易时`data`是附带的二进制信息，创建合约时`data`为合约代码。

#### `v`、 `r`、 `s`

##### 组装待签名消息
**因loopring_notifyTransactionSubmitted接口未实现，该部分选用loopring_submitOrder作为示例，接口文档见 https://loopring.github.io/relay-cluster/relay_api_spec_v2.html#loopring_submitorder**

- 把订单信息打平组装成Hash，用16进制数据依次拼接，顺序依次为
  - `delegate`
  - `address`
  - `tokenS`
  - `tokenB`
  - `walletAddress`
  - `authAddr`
  - `amountS`
  - `amountB`
  - `validSince`
  - `validUntil`
  - `lrcFee`
  - `buyNoMoreThanAmountB`
  - `marginSplitPercentage`

```swift
    func getOrderHash(order: OriginalOrder) -> Data {
        var result: Data = Data()
        result.append(contentsOf: order.delegate.hexBytes)
        result.append(contentsOf: order.address.hexBytes)
        let tokens = TokenDataManager.shared.getAddress(by: order.tokenSell)!
        result.append(contentsOf: tokens.hexBytes)
        let tokenb = TokenDataManager.shared.getAddress(by: order.tokenBuy)!
        result.append(contentsOf: tokenb.hexBytes)
        result.append(contentsOf: order.walletAddress.hexBytes)
        result.append(contentsOf: order.authAddr.hexBytes)
        result.append(contentsOf: _encode(order.amountSell, order.tokenSell))
        result.append(contentsOf: _encode(order.amountBuy, order.tokenBuy))
        result.append(contentsOf: _encode(order.validSince))
        result.append(contentsOf: _encode(order.validUntil))
        result.append(contentsOf: _encode(order.lrcFee, "LRC"))
        let flag: [UInt8] = order.buyNoMoreThanAmountB ? [1] : [0]
        result.append(contentsOf: flag)
        result.append(contentsOf: [order.marginSplitPercentage])
        return result
    }
```
##### keccak256加密(通用，与具体交易和参数无关)
- 对message进行keccak256加密
- 加密后的字符串，加上"\u{0019}Ethereum Signed Message:\n32"前缀
- 对合并后的字符串再进行keccak256加密，作为待签名的字符串
- 用**交易主账户**签名
- 签名后的Hash，`[0,32)`是`r`, `[32,64)`是`s`,`[64,65)`的`int`形式是`v`

```swift
    open class func sign(message: Data, keystore: GethKeyStore, account: GethAccount, passphrase: String) -> (SignatureData?, String?) {
        let hashedMessage = message.sha3(SHA3.Variant.keccak256)
        let header: Data = "\u{0019}Ethereum Signed Message:\n32".data(using: .utf8)!
        let newData: Data = header + hashedMessage
        let secret = newData.sha3(SHA3.Variant.keccak256)
        
        do {
            // TODO:- Add timed Unlock
            _ = try? keystore.unlock(account, passphrase: passphrase)
            let accountAddress = account.getAddress()
            print("############### use address \(accountAddress?.getHex()) to sign")
            let hashedSignedMessage = try keystore.signHash(accountAddress, hash: secret)
            // let hashedSignedMessage = try keystore.signHash(accountAddress, hash: hashedMessage)
            let r = hashedSignedMessage.subdata(in: Range(0..<32))
            let s = hashedSignedMessage.subdata(in: Range(32..<64))
            let v = hashedSignedMessage.subdata(in: Range(64..<65)) // TODO:- Use length
        
            if let recId = Int(v.toHexString()) {
                let headerByte = recId + 27
                let hash = "0x" + hashedMessage.hexString
                let signature = (SignatureData(v: headerByte, r: r, s: s))
                return (signature, hash)
            } else {
                return (nil, nil)
            }
        } catch {
            print("Failed to signhash \(error)")
            return (nil, nil)
        }
    }
```

### 返回值
- String - txHash.
### 调用举例
