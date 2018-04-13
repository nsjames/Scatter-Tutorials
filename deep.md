# Diving deeper into EOSIO Smart Contracts: Typedefs, Assertions and Singletons

Here's some topics we'll be covering
* typedefs
* Assertions
* Singletons



## Typedefs

Typedefs are a way to abstract away types to something more friendly. 

Consider the following: 
```
uint64_t varname = 1;
id varname = 1;
```

It's far easier to read and discern what the second variable is even though they are exactly the same. 
It's trivial to do this, let's look at a simple example:

```
typedef uint64_t id;
```

When we put something like this in our contract we can now use `id` instead of `uint64_t` to keep our code clean and succint.
However, this is only the tip of the iceberg.

```
// @abi table
struct users {
    account_name user;
    id index;

    account_name primary_key(){ return user; }
    EOSLIB_SERIALIZE( users, (user)(index) )
};
typedef multi_index<N(users), users> user_ids_table;
```

When we define a table in EOSIO we are actually defining a typedef which abstracts away 
` multi_index(uint64_t code, uint64_t scope) `
into something more friendly like `user_ids_table` so that when we want to use it we can simply do something like 
```
user_ids_table table(_self, _self);
```
where `_self` is a reference to the contract itself.


## Assertions

Assertions are our way of validating conditional logic within a contract and reverting if conditions are not met as well as 
returning an error message about the failure.

```
eosio_assert(1 == 1, "One is not equal to one.");
eosio_assert(1 == 2, "One is not equal to two.");
```

You can use any type of boolean logic within the first parameter to validate conditions.

There are a few helpers available to do specific things like assert authorization and permissions.
```
require_auth(someAccountName);
require_auth2(someAccountName, N(owner OR active));
``` 

## Singletons


[Singletons](https://github.com/EOSIO/eos/blob/master/contracts/eosiolib/singleton.hpp) 
are a great way of storing configuration data and single reference mappings based on scope/account.


Let's look at a way of storing a configuration struct into a singleton which has an 
`account_name application` 
which must be used to sign every action request along with the sender's signature.

```    
struct config {
    account_name application;

    EOSLIB_SERIALIZE( config, (application) )
};

typedef singleton<N(contractname), N(config), N(contractname), config>  configs;
```

We've created a singleton called `configs` to hold the `config` struct. To fill this singleton we're going to also 
create an action called `own` which will require authorization from the contract account ( owner ) as well as the sender which 
will become the `application`.

**If singleton `scope` parameters are omitted they default to `_self`.**

```
void own( account_name application ){
    require_auth(_self);
    require_auth(application);
    
    eosio_assert(!configs::exists(), "Configuration already exists");
    configs::set(config{application});
}
``` 

Now we can use this singleton elsewhere to easily grab the configs from the blockchain without having to iterate through
tables by specifying the scope which is then used as the index.

```
// Remember, no specified scope defaults to _self
configs::get().application
```

So how can we make use of this when dealing with actual mappings? 
Well lets put together some of the concepts we've learned so far and make an atomic integer mapping to a user's account
which can only be created using an `owner` authority permission and the application key.

```
#include <eosiolib/eosio.hpp>
#include <eosiolib/print.hpp>
#include <eosiolib/singleton.hpp>

using namespace eosio;

class deep : contract {
    using contract::contract;

public:
    deep( account_name self ) : contract(self){}

    void setapp( account_name application ){
        require_auth(_self);
        require_auth(application);

        eosio_assert(!configs::exists(), "Configuration already exists");
        configs::set(config{application});
    }

    void setacc( account_name user ){
        require_auth2(user, N(owner));
        require_auth(configs::get().application);
        eosio_assert(!ids::exists(user), "User already has an ID");
        ids::set(nextId(), user);
    }

    void getacc( account_name user ){
        eosio_assert(ids::exists(user), "User does not have an ID");
        print("User's ID is: ", ids::get(user));
    }

    void removeacc( account_name user ){
        require_auth(user);
        require_auth(configs::get().application);
        eosio_assert(ids::exists(user), "User does not have an ID");
        ids::remove(user);
    }

private:
    typedef uint64_t id;


    struct config {
        account_name application;

        EOSLIB_SERIALIZE( config, (application) )
    };

    typedef singleton<N(contractname), N(config), N(contractname), config>  configs;


    typedef singleton<N(deep), N(ids), N(deep), id>  ids;
    typedef singleton<N(deep), N(lastId), N(deep), id>  lastId;

    id nextId(){
        id lid = lastId::exists() ? lastId::get()+1 : 0;
        lastId::set(lid);
        return lid;

    }
};

EOSIO_ABI( deep, (setidx)(setacc)(getacc)(removeacc) )
```


