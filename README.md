## 概述

数字原生系列开发均基于Silvia链，Silvia目前完全兼容EVM，从开发视角来看仅在交易权限上有限制，目前限制有：

- 仅拥有创建合约权限的地址可创建合约
- 仅通过审核的合约可被调用
- 仅通过KYC的地址可发送交易

合约审核及KYC目前请先联系数字原生完成，公共入口即将提供

## 开发相关

### 页面开发
数字原生为页面开发提供了一个Web3的Provider和Signer，可以完全兼容Web3开发模式。

具体方法为引入数字原生的SW3包，使用SW3Provider或SW3Signer初始化ethers.js，之后和其他Web3应用开发没有区别。SW3会在需要时显示页面让用户完成授权地址和交易签名操作。

### 后台开发

### KYC
需要发交易上链的地址必须在数字原生完成KYC，数字原生会将地址的KYC状态置为完成，之后方可发交易。

### RPC
Silvia链的RPC完全兼容Geth，目前RPC地址为

Beta测试网:
- RPC: https://rpc.beta.silvia.link
- Chain ID: 49368
- 区块链浏览器: https://scan.beta.silvia.link

正式网:
- RPC: https://rpc.silvia.link
- Chain ID: 8848
- 区块链浏览器: https://scan.silvia.link

### Graph

目前可用的subgraph有 `/subgraphs/name/szys/nft/graphql`，追加在对应网络的graph服务域名后即可查看。具体可查询参数可访问后查看。

**注意**: 地址需要转为全小写查询

- Beta测试网： https://query.beta.shuziyuansheng.com
- 正式网： https://query.shuziyuansheng.com

#### 参考样例

- 查询用户的token
```graphql
query MyQuery {
  tokens(first: 10, where: {owner: "0x000...000"}) {
    collection {
      id
    }
    tokenID
    tokenURI
  }
}
```
- 查询token的持有者
```graphql
query MyQuery {
  tokens(first:10,where: {collection: "0x000...000"}) {
    owner {
      id
    }
    tokenID
    tokenURI
  }
}
```
- 查询token的转移信息
```graphql
query MyQuery {
  transactions(where: {token:"0x000...000-1"}, orderBy:timestamp){
    from {
      id
    }
    to {
      id
    }
    timestamp
    block
  }
}
```

### 合约调用

#### 转赠合约
转赠合约用于控制token的可转赠间隔，是否可以通过转赠合约转赠受两个条件限制，一个是转赠合约中记录哪些合约的token可转赠的开关，一个是用户需approve授权转赠合约转移token。

转赠合约的代码及ABI可参考
https://scan.silvia.link/address/0x691F7b4797d41464ad15366C9E5AfDA6F04Bba2c/contracts

读取和写入接口可参考
- https://scan.silvia.link/address/0x691F7b4797d41464ad15366C9E5AfDA6F04Bba2c/read-proxy
- https://scan.silvia.link/address/0x691F7b4797d41464ad15366C9E5AfDA6F04Bba2c/write-proxy

#### 委托合约
委托合约用于接受用户的交易委托，代用户进行销售，如拍卖。流程为用户向委托合约提交token的委托意向，合约接受后将token转入合约，合约根据实际成交情况将token转移给新持有者。

委托合约的代码及ABI可参考
https://scan.silvia.link/address/0xF6156B8581cEc2db6c0345Bd5a30fB716CB9E585/contracts

读取和写入接口可参考
- https://scan.silvia.link/address/0xF6156B8581cEc2db6c0345Bd5a30fB716CB9E585/read-proxy
- https://scan.silvia.link/address/0xF6156B8581cEc2db6c0345Bd5a30fB716CB9E585/write-proxy

委托状态变化顺序及方法调用为
- 用户调用register注册，注册时传入价格、acceptTimeout和dealTimeout
- 委托合约的owner调用preVerify进入预审核，或调用refuse直接拒绝
  - 预审核后会将token设置不可转赠
  - preVerify后会开始记录时间，超过acceptTimeout后用户可调用unregister撤回
- 用户调用approve授权委托合约可转移token
- 委托合约的owner调用accept将token转进合约，或调用refuse拒绝
  - 如果用户未approve，此步会失败
  - accept后会开始记录时间，超过dealTimeout后用户可调用unregister撤回
- 委托合约的owner调用deal将token标记为已成交
  - 进入此状态后用户将无法unregister
- 委托合约的owner调用finish将token转移给新持有者
  - 此步会检查token的最后转移时间，如果未超过合约中的transferInterval将会失败
  - 此步成功后会更新token的最后转移时间

委托合约目前有如下时间限制
- transferInterval
完成委托时会检查距离上一次转移是否超过这个时间，未超过则无法完成
- minAcceptInterval
register可传入的acceptTimeout最小值
- maxAcceptInterval
register可传入的acceptTimeout最大值
- minDealInterval
register可传入的dealTimeout最小值
- maxDealInterval
register可传入的dealTimeout最大值

