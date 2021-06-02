# Token Standard With Transfer Offer Handlers

- **PSP Number:** [To be assigned (=number of the initial PR to the PSPs repo)]
- **Authors:** @vminkov @limechain
- **Status:** Draft
- **Created:** 2021-06-01
- **Reference Implementation** pseudocode

# Summary

Enable receivers of token/asset transfers to handle the transaction in an arbitrary way.

# Abstract

A better token standard that fixes a lot of the issues of the ERC20 standard by replacing the `transfer()`
function and adding a few extras.

# Motivation

First, Ethereum's ERC20 standard brought a lot of pain to centralized exchanges because it forced them to create a
unique deposit wallet address for each user in the database in order to make sure it is exactly this user who made the
token transfer and then to fund this intermediary wallet address with ETH in order to be able to forward the tokens to
a cold wallet address. There were many attempts to make a fix for this (forwarding smart contracts, infinite `approve()`
on the intermediary wallet address, etc.), but the costs were never really taken off of the receiver of the transaction.

Second, the ERC20 `transfer()` allowed anyone to send unsolicited tokens to any address, including locking the tokens
forever in a smart contract, sending all kinds of trash tokens to Vitalik Buterin's address or having to use Uniswap
before making a second transaction to make the payment in the desired token/asset. This feature of `transfer()` is
likely to make the use of ERC20 wrapped assets uncomfortable for institutional investors who do not want to be sent
tokens of toxic assets or toxic senders.

To summarize, enabling the receiver to include their own transfer handler logic will significantly improve the use of
asset-backed tokens, allowing things like
- privately recognizing the sender/reason of a direct transfer to a cold wallet by attaching a custom identifier to the
  transfer
- leaving unsolicited token transfers as not accepted by default or even reject them automatically
- automatically convert the transferred asset to another desired asset by using a DEX router
- split the transaction costs arbitrary between the sender and the receiver by refunding the sender in the same tx or
converting a chunk of the asset to the native token, so the validator fee is topped up
- require prior to the tx or subsequently do KYC/AML checks before allowing the token asset to be added in the cold
  wallet/smart contract
- cross-chain bridges will be able to support direct transfer offers made from the wallet apps UI standard transfer functions

# Specification

The ERC20 `transfer(to, amount)` function is removed and replaced by a `makeOffer(to, amount, memo)` function that
instead of writing the transfer to the `Map<Address, uint> balances` map, first writes it to another map
`Map<Address, Map<Address, (uint, String, uint)>> offers` and then looks up if the receiver has specified an address of a
smart contract that implements a default transfer handler function and calls it. If no such is specified by the
receiver, the transfer is locked forever until the receiver decides to act upon it.
A second function `makeLimitedTimeOffer(to, amount, memo, expirationTime)` will enable the sender to retrieve the sent
tokens after some offer expiration time.

# Implementation

