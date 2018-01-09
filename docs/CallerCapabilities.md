# Caller Capabilities

Ethl introduces the concept of **caller capabilities**. While traditional computer programs have an entry point (the *main* function), smart contracts do not. After a contract is deployed on the blockchain, its code does not run until an Ethereum transaction is received. Smart contracts are in fact more akin to RESTful web services: any Ethereum user/contract call is a public function of a given contract. The function then gets executed until completion, an error is thrown, or the gas limit has been reached.

Contracts can maintain a state, which persists on the EVM blockchain. Developers use state to record data through function calls. Part of what can be recorded are Ethereum addresses, which can be used to impose limits on which functions can be called by a given address. The EVM bytecode specification doesn't include special instructions to perform these kinds of checks. Usually, developers prefix their function body by a caller check, as follows:

```
function giftAccount(address: Address, amount: Int) {
	if (caller != manager) {
		throw
	}
	// body
}
```

Adding these checks is cumbersome and error-prone. 

Contract functions in Ethl are declared within `ContractBehaviorDeclaration`s, which are used to restrict which users/contracts are allowed to call the functions.

The above example is declared as follows in Ethl:

```
Bank :: (manager) {
	func giftAccount(address: Address, amount: Int) {
		// body
	}
}
```

The functions in the `Bank :: (manager)` block can only be called by the manager of the bank.

Caller capabilities can be any property declared in the contract's declaration, as long as their type is `Address` or `[Address]`.

Note: The special caller capability `any` allows any caller to execute the function in the group.

Calls to Ethl functions are validated both at compile-time and runtime.

## Static checking

In an Ethl function, if a function call to another Ethl function is performed, the compiler checks that the caller has sufficient caller capabilities.

Consider the following example.

```
Bank :: (any) {
	func foo() {
		// Error: Capability "any" cannot be used to perform a call to a 
		// function for "manager"
		bar()
	}
}

Bank :: (manager) {
	func bar() {}
}
```

Within the context of `foo`, the caller is regarded as `any`. It is not certain that the caller also has capability `manager`, so the compiler rejects the call.

## Dynamic checking

### Attempt function calls

It is still possible for the caller of `bar` to have the capability `manager`.

For these cases, two additional language constructs exist:

- `try? bar()`: The function `bar`'s body is executed if at runtime, the caller's capability matches `bar`'s. The expression `try? bar()` returns a boolean.
- `try! bar()`: If at runtime, the caller's capability doesn't match `manager`, an exception is thrown and the body doesn't get executed. Otherwise, it does.

### Calls from Ethereum users or non-Ethl smart contracts

Functions to contracts on the Blockchain can also be called by users directly, through an Ethereum client, or another non-Ethl smart contract.

For those cases, Ethl checks at runtime whether the caller has the appropriate capabilities to perform the call, and throws an exception if not.

## Multiple capabilities

A contract behavior declaration can be restricted by multiple caller capabilities.

Consider the following contract behavior declaration:

```
Bank :: (manager, accounts.keys) {
	func forManagerOrCustomers() {}
}
```

The function `forManagerOrCustomers ` can only be called by either the manager, or any of the accounts registered in the bank.

Calls to functions of multiple capabilities are accepted if **each** of the capabilities of the enclosing function are compatible with **any** of the target function's capabilities.

Consider the following examples:

#### Insufficient capabilities

```
Bank :: (manager, accounts.keys) {
	func forManagerOrCustomers() {
		// Error: "accounts.keys" is not compatible with "manager"
		forManager()
	}
}

Bank :: (manager) {
	func forManager() {}
}

```

#### Sufficient capabilities

```
Bank :: (manager, accounts.keys) {
	func forManagerOrCustomers() {
		// Valid: "manager" is compatible with "manager", and "accounts.keys" is
		// compatible with "accounts.keys" 
		forManagerOrCustomers2()
	}
}

Bank :: (accounts.keys, manager) {
	func forManagerOrCustomers2() {}
}

```

#### `any` is compatible with any capability

```
Bank :: (manager, accounts.keys) {
	func forManagerOrCustomers() {
		// Valid: "manager" is compatible with "manager" (and "any", too), and "accounts.keys"
		// is compatible with "any"
		forManagerOrCustomers2()
	}
}

// The caller capability "manager" has no effect: "any" is compatible with any capability
Bank :: (manager, any) {
	func forManagerOrCustomers2() {}
}

```

### Static and dynamic checking

Just like single-capability definitions, capability lists are checked both at compile-time and runtime.

## Capability Binding

Capabilities can be bound to temporary variables.

Consider the following example.

```
Bank :: (a <- anyOf(accounts.keys)) {
	mutating func withdraw(amount: Int, destination: Address) {
		let value = accounts[a]
		accounts[a] -= amount
		send(value, destination)
	}
}

```

`withdraw` can be called by any caller which has an account in the bank. The `anyOf` function succeeds if the caller's capabilities is compatible with a capability in `accounts.keys`. If there exists such a capability, it is bound to the variable `a` as an `Address`, which can then be used in the body of the functions.