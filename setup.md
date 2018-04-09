# Setup EOSIO Virtual Machine

## 1) Getting VirtualBox
  - https://www.virtualbox.org/wiki/Downloads

## 2) Getting Ubuntu 16.04
  - https://www.ubuntu.com/download/alternative-downloads
   
## 3) Setting up the Virtual Machine
  - Minimum requirements: at least 8096 ( 8GB ) RAM to spare and 30-50GB disk space

  - Create new Virtual Machine
  - Select Linux as the OS
  - Select Ubuntu as the distribution
  - Double click the new Virtual Machine
  - Load the Ubuntu 16.04 disk as the startup disk and click "Start"
  - Select both "Download Updates" and "Install third party" for best coverage.
  - Select "Erase disk and continue".
  - Select timezone and fill out user data
  - Select your language and click "Install Ubuntu"
  - Let it finish installing and restart
  - Press enter when prompted "Please remove installation medium"
  - Open a Terminal window ( ctrl+alt+t )
  - `sudo apt update`
  - `sudo apt upgrade`
  - `sudo apt-get install virtualbox-guest-dkms`
  - `sudo adduser <YOUR_USER_NAME_HERE> vboxsf`
  - Close the Virtual Machine and make sure to "Power Off", not just to exit and save it's state.
  
## 4) Setting up host connection
  - Go back to VirtualBox Manager
  - Right click your new virtual box
  - Click Settings
  - Click Network
  - Select "Bridged Adapter" from the Attached To dropdown

## 5) Setting up shared folder
  - While still in the Settings panel click "Shared Folders"
  - Click the little plus sign folder icon on the right to add a shared folder
  - Create a folder for contracts somewhere on your local machine called "shared_contracts" and select it for the Folder Path
  - Select "Auto-Mount" and make sure "Read Only" is NOT selected

## 6) Finishing Basic Setup
  - Save your settings.
  - Restart the Virtual Machine. You can see your shared folder in "Files" now with "sf_" in front of it. 
    * This is where you will be putting your contracts so that you can work on them in Windows but use them on Ubuntu
      without having to copy paste them between host/guest machines.
  - Click the Devices menu from the top again, click "Shared Clipboard" and select "Bidirectional" 
    * This will allow you to copy/paste between guest/host and make it easier to follow these tutorials
  - Open a Terminal window ( ctrl+alt+t )
  - `sudo apt install git`
  - `ifconfig | grep "inet addr" | grep Bcast`
  - This should show you an IP ( something like 192.168.56.101 ). Grab it and save it somewhere.
  
## 7) Setting up EOSIO
  ( Refer to the EOSIO wiki for changes to this flow. )
  - Open a Terminal window ( ctrl+alt+t )
  - `cd ~`
  - `sudo apt install git`
  - `git clone https://github.com/EOSIO/eos --recursive`
  - `cd eos`
  - `./eosio_build.sh`
    * If prompted to install dependencies type 1 and click enter to install them.
  - `cd build`
  - `sudo make install`

## 8) Testing EOSIO setup
  - Create a new empty document on your desktop to save some information for later ( I'll refer to it as "eostext" from now on )
  - Open a Terminal window ( ctrl+alt+t )
  - nodeos 
  - Press "ctrl+c" to stop nodeos now that it has created a config.ini file.
  - Let's find the genesis file using `sudo find / -name genesis.json`
  - It should come up with a few options, save the one that has "/eosio/nodeos/config" in it to your eostext file, for instance "/home/nsjames/.local/share/eosio/nodeos/config/genesis.json"
  - `sudo nano ~/.local/share/eosio/nodeos/config/config.ini`
  - Towards the bottom of the config file there should be a private key starting with "5", save it in your eostext file.
  - Edit the config.ini file, adding/updating the following settings to the defaults already in place:
  - You can press "ctrl+w" to find things inside of nano.
```
	# Find and change ( This will be the IP you saved before, something like 192.168.56.101 )
	http-server-address = 192.168.56.101:8888

	# Fill this with the genesis path you save in your eostext file before.
	genesis-json = /path/to/eos/source/genesis.json

	# Find and switch to true
	enable-stale-production = true

	# Find, remove "#", and change to "eosio"
	producer-name = eosio

	# Copy the following plugins to the bottom of the file
	plugin = eosio::producer_plugin
	plugin = eosio::wallet_api_plugin
	plugin = eosio::chain_api_plugin
	plugin = eosio::http_plugin
	plugin = eosio::account_history_api_plugin
```

  - Press "ctrl+x", then "y" and press enter to save the config file.
  - Open a new Terminal Window ( ctrl+alt+t )
  - Run "nodeos" and let is keep running in it's own window.

  - Setup an alias for cleos now that we're using a different IP
  - `sudo nano ~/.bash_aliases`
  - Add the following line to the end of that file ( replace IPs with the one you entered into the config.ini ):
      `alias cleos='cleos -H 192.168.56.101 -p 8888 --wallet-host 192.168.56.101 --wallet-port 8888'`
  - `source ~/.bashrc`


## 9) Creating a wallet
  - `cleos wallet create`
  - This will create a wallet and return a password for you. Save it in your eostext file.
  - Create two sets of keypairs using `cleos create key`, Save each keypair in your eostext file and label the first "Owner Key" and the second "Active Key"
  - Import the private keys you just created. The private key from the config.ini should already be inside your wallet
  - `cleos wallet import <PRIVATE_KEY>`
  - `cleos wallet keys`
  - Make sure there are 3 private keys in your wallet, if it has 2 import the key from the config.ini

## 10) Creating an account
  - `cd ~/eos/contracts/eosio.bios/`
  - `eosiocpp -o eosio.bios.wast eosio.bios.cpp`
  - `cleos set contract eosio ../eosio.bios -p eosio`
  - `cleos create account eosio currency <Owner Public Key> <Active Public Key>`
  - `cleos get account currency`

## 11) Uploading and testing your first contract
  - `cd ~/eos/contracts/currency`
  - `eosiocpp -o currency.wast currency.cpp`
  - `cleos set contract currency ../currency -p currency`
  - `cleos push action currency create '{"issuer":"currency","maximum_supply":"1000000.0000 CUR","can_freeze":"0","can_recall":"0","can_whitelist":"0"}' -p currency`
  - `cleos push action currency issue '{"to":"currency","quantity":"1000.0000 CUR","memo":""}' -p currency`
  - `cleos get table currency currency accounts`
  - `cleos push action currency transfer '{"from":"currency","to":"eosio","quantity":"20.0000 CUR","memo":"my first transfer"}' -p currency`
  - `cleos get table currency eosio accounts`
  - `cleos get table currency currency accounts`
