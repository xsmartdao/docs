# Zillending Protocol

## Oracle Mechanism

1. oracle server is the verifier of the contracts.
2. oracle server uses schnorr algorithm to sign the hash of the message.
3. contract side use oracle server's public key to verify the hash of the message, further, extra parameters from the message.


```
procedure verifySignature(msg: String, sign: ByStr64)
  data_to_verify = builtin sha256hash msg;
  pubk <- verifier;
  e = {_eventname: "RawDataToVerifySignature"; data_to_verify: data_to_verify; sign: sign; pubk: pubk};
  event e;
  data = builtin to_bystr data_to_verify;
  is_valid = builtin schnorr_verify pubk data sign;
  match is_valid with
    | True =>
      e2 = {_eventname: "SignatureValid"};
      event e2
    | False =>
      err = InvalidSignature;
      ThrowError err
  end
end
```

## Callback Mechanism

1. Issue two messages, one for remote call, one for self call.
2. Complte the logic in the self-called transition.

```
transition MintOnDepositInit(account: ByStr20, amount: Uint128, this_timestamp: Uint256)
  IsCoreContract _sender;
  reserve <- underlying_asset_address;
  msg_to_core = {_tag: "set_reserve_normalized_income"; _recipient: _sender; _amount: zero;
                    reserve: reserve; this_timestamp: this_timestamp };
  msg_to_self = {_tag: "MintOnDepositComplete"; _recipient:_this_address; _amount: zero;
                    account: account; amount: amount; this_timestamp: this_timestamp };
  msgs = two_msgs msg_to_core msg_to_self;
  send msgs
end

transition MintOnDepositComplete(account: ByStr20, amount: Uint128, this_timestamp: Uint256)
  IsSelf _sender;
  core_reader <- core_contract_reader;
  reserve_normalized_income_temp <- &core_reader.reserve_normalized_income_temp;
  ...
end

```

## Formal Definitions

| Name | Description |
| ---------------| ------------------|
| `current_timestamp` | current numbers of seconds |
| `last_updated_timestamp` | Timestamp of the last update of the reserve data. It is updated every time a borrow, deposit, redeem, repay, swap, or liquidation event occurs
| `delta_time` | `current_timestamp - last_updated_timestamp`
| `year_of_seconds` | Number of seconds in a year: 31536000 |
| `yearly_period` | `delta_time / year_of_seconds` |
| `total_liquidity` | Total amount of liquidity available in the reserve. The decimals of the this value depend on the decimals of the currency. (unconfirmed comments: the balance of this protocol on the reserve) |
| `total_stable_borrows` | Total amount of liquidity borrowed at a stable rate. The decimals of this value depend on the decimals of the currency |
| `total_variable_borrows` | Total amount of liquidity borrowed at a variable rate. The decimals of this value depend on the decimals of the currency |
| `total_borrows` | Total amount of liquidity borrowed. The decimals of this value depend on the decimals of the currency. `total_borrows = total_stable_borrows + total_variable_borrows` |
| `utilization_rate` | Representing the utilization of the deposited funds `utilization = total_borrows / total_liquidity`|
| `target_utilization_rate` | The utilization rate targeted by the model, beyond the variable interest rate rises sharply |
| `base_variable_borrow_rate` | Constant for `total_borrows == 0`. Expression in ray |
| `variable_rate_slope1` | Slope of the variable interest curve when `utilization_rate >` 0 and <= `optimal_utilization_rate`. Expressed in ray |
| `variable_rate_slope2` | Slope of the variable interest curve when `utilization_rate > optimal_utilization_rate`. Expressed in ray |
| `stable_rate_slope1` | Slope of the stable interest curve when `utilization_rate > 0 and <= optimal_utilization_rate`. Expressed in ray |
| `stable_rate_slope2` | Slope of the stable interest curve when `utilization_rate > optimal_utilization_rate`. Expressed in ray |
| `variable_borrow_rate` | |
| `average_stable_borrow_rate` | |
| `overall_borrow_rate` | Overall borrow rate of the reserve, calculated as the weighted average between the `total_borrows_stable` and the `total_borrows_variable`. `(total_variable_borrows * variable_borrow_rate + total_stable_borrows * average_stable_borrow_rate) / total_borrows` |



