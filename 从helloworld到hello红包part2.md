
# 从helloworld到hello红包之二

上次我们学会了用EOS智能合约发红包，发红包时需要合约方知道对方的EOS账号，并通过合约账号授权发定向红包（转账送钱）。今天我们要实现的是多人“领红包”的功能，固定金额，见者有份，分完为止，不需要通过合约的EOS账号授权。（有人问，为什么不做拼手气红包呢？因为智能合约的随机数比较特殊，有机会再单独讲。）

## 数据存储

_只要我记得你，你就永远不会消失的啊_
							----《寻梦幻游记》CoCo


首先，我们需要让合约记住已经领过红包的账号，机会均等，每人只能领一次。在EOS上，最简单的办法就是用table保存合约的状态和数据。

### 创建表格
首先，我们在helloworld.cpp里加上两个表格。其中 primary_key 是主键，bonustable是表名，bonus_t是表结构。

红包表：

```
struct bonus_t {
    uint64_t id;
    asset balance;
    uint64_t primary_key() const { return id;}
    EOSLIB_SERIALIZE( bonus_t, (id)(balance) )
};
multi_index<N(bonustable), bonus_t > _bonus;
```
领红包的用户表：

```
struct user_t {
    uint64_t id;
    account_name user;
    uint64_t primary_key() const { return user;}
    EOSLIB_SERIALIZE( user_t, (id)(user) )
};
multi_index<N(usertable), user_t> _users;
```
### 初始化表格
加上default constructor，把红包金额初始化为10 EOS。

```
	hello( account_name self )
	:contract(self),_bonus( _self, _self),_users( _self, _self) {
		asset quantity;
		quantity.set_amount(100000); // 10 EOS
		if ( _bonus.begin() == _bonus.end() )
		_bonus.emplace( self, [&](auto& a) {
			a.id = 1;
			a.balance = quantity;
		});
	}
}
```
### 检查用户状态
我们增加一个检查用户的函数checkuser，检查用户是否领过红包。如果用户没领过红包，就把用户的账号加到表格里。


```
bool checkuser( account_name user ) {
	bool found = true;
	auto itr = _users.find(user);
	if( itr == _users.end() ) {
		found = false;
	    itr = _users.emplace(_self, [&](auto& acnt){
	        acnt.id = _users.available_primary_key();
	        acnt.user = user;
	    });
	}
	return found;
}

```
### 更新红包金额
再把取红包金额的功能放到一个函数里：

```
asset getbonus() {
	asset quantity;
	auto itr = _bonus.begin();
	auto balance = itr->balance;
	eosio_assert( balance.amount > 0, "must be positive quantity" );
	if (balance.amount > 10000){
		quantity.set_amount(10000); // 1 EOS
		_bonus.modify(itr, _self, [&](auto& p) {
		    eosio_assert( p.balance.amount > quantity.amount, "we're out of bonus!" );
		    p.balance.amount -= quantity.amount;
		});
	} else {
		quantity = balance;
        itr = _bonus.erase(itr);
	}
	return quantity;
}
```
### 新的红包接口
最后，我们把用户检查和取红包金额的功能加到领红包的接口上，改造后的hi()接口不再传入金额。

```
/// @abi action hi
void hi( account_name to ) {
	if ( checkuser(to) )
		return;

	asset bonus = getbonus();
	eosio_assert( bonus.amount > 0, "must be positive quantity" );
	require_recipient( _self );
	require_recipient( to );

	action(
		permission_level{ _self, N(active) },
		N(eosio.token), N(transfer),
		std::make_tuple(_self, to, bonus, std::string("hello money"))
	).send();
      print( "Hello, here is some money for ", name{to} );
}
```
具体的测试方法，请参考《从helloworld到hello红包part1》

## 知识点总结

### 创建表格

```
struct user_t {
    uint64_t id;
    account_name user;
    asset balance;
    uint64_t primary_key() const { return user;}
    EOSLIB_SERIALIZE( user_t, (id)(user)(balance) )
};
multi_index<N(usertable), user_t> _users;
```
### 查询表格：
```
	auto itr = _users.find(user);
```
### 给表格添加元素：
```
	auto itr = _users.find(user);
	if( itr == _users.end() ) {
	    itr = _users.emplace(_self, [&](auto& acnt){
	        acnt.id = _users.available_primary_key();
	        acnt.user = user;
	    });
	}
```
### 更新表格：
```
	auto itr = _users.find(user);
	if( itr ！= _users.end() ) {
	    _users.modify( itr, _self, [&]( auto& acnt ){
	        acnt.user = user;
	        acnt.balance.amount = new_amount;
	    });
	}
```
### 清空表格：
```
    for (auto itr = _users.begin(); itr != _users.end();)
        itr = _users.erase(itr);
```
