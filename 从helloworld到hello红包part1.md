# 从helloworld到hello红包之一

今天，我们来看看如何在EOS智能合约内部发起转帐（发红包）。在下面第一部分的例子代码里，我们在官方Helloworld标准的打招呼hi()接口的基础上,增加新的参数“金额”，实现在打招呼的同时，通过合约内部转帐给对方账户发红包。第二部分的测试中，合约账号eosonetest1通过新的打招呼接口，给 eosonetest2 发送转帐。
使用合约内转帐，和标准转帐相比，可以在合约里加上各种条件和控制逻辑，实现丰富多彩的合约玩法。

## 从零开始
 **创建 helloworld 智能合约**
```
$ eosiocpp -n helloworld
created helloworld from skeleton
```

**得到的helloworld.cpp源代码是这个样子的**
```
#include <eosiolib/eosio.hpp>

using namespace eosio;

class hello : public eosio::contract {
  public:
      using contract::contract;

      /// @abi action 
      void hi( account_name user ) {
         print( "Hello, ", name{user} );
      }
};

EOSIO_ABI( hello, (hi) )
```
现在我们开始把它改造成“红包合约”
**添加必要的头文件**
```
#include <eosiolib/asset.hpp>
```
**给hi() 函数添加通证**
```
void hi( account_name to, const asset& quantity ) {
```
**加上一些安全检查**
```
		require_auth( _self );
		eosio_assert( quantity.is_valid(), "invalid token" );
		eosio_assert( quantity.amount > 0, "must be positive quantity" );

		require_recipient( _self );
		require_recipient( to );
```
**添加合约里转账**
```
		action(
			permission_level{ _self, N(active) },
			N(eosio.token), N(transfer),
			std::make_tuple(_self, to, quantity, std::string("hello money"))
		).send();
```

**改造好的hi()函数变成这个样子：**
```
	/// @abi action hi
	void hi( account_name to, const asset& quantity ) {
		require_auth( _self );
		eosio_assert( quantity.is_valid(), "invalid token" );
		eosio_assert( quantity.amount > 0, "must be positive quantity" );

		require_recipient( _self );
		require_recipient( to );

		action(
			permission_level{ _self, N(active) },
			N(eosio.token), N(transfer),
			std::make_tuple(_self, to, quantity, std::string("hello money"))
		).send();
        print( "Hello, here is some money for ", name{to} );
    }
```
## 在 Jungle testnet 上实测

**给合约账号购买适当的RAM**
```
cleos -u http://jungle.cryptolions.io:18888 system buyram -k eosonetest2 eosonetest2 1000
```
**把合约布署到合约账号**
```
cleos -u http://jungle.cryptolions.io:18888 set contract eosonetest2 ../helloworld -p eosonetest2
```

**合约里转账需要添加的 eosio.code permission**
```
cleos -u http://jungle.cryptolions.io:18888 set account permission eosonetest2 active '{"threshold": 1,"keys": [{"key": "EOS8xxxxxxxxxxxxxxx","weight": 1}],"accounts": [{"permission":{"actor":"eosonetest2","permission":"eosio.code"},"weight":1}]}' owner -p eosonetest2@owner
```

### 测试网上的输出

**测试前的余额**
```
@joel-Lenovo-Y520:helloworld $ cleos -u https://jungle.eosio.cr:443 get currency balance eosio.token eosonetest1 EOS
7970.3411 EOS
@joel-Lenovo-Y520:helloworld $ cleos -u https://jungle.eosio.cr:443 get currency balance eosio.token eosonetest2 EOS
6733.2414 EOS

```
**通过合约发红包**
```
@joel-Lenovo-Y520:helloworld $ cleos -u http://jungle.cryptolions.io:18888 push action eosonetest2 hi '["eosonetest1", "103.0000 EOS"]' -p eosonetest2
executed transaction: 71fxxxxxxxxxab706fxx3ea1cxxxxx1f70f05cxxxxx  120 bytes  2483 us
#   eosonetest2 <= eosonetest2::hi              {"to":"eosonetest1","quantity":"103.0000 EOS"}
>> Hello, here is some money for eosonetest1
#   eosonetest1 <= eosonetest2::hi              {"to":"eosonetest1","quantity":"103.0000 EOS"}
#   eosio.token <= eosio.token::transfer        {"from":"eosonetest2","to":"eosonetest1","quantity":"103.0000 EOS","memo":"hello money"}
#   eosonetest2 <= eosio.token::transfer        {"from":"eosonetest2","to":"eosonetest1","quantity":"103.0000 EOS","memo":"hello money"}
#   eosonetest1 <= eosio.token::transfer        {"from":"eosonetest2","to":"eosonetest1","quantity":"103.0000 EOS","memo":"hello money"}
warning: transaction executed locally, but may not be confirmed by the network yet    ] 

```
**测试后的余额**
```
@joel-Lenovo-Y520:helloworld $ cleos -u https://jungle.eosio.cr:443 get currency balance eosio.token eosonetest1 EOS
8073.3411 EOS
@joel-Lenovo-Y520:helloworld $ cleos -u https://jungle.eosio.cr:443 get currency balance eosio.token eosonetest2 EOS
6630.2414 EOS

```

