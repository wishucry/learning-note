# 从EVM底层实现gas优化

## gas 计算原理

一笔以太坊交易的 gas 消耗计算公式如下：
```
gas = txGas + dataGas + opGas
```

如果是一笔普通交易（EOA转账或者合约调用），则 txGas 为固定值21000，如果是合约部署类型的交易，则 txGas 为53000。交易中 InputData 的每个零字节需要花费4个 gas，每个非零字节需要花费16个 gas，这构成了 dataGas 的消耗。opGas 是指运行完交易中所有的操作指令所需要的 gas。

在交易 gas 的构成中，dataGas 一般远小于 opGas，优化的空间也比较小，优化 gas 的主要焦点在 opGas 上。大部分的OP消耗的 gas 数量是固定的（比如 ADD 消耗3个 gas），少部分OP的gas消耗是可变的，也是 gas 消耗的大头（比如 SLOAD 一个全新的非零值需要花费至少20000个 gas）。

所以gas优化，其实主要是对opGas的消耗进行缩减。

## 数据类型

### uint256与uint8

一般情况下，定义大小为256bit的变量是gas消耗最优的选择。因为EVM一个slot是256bit，而当定义一个uint8或者uint128的变量时，数据未填满slot，EVM会自动派生指令将slot剩余空间填充为0。另外在取值进行计算时，也会按照一个slot 256bit为单位进行计算。

某些情况下，例如开发者定义了相邻的数个uint8或uint128等bit的变量时，编译器会将这些变量合并到一个slot中进行打包，这种情况下使用uint8或uint128等bit的变量是能够减少存储gas的，但是注意在读写时，如果对整个slot中的单个变量进行操作，也会对应增加额外指令，增加gas消耗。

代码示例如下：

```solidity
struct Data {
	uint128 a;
	uint128 b; // a b 变量占用一个slot
	uint256 c;
}

Data public data;

constructor(uint128 a_， uint128 b_, uint256 c_) public {
	Data.a = a_;
	Data.b = b_;
	Data.c = c_;
}
```

### 使用常量

在 solidity 中，声明为 constant 或者 immutable 的状态变量即常量。constant 修饰的常量的值在编译时确定，而 immutable 修饰的常量的值在部署时确定。尽量使用常量，常量是合约字节码的一部分，不占用存储插槽，使用常量比变量更省 gas。在部署时，常量消耗的 gas 更少。

### 频繁读写使用memory替代storage

为了避免存储操作，可以在数组后面加上memory关键字，表明这个数组是仅仅在内存中创建，不需要写入外部存储，并且在函数调用结束时它就解散了。与在程序结束时把数据保存进storage的做法相比，内存运算可以大大节省gas开销。

### 使用默克尔树。
在合约中保存一组数据的merkleRoot，然后提供verify方法验证某条数据在这组数据中。相比使用一个mapping或数组来保存全部数据，减少了gas消耗。

## 事件

在传达合约内状态的改变时，多使用事件。事件是`EVM`上比较经济的存储数据的方式，每个大概消耗几百到几千 `gas`；相比之下，链上存储一个新变量至少需要20,000 `gas`。

每个事件分为 topic 部分和 data部分，data部分的每个字节消耗8个gas，topic部分最多4个256bit，每个topic消耗375个gas，topic[0]一般默认是事件签名哈希，如果时匿名事件，这部分gas甚至都可以省略。

## 改写EVM Dispatch

每个函数最终都会形成一个函数选择器（selector），然后编译器会自动生成一个庞大的if-else分支，组成一个供用户调用的Dispatch。

可通过以改写EVM的Dispatch，把调用比较频繁的函数选择器放到if-else的前部，这样利用“短路原则”就可以节省判断函数调用的gas消耗了。

## 使用内联汇编

## 开启编译器优化