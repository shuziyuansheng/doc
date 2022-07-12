## 概述

数字原生系列开发均基于Silvia链，Silvia目前完全兼容EVM，从开发视角来看仅在交易权限上有限制，目前限制有：

- 仅拥有创建合约权限的地址可创建合约
- 仅通过审核的合约可被调用
- 仅通过KYC的地址可发送交易

合约审核及KYC目前请先联系数字原生完成，公共入口即将提供

## 开发相关

### 数据规范
数字原生发售的NFT目前遵循ERC721标准，接口参考
- [IERC721](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/IERC721.sol)
- [IERC721Enumerable](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/extensions/IERC721Enumerable.sol)
- [IERC721Metadata](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/extensions/IERC721Metadata.sol)

#### Meta json
接口IERC721Metadata中tokenURI方法会返回指向json的URL，格式如下
```jsonc
{
  "name": "NFT名称", // 名称
  "description": "NFT简介", // 简介
  "image": "https://domain/poster_image", // 缩略图URL，必须是图片，建议为方形
  "raw": "https://domain/raw_data", // 原始文件URL，可以是图片、视频、3D
  "creator": "NFT作者名称", // 作者
  "publisher": "NFT发行方", // 发行方
  "background_color": "99ccff", // NFT图片推荐背景填充色，开头不含#的hex [可选]
  "external_url": "https://domain/url", // NFT扩展页面地址 [可选]
  "properties":  // 扩展属性 [可选]
  {
    "rarity": {"name": "R级", "probability": "0.0001"} // 例：珍惜度 [可选]
  }
}
```

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

目前可用的subgraph有 `/subgraphs/name/szys/nft/graphql`，追加在对应网络的graph服务域名后即可查看。不同网络的查询地址如下，具体可查询参数可访问后查看。

- Beta测试网： https://query.beta.shuziyuansheng.com
- 正式网： https://query.shuziyuansheng.com

**注意**: 区块链地址需要转为全小写查询

GraphQL的文档可参考 [GraphQL API](https://thegraph.com/docs/en/developer/graphql-api/)

目前subgraph里定义有如下数据信息，对应复数为列表查询，具体包含关系可以在graph查询界面右侧文档中点击相应结构体查看。

- collection: 一个collection是一个NFT合约，主要属性描述如下
  - id: 合约地址
  - name: NFT名称
  - tokens: 合约所有token
  - totalTokens: 合约共mint出来的token数量
  - transactions: 合约所有交易
- owner: 持有人
  - id: 区块链地址
  - tokens: 持有token信息
  - totalTokens: 持有token总数
  - fromTransactions: 作为from的交易
  - toTransactions: 作为to的交易
- token: Token信息
  - id: Token在graph中唯一ID，格式为`<合约地址>-<Token ID>`
  - collection: 对应合约
  - tokenID: Token在合约中的ID
  - tokenURI: Token meta json的地址
  - owner: 持有人
  - burned: 是否已销毁
  - transactions: 相关交易
- transaction
  - id: 交易hash
  - from: 发送方
  - to: 接收方
  - collection: 对应合约
  - token: 对应token


#### 参考样例
可将样例复制到graph查询界面中，左侧会展开可选择的查询项及查询条件供选择

- 查询合约列表
```graphql
query MyQuery {
  collections(first: 10) {
    id
    name
    totalTokens
  }
}
```
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