```pseudocode
IERC20Killer {
    // replaces transfer(to, amount)
    fn makeOffer(to, amount, memo) -> bool;

    // replaces transferFrom(from, to, amount)
    fn makeDelegatedOffer(from, to, amount, memo) -> bool;

    fn makeLimitedTimeOffer(to, amount, memo, expirationTime) -> bool;
    fn takeOfferedTransfer(from, to, index) -> bool;
    fn redirectOfferTransfer(from, to, index) -> bool;
    fn getOffer(from, to, index) -> (uint, String, uint);
    fn registerDefaultHandler(handler);
}

ITokenOfferHandler {
    fn getToken() -> Address;
    fn getOwner() -> Address;
    fn handleOffer(from, index) -> bool;
}

ExampleToken implements IERC20Killer {
    Map<Address, uint> balances;
    Map<Address, Map<Address, (uint, String, uint)>> offers;
    Map<Address, Address> ownersToHandlers;

    // returns true in case the offer is immediately taken, false otherwise
    fn makeOffer(to, amount, memo) -> bool {
        balances[env().msgSender] -= amount;
        var offersFromSender = offers[to][env().msgSender];
        offersFromSender.push((amount, memo, env().block.timestamp))
        if handlers[to] != null {
            return ITokenOfferHandler(handlers[to]).handleOffer(env().msgSender, handlerOffersFromSender.length - 1);
        }

        return false;
    }

    fn makeDelegatedOffer(from, to, amount, memo) -> bool {
        allowances[owner][env().msgSender] -= amount;
        balances[owner] -= amount;

        if handlers[to] != null {
            var handlerOffersFromSender = offers[to][from];
            handlerOffersFromSender.push((amount, memo, env().block.timestamp))
            return ITokenOfferHandler(handlers[to]).handleOffer(from, handlerOffersFromSender.length - 1);
        } else {
            var receiverOffersFromSender = offers[to][from];
            receiverOffersFromSender.push((amount, memo, env().block.timestamp))
        }

        return false;
    }

    fn makeLimitedTimeOffer(to, amount, memo, expirationTime) -> bool {
        // same as makeOffer() but expirationTime is added to env().block.timestamp
        ...
    }

    fn takeOfferedTransfer(from, to, index) -> bool {
        // either owner or handler
        require(to == env().msgSender || ownersToHandlers[to] == env().msgSender);
        var (amount, memo, expirationTime) = offers[to][from][index];
        require(env().block.timestamp <= expirationTime);

        balances[to] += amount;
        Arrays.remove(offers[to][from][index]);

        return true;
    }

    // adds amount to the balance of the handler
    // the handler has to make sure it is redirected elsewhere
    fn redirectOfferTransfer(from, to, index) -> bool {
        // has to be called by the handler
        require(ownersToHandlers[to] == env().msgSender);
        var (amount, memo, expirationTime) = offers[to][from][index];
        require(env().block.timestamp <= expirationTime);

        balances[ownersToHandlers[to]] += amount;
        Arrays.remove(offers[to][from][index]);

        return true;
    }

    fn getOffer(from, to, index) -> (uint, String, uint) {
        return offers[to][from][index];
    }

    fn registerDefaultHandler(handler) {
        ownersToHandlers[env().msgSender] = handler;
    }
}

MyTokenOfferHandler implements ITokenOfferHandler {
    Address owner;
    Address token;

    // this example handler swaps all token deposits for DOT automatically
    fn handleOffer(from, index) -> bool {
        var (amount, memo, expirationTime) = IERC20Killer(token).getOffer(from, owner, index);
        if env().block.timestamp >= expirationTime {
            if memo.length == 64 && memo.startsWith("exchange_user_obfuscated_id_") {
                // forward to owner (cold storage)
                return IERC20Killer(token).takeOfferedTransfer(from, owner, index);
            } else if memo.startsWith("payment_id_") {
                // adds amount to the balance of the handler
                IERC20Killer(token).redirectOfferTransfer(from, owner, index);
                IERC20Killer(token).approve(env().contractAddress, Uniswap().address, amount);
                return Uniswap().swapForDOT(env().contractAddress, token, amount);
            }
        } else {
            emit TooLateEvent();
        }
        return false;
    }
}

Uniswap {
    UniswapPool pool;

    fn listToken(token) -> bool {
        pool.registerHandler(); // pool calls IERC20Killer(token).registerDefaultHandler(pool);
    }

    fn swapForDOT(from, token, amount) -> bool {
        IERC20Killer(token).makeDelegatedOffer(env().msgSender, pool, amount, "swap_for_DOT");
        WrappedDOTToken().makeOffer(env().msgSender, amount, "swap_for_DOT");
        return true;
    }
}

UniswapPool implements ITokenOfferHandler {
    fn getToken() -> Address {
        return ExampleToken().address;
    }
    fn getOwner() -> Address {
        return env().contractAddress;
    }

    fn handleOffer(from, index) -> bool {
        return IERC20Killer(token).takeOfferedTransfer(from, getOwner(), index);
    }
}

OldERC20Token implements ERC20 {
}

WrappedERC20 implements ERC20Killer {
    Address oldErc20TokenAddress;
    Address handlerAddress;
    
    WrappedERC20(oldErc20TokenAddress) {
        this.oldErc20TokenAddress = OldERC20Token(oldErc20TokenAddress);
        UnwrapHandler handler = new UnwrapHandler(env().contractAddress, oldErc20TokenAddress);
        this.handlerAddress = handler.address;
        this.registerDefaultHandler(handler.address);
    }
    
    fn makeOffer(to, amount, memo) -> bool;
    fn makeDelegatedOffer(from, to, amount, memo) -> bool;
    fn makeLimitedTimeOffer(to, amount, memo, expirationTime) -> bool;
    fn takeOfferedTransfer(from, to, index) -> bool;
    fn redirectOfferTransfer(from, to, index) -> bool;
    fn getOffer(from, to, index) -> (uint, String, uint);
    fn registerDefaultHandler(handler);
    
    fn approve(spender, amount); // as implemented in ERC20
    fn allowance(owner, spender); // as implemented in ERC20
    
    fn wrapErc20Tokens(amount) {
        if OldERC20Token(oldErc20TokenAddress).transferFrom(env().msgSender, env().contractAddress, amount) {
            _mint(env().msgSender, amount);
            require(OldERC20Token(oldErc20TokenAddress).increaseAllowance(this.handlerAddress, amount));
        }
    }
    
    
    fn unwrapErc20Tokens(amount) {
        require( 
            makeDelegatedOffer(env().msgSender, env().contractAddress, "unwrap", env().block.timestamp);
        );
    }
    
    fn handlerBurn(amount) {
        require(env().msgSender == handlerAddress);
        
        _burn(amount);
    }
}

UnwrapHandler implements ITokenOfferHandler {
    Address token;
    Address oldErc20TokenAddress;

    UnwrapHandler(tokenAddress, oldErc20TokenAddress) {
        this.token = tokenAddress;
        this.oldErc20TokenAddress = oldErc20TokenAddress;
    }

    fn getToken() -> Address {
        return token;
    }
    
    fn getOwner() -> Address {
        return token;
    }

    fn handleOffer(from, index) -> bool {
        var (amount, memo, expirationTime) = IERC20Killer(token).getOffer(from, getOwner(), index);
        if memo.startsWith("unwrap") {
            require(IERC20Killer(token).takeOfferedTransfer(from, getOwner(), index));
            token.handlerBurn(amount);
            require(OldERC20Token(oldErc20TokenAddress).transferFrom(token, env().msgSender, amount));
            return true;
        } else {
            IERC20Killer(token).redirectOfferedTransfer(from, getOwner(), index);
            ERC20(oldErc20TokenAddress).transfer(from, amount);
            return false;
        }
    }
}
```

## Tests

No tests yet.

## Copyright
[no copyright](https://creativecommons.org/publicdomain/zero/1.0/).
