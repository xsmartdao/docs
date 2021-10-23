# Zillending Protocol

## Core Contract

### User transitions

Transitions that called by users via UI.

| Name | Type | Description |
| ---------------| ----------|---------|
| `deposit`         | `reverse: ByStr20, amount: Uint128` | deposit the underlying asset into the reserve. |


### Admin transitions

Transitions that called by admin-cli.

| Name | Type | Description |
| ---------------| ----------|---------|
| `set_configurator`         | `configurator: ByStr20` | change configurator address. |


## AToken Contract

### Internal Functions

| Name | Type | Description |
| ---------------| ----------|---------|
| `calculate_cumulated_balance_internal`         | `user: ByStr20, balance: Uint256, reserve_normalized_income: Uint256` | calculate the interest accrued by user on a specific balance. `reserve_normalized_income` shoule be calculated by core contract, but we now need use oracle server as the lacking of the external library feature. |

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