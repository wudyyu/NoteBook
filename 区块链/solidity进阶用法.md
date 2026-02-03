### solidity进阶用法

> ABI

>> 在以太坊生态系统中, ABI (Application Binary Interface, 应用二进制接口) 是连接智能合约与外部应用 (如前端 DApp) 的关键桥梁。对于刚接触区块链和 Solidity 的初学者来说, 理解 ABI 的概念和作用至关重要。

> 什么是 ABI?

> > ABI, 全称为 Application Binary Interface, 即应用二进制接口。它定义了合约中的函数和事件如何与外部系统进行交互。换句话说, ABI 描述了合约的接口, 包括函数的名称、参数类型、返回值以及事件的结构等信息。
智能合约部署到以太坊区块链后, 其源代码对外不可见, 外部应用需要一种方式与合约进行交互。ABI 就是这种桥梁, 它让外部应用知道如何调用合约的函数、如何解码合约返回的数据以及如何监听合约的事件。

> ABI 的组成

> > 一个标准的 Solidity ABI 是一个 JSON (JavaScript Object Notation) 数组, 每个数组元素描述了合约中的一个函数、构造函数或事件。每个元素包含以下字段:

> > >> type: 描述类型, 如 function、constructor、event。

> > > > name: 函数或事件的名称。

> > > > inputs: 输入参数的详细信息, 包括类型和名称。

> > > > outputs: (仅限函数) 输出参数的详细信息。

> > > > stateMutability: 函数的状态可变性, 如 view (只读)、nonpayable (不接受以太币) 等。

> > > > anonymous: (仅限事件) 是否为匿名事件

> ABI 的作用

> > 1.函数调用: ABI 描述了如何编码函数调用的参数,并解码合约返回的数据。外部应用通过ABI知道如何与合约的函数进行交互。

> > 2.事件监听: ABI 定义了事件的结构,外部应用可以根据ABI监听和解析合约触发的事件。

> > 3.接口兼容性: ABI使得不同编程语言和工具能够一致地与合约进行交互,确保兼容性和互操作性。

当我们要调用一个函数时,使用ABI JSON 的规范的要求,进行编码,传给EVM,同时在EVM层生成的字节数据(如时间日志等),ABI JSON 的规范进行解码。

![image_227](../image_227.png)

> 合约ABI示例
>> 假设有以下简单Solidity合约

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private data;

    event DataStored(address indexed sender, uint256 data);

    function storeData(uint256 _data) public {
        data = _data;
        emit DataStored(msg.sender, _data);
    }

    function getData() public view returns (uint256) {
        return data;
    }
}
```

> 在编写 Solidity 合约后,编译工具(如 Solidity Compiler 或 Remix IDE)会自动生成ABI。
![image_228](../image_228.png)

> 对应的ABI如下：

```
[
    {
        "anonymous": false,
        "inputs": [
            {
                "indexed": true,
                "internalType": "address",
                "name": "sender",
                "type": "address"
            },
            {
                "indexed": false,
                "internalType": "uint256",
                "name": "data",
                "type": "uint256"
            }
        ],
        "name": "DataStored",
        "type": "event"
    },
    {
        "inputs": [],
        "name": "getData",
        "outputs": [
            {
                "internalType": "uint256",
                "name": "",
                "type": "uint256"
            }
        ],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "inputs": [
            {
                "internalType": "uint256",
                "name": "_data",
                "type": "uint256"
            }
        ],
        "name": "storeData",
        "outputs": [],
        "stateMutability": "nonpayable",
        "type": "function"
    }
]
```











































