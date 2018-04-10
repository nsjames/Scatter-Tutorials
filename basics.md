# Creating your first Smart Contract

1) Changing some settings to listen on all interfaces
  - `sudo nano ~/.local/share/eosio/nodeos/config/config.ini`
  - Change the `http-server-address` to `0.0.0.0:8888`
  - `sudo nano ~/.bash_aliases`
  - Add a `#` in front of the `cleos` alias we set up before, we will no longer use it
  - restart `nodeos`

2) Setting up the `comptract.sh` script
  - `cd ~`
  - `mkdir scripts`
  - `cd scripts`
  - `nano comptract.sh`

```
#!/bin/bash

if [[ $# -ne 2 ]]; then
    echo "USAGE: comptract.sh <ACCOUNT NAME> <Contract Name> from within the directory"
    exit 1
fi

ACCOUNT=$1
CONTRACT=$2

eosiocpp -o ${CONTRACT}.wast ${CONTRACT}.cpp && 
eosiocpp -g ${CONTRACT}.abi ${CONTRACT}.cpp && 
cleos set contract ${ACCOUNT} ../${CONTRACT}
```

  - `chmod +x comptract.sh`
  - Add `export PATH=$PATH:~/scripts` to the end of your `~/.profile` file
  - `export PATH=$PATH:~/scripts`

3) Create a new account for the `basics` contract
  - `cleos create account eosio basics <PUBLIC KEY> <PUBLIC KEY>`


4) Creating the basics contract
  - Make a new directory inside of your `shared_contracts` folder called `tutorials` and then another inside of that called `basics`
  - Create a new cpp file called `basics.cpp`

```
#include <eosiolib/eosio.hpp>
#include <eosiolib/print.hpp>

using namespace eosio;
using namespace std;

class basics : public contract {
    using contract::contract;

    public:
        basics( account_name self ) :
            contract(self),
            _statuses(_self, _self){}

        // @abi action
        void test( name sender, string status ){
            require_auth(sender);

            auto iter = _statuses.find(sender);
            if(iter == _statuses.end()) _statuses.emplace( sender, [&]( auto& row) {
                row.sender = sender;
                row.status = status;
            });
            else _statuses.modify( iter, 0, [&]( auto& row) {
               row.status = status;
            });
        }

    private:

        // @abi table
        struct statuses {
            name sender;
            string status;

            name primary_key() const { return sender; }
            EOSLIB_SERIALIZE( statuses, (sender)(status) )
        };

        multi_index<N(statuses), statuses> _statuses;


};

EOSIO_ABI( basics, (test) )
```

4) Compiling and testing
  - `cd /media/sf_shared_contracts/tutorials/basics/`
  - `comptract.sh basics basics`
  - `cleos push action basics test '["asdf"]' -p eosio`
  - `cleos push action basics test '["eosio", "Hello World"]' -p eosio`
