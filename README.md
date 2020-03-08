# Solidity Monolithic Flattener

*The Solidity Monolithic Flattener instantly transforms your solidity source code in to a logically equivalent contract with increased run-time performance. We have demonstrated reductions of ~20% gas on popular contracts using this technique.*

INPUT: A single solidity source file (flatten from multiple files using [Solidity Flattener](https://github.com/BlockCatIO/solidity-flattener/blob/development/flattener/core.py))

OUTPUT: An equivalent smart contract that is optimized for run-time performance

SYNOPSIS:  `python sol-monolythic.py sourceCode.sol [ContractToDeploy]`

## What is a monolithic contract?

A monolithic solidity source file has these properties:

1. There are no `import` statements.
2. Exactly one `contract` is defined.
   - Logically follows: The contract does not inherit from any other contract.
   - Note: One or more `interface`s *may* be defined.
3. No `modifier` is defined.
4. Every function of the contract is `external`.
   1. Logically follows: No function will call any other function.
   2. Logically follows: No function will call itself.
5. All variables are defined as `private`.

It stands to reason that any solidity file can be transformed into a monolithic contract unless:

- The contract to deploy has a circular function call.
- The solidity file fails to compile before transformation.

## Work plan

Each feature in this work plan transforms a valid solidity file into another valid solidity file. After all transformations are performed, the result will be a monolithic contract.

- [ ] Feature: create Python script that opens input solidity file, compiles it and returns the original code

  - [ ] The program will exit with status 1 if the file fails to compile

- [ ] Feature: unroll all `imports`

  - At this time we will not implement this feature

- [ ] Feature: unroll all contract inheritance for the ContractToDeploy

  - Note: contracts other than ContractToDeploy may still have inheritance after this feature is implemented

  - Implement this feature in this way:

    1. Make a list of all inheritances of the ContractToDeploy
    2. Unroll the first in the list, if any
    3. Repeat if the list is not empty

  - Note: implement this feature by applying iteratively while the ContractToDeploy still uses inheritance

  - [ ] Refactor names of all inherited `private` variables

  - [ ] Refactor names of all inherited `private` functions

  - [ ] Refactor constructor calls to ancestors

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        int private a;
        constructor(int b) public {
            b;
        }
    }
    contract B is A {
        constructor(int c)
            public
            A(5)
        {
            c;
        }
    }
    ```

    will become

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        int private a;
        constructor(int b) public {
            b;
        }
    }
    contract B {
        int private ___SMF__A__a;
        constructor(int c) public
        {
            int ___SMF__A_constructor__b = 5;
            ___SMF__A_constructor__b;
            c;
        }
    }
    ```

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        int public a;
        constructor(int _){a = _;}
    }
    contract B is A {
        constructor(int _)A(_ + 1){}
    }
    contract C is B {
        constructor(int _)B(_ * 2){}
    }
    contract D is C {
        constructor()C(1){}
    }
    ```

    must produce a result of 3 before and after transformation

- [ ] Feature: convert all `public` variables to `internal` and make explicit accessors

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        int[] public a;
    }
    contract B {
        int[] private __a;
        function a(uint256 index) public returns(int) {
            return __a[index];
        }
    }
    ```

    will become

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        int[] private ___SMF__A__a;
        function a(uint256 index) public returns(int) {
            return ___SMF__A__a[index];
        }
    }
    ```

- [ ] Feature: refactor all `modifier` usage to became inline at the call site

  - Postcondition: every function will not use the `modifier` keyword

  - Note: use the correct `modifier` resolution order that solidity uses, the following example emits 1 and then 2:

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        event log(int);
        modifier logOne() {
            emit log(1);
            _;
        }
        modifier logTwo() {
            emit log(2);
            _;
        }
        function test() logOne logTwo external {
        }
    }
    ```

  - [ ] After refactor is complete, remove all existing `modifier`s

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        modifier a(int b) {
            b;
            _;
        }
        function c() a(6) external {
        	7;
        }
    }
    ```

    will become

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
    	function c() external {
            int private ___SMF__A__a__b = 6;
            ___SMF__A__a__b;
            7;
    	}
    }
    ```

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        modifier twice() {
            _;
            _;
        }
        function test(int b) twice external {
            int a;
        }
    }
    ```

    will become

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        function test(int b) twice external {
            int ___SMF__A__twice_1__a;
            int ___SMF__A__twice_2__a;
        }
    }
    ```

    Note for test case: function parameters (`int b` here) are l-values. That means you do not need to refactor them. They are constant across both function calls.

