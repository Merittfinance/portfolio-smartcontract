# Portfolio Smartcontract

_a smart contract for holding multiple loan contracts_

## Description

The portfolio smart contract is a contract for collecting multiple loans into one transferrable set of ERC20 tokens. To simplify governance it is a fixed portfolio, ie the composition can only be changed before any of the tokens are distributed. This is to avoid surprises if the portfolio composition changes after the portfolio has been sold, in particular if loans are removed.


## Interfaces

From in interface point of view the portfolio contract looks very similar to a *payment conduit contract* with some additional function related to the creating and managing the portfolio. However, the semantics of this contract and the interface functions is not entirely the same is in the *payment conduit contract*.

### ERC20 Interface

The contract implements an interface that is compliant with the ERC20 token standard (described [here][ERC20]). That means it implements the following functions

- `totalSupply`
- `balanceOf(address _owner) constant returns (uint256 balance)`
- `transfer(address _to, uint256 _value) returns (bool success)`
- `transferFrom(address _from, address _to, uint256 _value) returns (bool success)`
- `approve(address _spender, uint256 _value) returns (bool success)`
- `allowance(address *_owner*, address *_spender*) constant returns (uint256 remaining)`

and the following events

- `Transfer(address indexed _from, address indexed _to, uint256 _value)`
- `Approval(address indexed _owner, address indexed _spender, uint256 _value)`


### Portfolio Interface

The *portfolio interface* is the interface that implements the specific functionality that is needed to model portfolios of loans and other financial instruments. The portfolio contract is very similar to the *payment conduit contract* in many respects, but there are important differences. For example, it does not allow for payments directly into the contract, other than those coming from the smart contracts the portfolio contract owns. Also, the portfolio smart contract does not immediately forward the payments received to its tokens. Instead, received Ether is stored in the contract and paid out periodically.

The following functions are more or less the same as in the *payment conduit contract*:

**payments**. The `payments` function returns a list of all payments that have been disbursed from contract. Note that there is no retained tokens mechanism in this case: all payments are made to the address of the respective token

    [
        {height: <block height>, amount: <payment amount>},
        {height: <block height>, amount: <payment amount>},
        ...
        {height: <block height>, amount: <payment amount>},
    ]

**recipients**. The `recipients(height)` function returns a list of addresses that have received a payment at blockheight _height_ together with the number of tokens the address held for that particular payment. The list contains retained tokens, and it returns its data as follows

    [
        {addr: <recipient address>, tokens: <number of tokens>},
        {addr: <recipient address>, tokens: <number of tokens>},
        ...
        {addr: <recipient address>, tokens: <number of tokens>},
    ]

**details**. The `details` function returns (static) details of the underlying contract. Details are yet to be determined, but the result of the function call might look as follows:

    {
        /* generic data fields */
        version: "1.0"                      /* the version of the smart contract */

        payer: <payer address>              /* returns null (not meaningful for portfolio) */

        type_f: "portfolio"                 /* friendly name of the contract type */
        subtype_f: "fixed portfolio"        /* friendly name of the contract subtype */
        type_ns: "types.meritt.co"          /* namespace URL for the contract definition */
        type_nm: "portf"                    /* exact type of the contract (unique in namespace) */

        /* this URL is pointing to exact contract terms of this specific contract */
        contract: "https://types.meritt.co/portf?addr=<contract addr>"                    

        /* portfolio-specific data fields */
        constituents: [
            {addr: <address of constituent contract>, amount: <number of tokens>},
            {addr: <address of constituent contract>, amount: <number of tokens>},
            ...
            {addr: <address of constituent contract>, amount: <number of tokens>},
        ]
    }

**killNewContract**. The `killNewContract` function allows to kill a freshly created contract. It only works if firstly all tokens are owned by the payer, ie none have been distributed (or all have been bought back) and secondly no payments have been registered in the contract (see `payments`). The purpose of this function is solely to clean up unused and/or erroneously created loan contracts. It is not meant for killing contracts after repayment.

The functions below are specific to the portfolio functionalities of the contract.

**lockPortfolio**. The `lockPortfolio` function locks the contract portfolio, meaning loans can no longer be transferred in or out. This function must be called before any token-transfer related functions are called, otherwise they'll fail.

**locked**. The `locked` property returns `False` before the `lockPortfolio` function has been called for the first time, and `True` thereafter.

**transferIn**. The `transferIn(token_address)` function transfers loans into the portfolio. The transfer of loans must have been previously approved in the loan contract using the ERC20 `approve` function; the maximum amount possible--as determined by the ERC20 `allowance` function--is transferred in. This function can only be called up the point where `lockPortfolio` is called for the first time.

**transferOut**. The `transferIn(tokenaddress, destination, number_tokens)` function transfers loans out of the portfolio, calling the loan smart contract's ERC20 `transfer` function. This function can only be called up the point where `lockPortfolio` is called for the first time.

**balance**. The `balance(owner_addr)` returns the amount of Ether held on behalf of the token owned by `owner_addr`, or the entire amount of Ether held in case `owner_addr` is null. Note that when a token is transferred, the undistributed Ether balance _always_ moves together with the token.

**payDividend**. The `payDividend` function--which can only be called by the manager--triggers a disbursement of the Ether held in the contract to the token holders. All dividends are sent to the address of the current owner of the token, regardless of who owned the token when the Ether was received. We need to work on a mechanism that does not lead to bad surprises here, ie we need to avoid paying out dividends just a block before transfers confirm TODO.

[ERC20]: https://theethereum.wiki/w/index.php/ERC20_Token_Standard
[ERC20wp]: https://en.wikipedia.org/wiki/ERC20
