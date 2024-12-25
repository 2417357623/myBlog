---
title: solidity入门学习
date: 2024-12-25 21:54:48
tags:
---


## Reference

[solidity by example](https://solidity-by-example.org/variables/)



## Variable

### Variable Type 

Local  state  globel 

[[solidity内置方法]]

### Constant vs immutable

- Constant value is  fixed and stored in the contract's bytecode.
- Immutable value is assigned dynamically during contract deployment ,in constructor ,and remains fixed afterward. 

Both Improve gas efficiency.
### Primitive

uint256，bool, address, bytes

### Data Locations

Storeage, memory, calldata

```js
//不能直接在 `storage` 类型的局部变量中使用 `new` 或通过值初始化来创建一个新的 `MyStruct` 实例。`storage` 是引用类型，意味着它必须引用合约状态中的数据（如映射或数组中的元素），而不能是一个新的内存中的结构体。

function f() public {
    // call _f with state variables
    _f(arr, map, myStructs[1]);

    // get a struct from a mapping
    MyStruct storage myStruct = myStructs[1];
    // create a struct in memory
    MyStruct memory myMemStruct = MyStruct(0);
}
```

Internal 和 private 的函数的入参和返回值可以是 storage，但是其余的只能是 memory 或者 calldata，因为这些会被外部的调用，我们不能让外面这些调用者改变我内部的状态。

```js

function f() public {

	MyStruct storage myStruct = myStructs[1];
	MyStruct memory myStruct2 = myStructs[2];
    // call _f with state variables
    _f(arr, map, myStruct,myStruct2);
}


function _f(
    uint256[] storage _arr,
    mapping(uint256 => address) storage _map,
    MyStruct memory _myStruct,
    MyStruct storage _myStruct2
) internal {
    _myStruct.foo += 1;
    _myStruct2.foo += 1;
}
```

Memory 不会修改传入的参数的值，但是 storage 的参数会被函数里面的逻辑修改。

同时，我们可以把一个 storage 赋值给 memory，相当于创建了一个复制版本


*默认值*

![](attachment/Pasted%20image%2020241211230810.png)

### Mapping

Mapping 不支持 memory，所以不支持作为参数传入，也不支持遍历，这和 js 的 map 有所不同


## 函数


### Destructuring assignment 

```js
function destructuringAssignments()
    public
    pure
    returns (uint256, bool, uint256, uint256, uint256)
{
    (uint256 i, bool b, uint256 j) = returnMany();

    // Values can be left out.
    (uint256 x,, uint256 y) = (4, 5, 6);

    return (i, b, j, x, y);
}
```

解构元组，解构赋值，并且返回元组。

### 返回值

可以返回元组，并且解构赋值别的函数的返回的元组。

返回的基本类型无所谓，基本类型不需要定义 location。
返回如果是动态类型，那么就应该是memory，不能返回一个 storage。因为它们是引用类型，并且可能会引发副作用，也会造成数据泄露。


### 修饰词

- Pure, declares that no state variable will be changed or read.
- View, declares that no state will be changed.

### Modifire

Modifiers are code that can be run before and / or after a function call.

修饰符将验证逻辑与函数主体分离，提高代码可读性和可维护性。防止重入攻击。


## Visibility

*Functions and state variables* have to declare whether they are accessible by other contracts.

![](attachment/Pasted%20image%2020241211223345.png)


## Constructor

Two ways to initialize parent contract with parameters

```js
// Pass the parameters here in the inheritance list.
contract B is X("Input to X"), Y("Input to Y") {}

contract C is X, Y {
    // Pass the parameters here in the constructor,
    // similar to function modifiers.
    constructor(string memory _name, string memory _text) X(_name) Y(_text) {}
}
```

## Error

An error will undo all changes made to the state during a transaction.

Require should be used to validate conditions such as
- inputs  
- conditions before execution
- return values from calls to other functions


## Event

`Events` allow logging to the Ethereum blockchain. Is a cheap form of storage。

日志被存储在该交易发生的区块里而不是合约的状态变量里，所以日志不能被这个发出的合约监测，但可以被监测区块的人监测到。

事件参数分为两种参数：indexed and non-indexed，区别在于有没有放进 topics 栏位。
事件最多允许三个参数被索引，其他没有索引的参数将会被一起编码并放到 data 里去。

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Event {
    // Event declaration
    // Up to 3 parameters can be indexed.
    // Indexed parameters helps you filter the logs by the indexed parameter
    event Log(address indexed sender, string message);
    event AnotherLog();

    function test() public {
        emit Log(msg.sender, "Hello World!");
        emit Log(msg.sender, "Hello EVM!");
        emit AnotherLog();
    }
}

```

交易的日志部分

![](attachment/Pasted%20image%2020241224220756.png)


## Interface

满足合约的交互需求

你可以在合约中声明接口，然后用该接口来定义如何调用其他合约的函数。合约 A 通过接口与合约 B 交互时，只需要知道接口的签名，而不需要了解 B 合约的具体实现。

```js
contract A {
    function incrementCounter(address _counter) external {
        ICounter(_counter).increment();  // 调用 ICounter 接口中的 increment 方法
    }

    function getCount(address _counter) external view returns (uint256) {
        return ICounter(_counter).count();  // 调用 ICounter 接口中的 count 方法
    }
}

```

## Try/catch

可以针对外部合约调用和合约创建做异常的捕获，让这些异常不会终止交易的进行，回滚。