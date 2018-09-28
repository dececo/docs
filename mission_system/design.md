# 付费任务系统设计文档

## 一．引言

### 1. 目标读者

本文档面向付费任务系统的产品、开发、测试，及其他相关人员。对此文档有任何问题，都可以[新建Issue](https://github.com/dececo/docs/issues/new)向我们提问，也可以联系Chole（微信 **grey0805**）加群讨论。

### 2. 项目介绍

付费任务系统是 [DEC](dechain.io) 旗下生态系统中重要的垂直行业细分领域产品，专门针对软件开发相关人员，包括但不限于开发、产品、运营及外包等。

### 3. 名词解释

* **付费**: 该文档中所说的付费，通常指的是支付 [DET](https://github.com/dececo/docs/blob/master/token/det/DET.md)。
* **解决方案**: 对应任务的系列解决方法或完成后的产出物，如教程或者软件包。
* **ABI**: Application Binary Interface，应用二进制接口，以太坊智能合约编译时会随二进制一起提供。
* **MissionID**: 全局唯一的任务ID，由[任务管理平台](#任务管理平台)在任务上传时生成。
* **SolutionID**: 全局唯一的解决方案ID，由[任务管理平台](#任务管理平台)在任务完成时生成。

### 4. 参考资料

* **以太坊(ethereum)**: [https://ethereum.org/](https://ethereum.org/)
* **Oraclize**: [https://docs.oraclize.it/](https://docs.oraclize.it/)

## 二．总体设计

付费任务系统是一个基于区块链的分布式系统，运行于多种兼容的云计算平台上，并且可以平行扩展。

使用该软件系统，可以实现发布开发任务、结算佣金、争议解决等协作任务。

### 1. 需求概述

- **发布开发任务**：用户可以在系统内发布自己的付费任务，指定一定数额的 [DET](https://github.com/dececo/docs/blob/master/token/det/DET.md) 作为回报，该任务应该被所有人看到；
- **发布解决方案**：对任务感兴趣的用户，可以留言尝试解决任务，并在完成后提供自己的解决方案；
- **提起争议诉讼/任务鉴定**：如果付费方（甲方）和方案提供方（乙方）对任务完成结果意见不一致时，可以通过请求第三方鉴定的方式解决争议，类似于法律诉讼；
- **围绕任务互动**：任务列表是公开的，因此所有用户包括发布者在内，均可以围绕任务进行互动，比如评论、回复评论、点赞等；

### 2. 软件结构

付费任务系统由[智能合约](#智能合约)，[任务管理平台](#任务管理平台)、[任务展示平台](#任务展示平台)、[任务鉴定平台](#任务鉴定平台)、[任务鉴定平台](#任务鉴定平台)构成。

- **[智能合约（Smart Contract）](#智能合约)**：系统交易部分，会全部上链，做到公开透明。因此交易逻辑会由多组智能合约协同完成。
- **[任务管理平台（Task Engine）](#任务管理平台)**：托管用户上传的任务说明和后续的交付物。是智能合约最主要的驱动方。
- **[任务展示平台（Task Client）](#任务展示平台)**：展示现有任务列表，以及一些较为简单的互动，如点赞、关注等，包括但不限于客户端、网页等。
- **[任务鉴定平台（Task Arbitrator）](#任务鉴定平台)**：对任务完成与否进行鉴定。一般不需要，只有交易双方出现分歧时才引入。
- **[任务撮合系统（Task Matcher）](#任务撮合系统)**：对任务进行撮合并推送到相关用户。因前期采用广播模式，该模块不需要实现。

各组件及交互可见下图。

<img src='https://g.gravizo.com/svg?
cloud "Ethereum" {
        [Smart Contract];
};
node "AWS EC2" {
        [Engine] -> [Arbitrator];
        [Matcher] -left-> [Engine];
};
[Engine] ..> [Smart Contract]: Asynchronous Call;
[Smart Contract] ..> [Arbitrator]: Oraclize \nAsynchronous Call;
package "Client" {
        [Browser];
        [iOS];
        [Android];
};
[Browser] ..> [Engine];
[iOS] ..> [Engine];
[Android] ..> [Engine];
database "AWS RDS" {
        [MySQL];
};
[Engine] --> [MySQL];
database "AWS ElastiCache" {
        [Redis];
};
[Engine] --> [Redis];
'/>

#### 2.1 发布任务

用户发起任务时，使用任务展示平台客户端（PC网页、iOS/安卓客户端）上传相应的说明和必要的文件，任务管理平台验证后，调用智能合约抵押发布人的对应资产，并在成功后通知发布人。

##### 发布任务时序图

<img src='https://g.gravizo.com/svg?
@startuml;
actor Publisher as User;
participant "Client" as A;
participant "Engine" as B;
participant "Smart Contract" as C;
User -> A: Publish Task;
activate A;
A -> B: Upload Task;
activate B;
B -> C: Send DET;
activate C;
C --> B: Return TxHash;
B --> A: Task Created;
A --> User: Task Created;
deactivate A;
C -> C: Confirm Transaction;
deactivate C;
B -> C: Is TxHash Confirmed?;
activate C;
C --> B: Confirmed;
deactivate C;
B -> A: Message: "Task Published";
deactivate B;
activate A;
A -> User: Informed("Published");
deactivate A;
@enduml
'/>

#### 2.2 发布解决方案

任务发布后，允许任何人浏览访问，感兴趣的用户可能会尝试提交解决方案并期望获得对应报酬。解决者使用任务管理平台上传解决方案说明和必要的文件，任务管理平台验证完整性，获得通过后会得到通知。

##### 发布解决方案时序图

<img src='https://g.gravizo.com/svg?
@startuml;
actor Solver as User;
participant "Client" as A;
participant "Engine" as B;
participant "Smart Contract" as C;
User -> A: Publish Solution;
activate A;
A -> B: Upload Solution;
activate B;
B --> A: Solution Published;
deactivate B;
A --> User: Solution Published;
@enduml
'/>

注意：
**此时并未发生资产转移**。

#### 2.3 确认解决方案

如果任务发布人对解决方案较为满意，会直接“确认”解决方案，此时任务管理平台会尝试调用智能合约转账对应报酬到解决方案提供方帐户。同样的，成功后双方会获得通知。

##### 确认解决方案时序图

<img src='https://g.gravizo.com/svg?
@startuml;
actor Publisher as User;
actor Solver as User2;
participant "Client" as A;
participant "Engine" as B;
participant "Smart Contract" as C;
activate B;
B -> A: Solution Uploaded;
deactivate B;
activate A;
A -> User: Message: "Solution Uploaded";
deactivate A;
== Solution Confirming ==;
User -> A: Confirm Solution;
activate A;
A -> B: Confirm Solution;
activate B;
B -> C: Send DET to Solver;
activate C;
C --> B: Return TxHash;
B --> A: Solution Confirmed;
A --> User: Solution Confirmed;
deactivate A;
C -> C: Confirm Transaction;
deactivate C;
B -> C: Is TxHash Confirmed?;
activate C;
C --> B: Confirmed;
deactivate C;
B -> A: Message: "DET Payed";
deactivate B;
activate A;
A -> User: Informed("Payed");
A -> User2: Informed("Payed");
deactivate A;
@enduml
'/>

#### 2.4 请求鉴定

如果发布者不满意解决方案，或者希望获得专业意见，可以通过请求鉴定的方式，获得任务鉴定平台的专业意见，并按裁定的任务完成比例支付报酬。

##### 请求鉴定时序图

<img src='https://g.gravizo.com/svg?
@startuml;
actor Publisher as User;
actor Solver as User2;
participant "Client" as A;
participant "Engine" as B;
participant "Arbitrator" as D;
participant "Smart Contract" as C;
activate B;
B -> A: Solution \nUploaded;
deactivate B;
activate A;
A -> User: Message: "Solution Uploaded";
deactivate A;
User -> User: Not Satisfied;
== Arbitration ==;
User -> A: Request Arbitration;
activate A;
A -> B: Request \nArbitration;
activate B;
B -> D: Request \nArbitration;
activate D;
B -> A: Arbitration \nSend;
deactivate B;
A -> User: Arbitration Send;
A -> User2: Arbitration Send;
deactivate A;
D -> D: Process \nArbitration;
D -> C: Arbitration \nCompleted;
activate C;
C -> C: Transaction \nConfirmed;
deactivate C;
D -> B: Arbitration \nCompleted;
deactivate D;
activate B;
B -> A: Arbitration \nCompleted;
deactivate B;
activate A;
A -> User: Message: "Arbitration Completed";
A -> User2: Message: \n"Arbitration Completed";
deactivate A;
@enduml
'/>

## 三．详细设计

以下分模块介绍详细设计方案及相关指标。

### 智能合约

智能合约完成的是核心交易逻辑，包括提交任务时抵押 [DET](https://github.com/dececo/docs/blob/master/token/det/DET.md)，完成任务时支付 [DET](https://github.com/dececo/docs/blob/master/token/det/DET.md) 等。

#### 1．功能

1. 向指定地址转账指定数额的 [DET](https://github.com/dececo/docs/blob/master/token/det/DET.md)，可用于指定任务的抵押或者支付；
2. 获取指定任务的鉴定结果（是否胜诉及可获得财产比例）；
3. 从系统账户取回 [DET](https://github.com/dececo/docs/blob/master/token/det/DET.md)，可用于鉴定胜诉时取回资产；
4. 核心数据使用对方公钥加密（ ECDSA 算法），拥有私钥才能解密查看；

#### 2．性能

目前基于以太坊，执行速度取决于`Gas Price`设置，不建议使用超高`Gas`加速，推荐将所有交易和回调**异步**化。

#### 3．输入项目

预先定义的合约函数，一般由[任务管理平台](#任务管理平台)发起调用。当然也可以由其他任何第三方发起调用。

#### 4．输出项目

除合约状态改变外，还会输出 Event Log 。部分场合可能还会通过 Oraclize 调用其他平台，如[任务管理平台](#任务管理平台)和[任务鉴定平台](#任务管理平台)等。

#### 5．算法

无特殊算法。

#### 6．程序逻辑

以太坊智能合约代码风格采用 [Solidity 官方推荐风格](https://solidity.readthedocs.io/en/v0.4.25/style-guide.html#)。

合约代码样例如下。

```solidity
// OpenTask.sol
pragma solidity ^0.4.25;
import "github.com/dececo/dechainio/contracts/DET.sol";

contract OpenTask  {

    DET det;
    address owner;

    constructor(address initialDETAddress, address initialOwner) public {
        det = DET(initialDETAddress);
        owner = initialOwner;
    }

    function updateDETAddress(address newDETAddress) public onlyOwner {
    }

    function updateOwner(address newOwner) public onlyOwner {
    }

    function publish(string missionID, uint amountInWei) public {
    }

    function solve(string solutionID) public {
    }

    function confirmSolution(string solutionID) public {
    }

    function confirmArbitration(string solutionID, string arbitration) public {
    }

}
```

#### 7．接口

见示例代码。

#### 8．存储分配

EVM 内部存储分配取决于具体实现，不使用任何以太坊外部存储。

#### 9．限制条件

任何人都可能对合约发起调用，但只有符合条件的才能执行成功。例如，只有完成了任务的人才能取走属于自己的那部分佣金。

#### 10． 测试要点

1. 完成任务后才能取走相应比例的佣金；
2. 任务发起后资金抵押，任务取消前或完成前，资金不能移动；
3. 通过其他合约调用`OpenTask`，不会改变现金流走向；
更多要点详见功能。

### 任务管理平台

任务管理平台托管用户上传的任务说明和后续的解决方案交付物。是智能合约最主要的驱动方。数据存储在 AWS RDS 数据库中，兼容 MySQL。

#### 1．功能

1. 简单的登录功能，无需实名验证，并可以把 ID 与公钥/地址关联；
2. 上传任务说明（引入版本控制前，暂定上传后不能更改），生成全局唯一的任务`MissionID`；
3. 上传解决方案说明（交付物，引入版本控制前，暂定上传后不能更改），生成全局唯一的`SolutionID`；
4. 对任务说明和完成说明留言评论，及点赞互动；
5. 确认接受任务交付/任务完成；
6. 请求鉴定机构鉴定任务是否完成及完成百分比；
7. 按完成百分比支付佣金（调用智能合约）；
8. 提供鉴定结果（供智能合约调用）；
9. 权限控制（仅能查看自己发布的任务的解决方案，或自己提交的解决方案）；

#### 2．性能

服务将部署在 AWS 上（最好是使用 Docker），并使用 RDS 作为存储。
1. QPS 无苛刻要求；
2. 内存要求 8G 以下；

#### 3．输入项目

用户上传或点击。

#### 4．输出项目

- 动作完成的相关提示
- 智能合约调用
- 写数据库等持久化存储（RDS）

#### 5．算法

无特殊算法。

#### 6．程序逻辑

待补充细化。

#### 7．接口

1. 登录；
2. 上传/下载任务说明/完成说明/附件；
3. 点赞互动；
4. 任务确认；
5. 发起诉讼/鉴定；
6. 接受鉴定结果；
7. （胜诉方）取回资产（调用智能合约）；

#### 8．存储分配

[AWS RDS](https://amazonaws-china.com/cn/rds/)。

#### 9．限制条件

依赖项要尽量精简，推荐使用 Docker 方式部署，能打包成 Docker 镜像最好。

#### 10． 测试要点

见功能。

### 任务展示平台

任务展示平台展示现有任务列表，以及一些较为简单的互动，如点赞、关注等，包括但不限于客户端、网页等。

以下功能仅以网页端为例。

#### 1．功能

1. 展示任务列表，包含完成情况和点赞互动统计；
2. 任务详情页面，可以查看完成情况和解决方案细节（仅有权限的任务解决方案）；
3. 点赞互动；
4. 上传任务或解决方案；
5. 发起鉴定；
6. 确认解决方案或鉴定；
7. 取回资产；

#### 2．性能

服务将部署在 AWS 上（最好是使用 Docker）。
1. QPS 无苛刻要求；
2. 内存要求 4G 以下；

#### 3．输入项目

无

#### 4．输出项目

详见功能。

#### 5．算法

无特殊算法。

#### 6．程序逻辑

待补充。

#### 7．接口

无外部调用方。

#### 8．存储分配

无。

#### 9．限制条件

依赖项要尽量精简，推荐使用 Docker 方式部署，能打包成 Docker 镜像最好。

#### 10． 测试要点

详见功能。

### 任务鉴定平台

任务鉴定平台对任务完成与否进行鉴定。一般不需要，只有交易双方出现分歧时才引入。

#### 1．功能

1. 投票决定是否接受任务完成，以及任务完成百分比；
2. 提供任务完成百分比，供智能合约查询；

#### 2．性能

服务将部署在 AWS 上（最好是使用 Docker），并使用 RDS 作为存储。
1. QPS 无苛刻要求；
2. 内存要求 8G 以下；

#### 3．输入项目



#### 4．输出项目



#### 5．算法

待补充投票规则。

#### 6．程序逻辑



#### 7．接口

##### **鉴定结果查询接口**
标准 [JSON-RPC](https://www.jsonrpc.org/specification) 接口，支持 HTTPS 协议，该接口主要供智能合约通过 Oraclize 调用。URL 路径中推荐含有网络和版本，方便针对不同合约版本进行加密，如`https://www.identify.com/rinkeby/v1/verify`。

请求参数如下：
  - `solution_id`: 要查询解决方案ID

返回参数如下：
- `solution_id`: 要查询解决方案ID
- `completed`: 是否完成，完成: `true`, 未完成: `false`
- `percentage`: 完成百分比，`0-100`的整数，比如`30`表示该任务完成了`30%``

注意：
**在合约调用中该URL地址加密**。

以下是调用示例：
```bash
// Request
curl -s -H 'Content-Type: application/json' -X POST --data '{"jsonrpc":"2.0","method":"dec_verify","params":[{"solution_id":"4d2be90d-24da-4857-8590-a9d3233c888d"}],"id":67}' 'https://www.identify.com/rinkeby/v1/verify'

// Result
{
  "id":67,
  "jsonrpc": "2.0",
  "completed": true,
  "percentage": 100
}
```

#### 8．存储分配

[AWS RDS](https://amazonaws-china.com/cn/rds/)。

#### 9．限制条件

无。

#### 10． 测试要点

见功能。

### 任务撮合系统

任务撮合系统对任务进行撮合并推送到相关用户。因前期采用广播模式，该模块不需要实现。

#### 1．功能

对已发布的任务进行撮合，找出匹配的潜在解决者，进行消息推送，吸引该用户的注意。

#### 2．性能

暂无。

#### 3．输入项目

- 现有已发布未解决的任务列表
- 现有已完成过任务（已提议过解决方案）的用户

#### 4．输出项目

- 任务与潜在解决者的关联关系
- 该次撮合的佣金（如用户看过提示信息后开始尝试完成任务，应从最终受益中支付该佣金）

#### 5．算法

自定义撮合算法是撮合系统的核心，每个实现均可以不同。

#### 6．程序逻辑

1. 撮合引擎获得任务列表和用户列表；
2. 尝试撮合并产生匹配结果；
3. 推送匹配结果到用户；
4. 用户浏览推送；
5. 任务展示系统记录下浏览行为及待支付佣金；
6. 用户完成任务；
7. 结算函数（智能合约）扣除佣金后把剩余资产发放给用户；

#### 7．接口

待补充

#### 8．存储分配

待补充

#### 9．限制条件

多个撮合引擎均推送给用户同一任务时，以用户阅读的最早的推送为准。后续即使用户多次阅读类似推送消息，也不会再支付任何佣金。

#### 10． 测试要点

见功能。