- [ ] Feature: convert all contracts other than the ContractToDeploy into interfaces

  - Precondition: The ContractToDeploy does not inherit from those contracts (link to issue number)

  - Precondition: The contracts to be converted do not have any public variables (link to issue number)

  - Precondition: The contracts to be converted do not have any modifiers (link to issue number)

  - [ ] Convert any `public` functions to `external`

  - [ ] Remove state variables, and all functions which are not `external`

  - [ ] Remove implementations of all functions

  - [ ] Rename from `contract` to `interface`

    - Solidity versions before 0.5.0 do not support inheritance for interfaces
    - Do not implement this item until 0.5.0 is released

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
    }
    contract B is A {
        int private a;
        function b() internal {}
        function c() public {}
    }
    ```

    will become

    ```solidity
    pragma solidity ^0.4.24;
    interface A {
    }
    interface B /* is A */ {
        function c() external {}
    }
    ```

- [ ] Feature: refactor every `function` to not call another `function`

  - Note: implement this feature iteratively, keep running it one-at-a-item until no items remain to be refactored (this will prevent infinite recursion)

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        function jump(int height) internal {
            int trajectory;
            trajectory;
        }
        function twice() external {
            jump(4);
            jump(5);
        }
    }
    ```

    will become

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        function jump(int height) internal {
            int trajectory;
            trajectory;
        }
        function twice() external {
            int ___SMF__A__jump_1__height = 4;
            int ___SMF__A__jump_1__trajectory;
            ___SMF__A__jump_1__trajectory;
            int ___SMF__A__jump_2__height = 5;
            int ___SMF__A__jump_2__trajectory;
            ___SMF__A__jump_2__trajectory;
        }
    }
    ```

- [ ] Feature: crash if any function calls itself

  - Prerequesite: all function calls to other function calls have been unrolled (link to issue number)

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        function a() private {
            a();
        }
    }
    ```

    will cause the program to abort with an error:

    >  Input file(s) include a circular reference function call, as such these files are not flattenable.
    >
    > *Exit return code 1*

- [ ] Feature: change every `public` function to `external`, remove functions that are not `external`

  - Precondition: no function will call any other function (link to issue number)

  - Precondition: no function does call itself (link to issue number)

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        function a() public {}
        function b() internal {}
    }
    ```

    will become

    ```solidity
    pragma solidity ^0.4.24;
    contract A {
        function a() external {}
    }
    ```

- [ ] Feature: the ContractToDeploy shall be kept, an interface whos type is referenced by a kept contract/interface shall be kept, all other contracts/interfaces shall be removed

  - Note: this feature is not part of the minimal viable product

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract A {}
    contract B is A {}
    contract C {}
    contract D { // <--------------- This is the designated ContractToDeploy
        B public b;
    }
    ```

    will become

    ```solidity
    pragma solidity ^0.4.24;
    contract A {}
    contract B is A {}
    contract D { // <--------------- This is the designated ContractToDeploy
        B public b;
    }
    ```

- [ ] Feature: remove collision-avoidance identifier naming if possible

  - Prerequesite: all features which add collision-avoidance to identifier names have finished running

  - Note: in order to all possible interdependencies, implement this feature follows:

    1. LABEL_1: Identify all possible renamings to perform
    2. For each renamings as renaming:
       1. Attempt renaming
       2. If renaming is successful GOTO LABEL_1

  - [ ] For each state variable, variable or function name, test the transformation ("renaming"):

    ```regex
    s/^___SMF.*___//
    ```

    - This must be implemented as a "not greedy" regex match
    - If the regex does not match, then the renaming is a failure
    - If the regex does match, then attempt to compile the new code, if it does not compile then the renaming is a failure
    - If the regex does match and the new code does compile, then keep the new code and start over

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract B {
        int private ___SMF__A__a;
        constructor(int c) public
        {
            int ___SMF__A_constructor__b = 5;
            ___SMF__A_constructor__b;
            c;
        }
    }
    ```

    will become

    ```solidity
    pragma solidity ^0.4.24;
    contract B {
        int private a;
        constructor(int c) public
        {
            int b = 5;
            b;
            c;
        }
    }
    ```

  - [ ] Test case

    ```solidity
    pragma solidity ^0.4.24;
    contract B {
        constructor(int c) public
        {
            int ___SMF__A_constructor__c = 5;
            ___SMF__A_constructor__c;
            c;
        }
    }
    ```

    will not change (because this would create a collision on initializing the identifier)