## Core Contract

### User transitions

Transitions that called by users via UI.

| Name | Type | Description |
| ---------------| ----------|---------|
| `deposit`         | `reverse: ByStr20, amount: Uint128` | deposit the underlying asset into the reserve |
| `borrow` | | |
| `redeem` | | |
| `repay` | | |


### Admin transitions

Transitions that called by admin-cli.

| Name | Type | Description |
| ---------------| ----------|---------|
| `set_configurator`         | `configurator: ByStr20` | changes configurator address |
| `init_reserve`         | `reserve: ByStr20, underlying_asset_decimals: Uint256, interest_rate_strategy_address: ByStr20, aToken_address: ByStr20, reserve_name: String, reserve_symbol: String` | initializes a reserve |
| `set_reserve_interest_rate_strategy_address` | `reserve: ByStr20, rate_strategy_address: ByStr20` | updates the address of the interest rate strategy contract |
| `enable_borrowing_on_reverse` | `reserve: ByStr20, stable_borrow_rate_enabled: Bool` | enables borrowing on a reserve. Also sets the stable rate borrowing |
| `disable_borrowing_on_reverse` | `reserve: ByStr20` | disables borrowing on a reserve |


## AToken Contract

### Internal Functions

*We might need some temporary fields for internal communication, try not use it, but sometimes we have to*

| Name | Type | Description |
| ---------------| ----------|---------|
| `calculate_cumulated_balance_internal`         | `user: ByStr20, balance: Uint256, reserve_normalized_income: Uint256` | calculate the interest accrued by user on a specific balance. `reserve_normalized_income` should be calculated by core contract, but we now need use oracle server as the lacking of the external library feature |
| `balance_of` | `user: ByStr20, reserve_normalized_income: Uint256` | calculate the balance of the user, whihc is the principal balance + interest generated by the principal balance + interest generated by the redirected balance |


### Transitions

| Name | Type | Description |
| ---------------| ----------|---------|
| `mint_on_deposit`         | `account: ByStr20, amount: Uint256, reserve_normalized_income: Uint256` | Only can be called by core contract. Mints token in the event of the users depositing the underlying asset into the lending pool |

## Stragety Contract

We deploy one strategy contract for every reserve added. The stragety (see following) fields are fetched by core contract via remote read.

### Fields

| Name | Type | Description |
| ---------------| ----------|---------|
| `base_variable_borrow_rate`         | `Uint256` | base variable borrow rate when Utlization rate = 0. Expressed in ray. |
| `variable_rate_slope1`         | `Uint256` | slope of the variable interest curve when utilization rate > 0 and <= optimal_utilization_rate. Expressed in ray. |
| `variable_rate_slope2`         | `Uint256` | slope of the variable interest curve when utilization rate > optimal_utilization_rate. Expressed in ray. |
| `stable_rate_slope1`         | `Uint256` | slope of the stable interest curve when utilization rate > 0 and <= optimal_utilization_rate. Expressed in ray. |
| `stable_rate_slope2`         | `Uint256` | slope of the stable interest curve when utilization rate > optimal_utilization_rate. Expressed in ray. |


## Formula

### Constants

```
let wad = Uint256 1000000000000000000
let half_wad = Uint256 500000000000000000
let ray = Uint256 1000000000000000000000000000
let half_ray = Uint256 500000000000000000000000000
let wad_ray_ratio = Uint256 1000000000
```

### Linear Interest

```
Parameters: rate, last_update_timestamp

1. time_difference = current_timestamp - last_update_timestamp
2. time_delta = time_difference.way_to_ray().ray_div(seconds_per_year.wad_to_ray())
3. rate.ray_mul(time_delta).add(ray)
```

### Compounded Interest

```
Parameters: rate, last_update_timestamp

1. time_difference = current_timestamp - last-update_timestamp
2. rate_per_second = rate.div(seconds_per_year)
3. rate_per_second.add(ray).ray_pow(time_difference)
```