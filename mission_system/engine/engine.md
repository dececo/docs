# OpenTask Engine

OpenTask Engine 是任务引擎，目前主要用于任务列表的收集整理，后期会加入任务管理的功能，发展为任务管理系统。

目前服务处于开发调试阶段，可以调用，但会频繁更新，地址是 http://opentask.api.chainpower.io:8080。

## 依赖

引擎与智能合约一一对应，目前，引擎使用Rinkeby网络的智能合约调试，相关信息如下：
- Address: [0xF562a7c51a158ae6E6170Ef7905af5d1cE43d24A](https://rinkeby.etherscan.io/address/0xf562a7c51a158ae6e6170ef7905af5d1ce43d24a)
- Transaction: [0xb3cfac13487637c36a6bbfb9ee0d9a0096e7355e6cb443c80e55cfa1377569b5](https://rinkeby.etherscan.io/tx/0xb3cfac13487637c36a6bbfb9ee0d9a0096e7355e6cb443c80e55cfa1377569b5)
- Compiler: 0.4.25+commit.59dbf8f1.Emscripten.clang
- Optimize: false

详见[合约地址](/mission_system/open_task/releases/rinkeby/v0.1.0/OpenTask.md)和[合约接口](/mission_system/open_task/open_task.md)

## 接口

引擎提供json-rpc的接口，供前端使用，详见[API](./api.md)
