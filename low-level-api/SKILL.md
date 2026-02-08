---
name: low-level-api
description: Interact directly with Orderly Network's low-level REST API (EVM). Explicitly separates Public and Private endpoints with authentication details.
argument-hint: <command> [params]
---

# Low Level API Skill

Use this skill to access the raw REST API. Includes full authentication details.

## Configuration
- **Testnet:** `https://testnet-api-evm.orderly.org`
- **Mainnet:** `https://api-evm.orderly.org`

## Authentication & Signing
Private endpoints require the following headers.

### Required Headers
- `orderly-account-id`: Your unique account ID (e.g., `0x...`).
- `orderly-key`: Your API public key (ed25519).
- `orderly-timestamp`: Current timestamp in milliseconds.
- `orderly-signature`: The request signature.

### Generating `orderly-signature`
1. **Construct the Message:**
   ```
   message = timestamp + method + requestPath + bodyString
   ```
   - `timestamp`: Same as header (e.g., `1680000000000`).
   - `method`: Uppercase (e.g., `POST`, `GET`).
   - `requestPath`: URL path (e.g., `/v1/order`).
   - `bodyString`: JSON string of body parameters (if any). Empty string if none.

2. **Sign:**
   - Use your **API Secret Key** (ed25519) to sign the `message` bytes.

3. **Encode:**
   - Encode the signature in **Base64URL** format.

## Rate Limits
- **General:** 10 requests per second per IP (unless specified otherwise).

## Private Endpoints (Requires Auth)
These endpoints require a valid signature and API key headers.

### `POST` /v1/client/add_sub_account
- **Summary:** Add sub-account
- **Limit:** 10 requests per second
- **Parameters:** None


### `GET` /v1/client/sub_account
- **Summary:** Get sub-account list
- **Limit:** 10 requests per second
- **Parameters:** None


### `POST` /v1/client/update_sub_account
- **Summary:** Update sub-account
- **Limit:** 10 requests per second
- **Parameters:** None


### `GET` /v1/client/aggregate/positions
- **Summary:** Get aggregate positions
- **Limit:** Standard
- **Parameters:** None


### `GET` /v1/client/aggregate/holding
- **Summary:** Get aggregate holding
- **Limit:** Standard
- **Parameters:** None


### `POST` /v1/sub_account_settle_pnl
- **Summary:** Settle sub-account PnL
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly_account_id` | header | string | Yes |


### `GET` /v1/positions
- **Summary:** Get All Positions Info
- **Limit:** 30 requests per 10
- **Parameters:** None


### `GET` /v1/position/{symbol}
- **Summary:** Get One Position Info
- **Limit:** 30 requests per 10
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes |


### `GET` /v1/position_history
- **Summary:** Get Position History
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `limit` | query | number | No |


### `GET` /v1/funding_fee/history
- **Summary:** Get Funding Fee History
- **Limit:** 20 requests per 60
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `start_t` | query | string | No |
  | `end_t` | query | string | No |
  | `page` | query | string | No |
  | `size` | query | string | No |


### `GET` /v1/client/liquidator_liquidations
- **Summary:** Get Liquidated Positions by Liquidator
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `start_t` | query | number | No |
  | `end_t` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/client/leverage
- **Summary:** Get Leverage Setting
- **Limit:** 1 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |


### `POST` /v1/client/leverage
- **Summary:** Update Leverage Setting
- **Limit:** 5 requests per 60
- **Parameters:** None


### `GET` /v1/liquidations
- **Summary:** Get Liquidated Positions of Account
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `start_t` | query | number | No |
  | `end_t` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `sort_by` | query | string | No |
  | `liquidation_id` | query | number | No |


### `POST` /v1/liquidation
- **Summary:** Claim Liquidated Positions
- **Limit:** 5 requests per 1
- **Parameters:** None


### `POST` /v1/order
- **Summary:** Create Order
- **Limit:** 10 requests per 1
- **Parameters:** None


### `PUT` /v1/order
- **Summary:** Edit Order
- **Limit:** Standard
- **Parameters:** None


### `DELETE` /v1/order
- **Summary:** Cancel Order
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `order_id` | query | number | Yes |


### `GET` /v1/order/{order_id}
- **Summary:** Get Order by order_id
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `order_id` | path | string | Yes |


### `GET` /v1/trades
- **Summary:** Get Trades
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `start_t` | query | number | No |
  | `end_t` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/client/holding
- **Summary:** Get Current Holding
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `all` | query | boolean | No |


### `GET` /v1/client/info
- **Summary:** Get Account Information
- **Limit:** 10 requests per 60
- **Parameters:** None


### `GET` /v1/orders
- **Summary:** Get Orders
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `side` | query | string | No |
  | `order_type` | query | string | No |
  | `status` | query | string | No |
  | `order_tag` | query | string | No |
  | `start_t` | query | number | No |
  | `end_t` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `sort_by` | query | string | No |


### `DELETE` /v1/orders
- **Summary:** Cancel All Pending Orders
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |


### `GET` /v1/client/statistics
- **Summary:** Get User Statistics
- **Limit:** 10 requests per 60
- **Parameters:** None


### `GET` /v1/asset/history
- **Summary:** Get Asset History
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `token` | query | string | No |
  | `side` | query | string | No |
  | `status` | query | string | No |
  | `start_t` | query | number | No |
  | `end_t` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `POST` /v1/claim_insurance_fund
- **Summary:** Claim Insurance Fund
- **Limit:** 5 requests per 1
- **Parameters:** None


### `POST` /v1/orderly_key
- **Summary:** Add Orderly Key
- **Limit:** 10 requests per 1
- **Parameters:** None


### `POST` /v1/client/remove_orderly_key
- **Summary:** Remove Orderly Key
- **Limit:** 10 requests per 1
- **Parameters:** None


### `POST` /v1/order/cancel_all_after
- **Summary:** Cancel All After
- **Limit:** Standard
- **Parameters:** None


### `POST` /v1/batch-order
- **Summary:** Batch Create Order
- **Limit:** 10 requests per 1
- **Parameters:** None


### `DELETE` /v1/batch-order
- **Summary:** Batch Cancel Orders
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `order_ids` | query | string | Yes |


### `DELETE` /v1/client/order
- **Summary:** Cancel Order By client_order_id
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `client_order_id` | query | string | Yes |


### `DELETE` /v1/client/batch-order
- **Summary:** Batch Cancel Orders By client_order_id
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `client_order_ids` | query | string | Yes |


### `GET` /v1/client/order/{client_order_id}
- **Summary:** Get Order by client_order_id
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `client_order_id` | path | string | Yes |


### `GET` /v1/orderbook/{symbol}
- **Summary:** Orderbook Snapshot
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes |
  | `max_level` | query | integer | No |


### `GET` /v1/kline
- **Summary:** Get Kline
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `type` | query | string | Yes |
  | `limit` | query | number | No |


### `GET` /v1/order/{order_id}/trades
- **Summary:** Get All Trades of Specific Order
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `order_id` | path | number | Yes |


### `GET` /v1/trade/{trade_id}
- **Summary:** Get Trade
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `trade_id` | path | number | Yes |


### `POST` /v1/client/maintenance_config
- **Summary:** Set Maintenance Config
- **Limit:** 10 requests per 60
- **Parameters:** None


### `GET` /v1/volume/user/stats
- **Summary:** Get User Volume Statistics
- **Limit:** Standard
- **Parameters:** None


### `GET` /v1/volume/user/daily
- **Summary:** Get User Daily Volume
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |


### `GET` /v1/withdraw_nonce
- **Summary:** Get Withdrawal Nonce
- **Limit:** Standard
- **Parameters:** None


### `GET` /v1/settle_nonce
- **Summary:** Get Settle PnL Nonce
- **Limit:** 10 requests per 1
- **Parameters:** None


### `POST` /v1/withdraw_request
- **Summary:** Create Withdraw Request
- **Limit:** 10 requests per 1
- **Parameters:** None


### `POST` /v1/settle_pnl
- **Summary:** Request PnL Settlement
- **Limit:** 1 requests per 1
- **Parameters:** None


### `GET` /v1/pnl_settlement/history
- **Summary:** Get PnL Settlement History
- **Limit:** 20 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_t` | query | integer | No |
  | `end_t` | query | integer | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `POST` /v1/internal_transfer
- **Summary:** Create Internal Transfer
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/transfer_nonce
- **Summary:** Get Transfer Nonce
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/internal_transfer_history
- **Summary:** Get Internal Transfer History
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `status` | query | string | No |
  | `start_t` | query | string | No |
  | `end_t` | query | string | No |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `side` | query | string | Yes |
  | `from_account_id` | query | string | No |
  | `to_account_id` | query | string | No |
  | `main_sub_only` | query | boolean | No |


### `GET` /v1/client/key_info
- **Summary:** Get Current Orderly Key Info
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `key_status` | query | string | No |


### `GET` /v1/client/orderly_key_ip_restriction
- **Summary:** Get Orderly Key IP Restriction
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly_key` | query | string | Yes |


### `POST` /v1/client/set_orderly_key_ip_restriction
- **Summary:** Set Orderly Key IP Restriction
- **Limit:** 10 requests per 60
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly_key` | query | string | Yes |
  | `ip_restriction_list` | query | string | Yes |


### `POST` /v1/client/reset_orderly_key_ip_restriction
- **Summary:** Reset Orderly Key IP Restriction
- **Limit:** 10 requests per 60
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly_key` | query | string | Yes |
  | `reset_mode` | query | string | Yes |


### `GET` /v1/notification/inbox/notifications
- **Summary:** Get All Notifications
- **Limit:** 10 requests per 60
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `type` | query | string | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/notification/inbox/unread
- **Summary:** Get Unread Notifications
- **Limit:** 10 requests per 60
- **Parameters:** None


### `POST` /v1/notification/inbox/mark_read
- **Summary:** Set Read Status of Notifications
- **Limit:** 10 requests per 60
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `flag` | query | number | Yes |
  | `ids` | query | array | Yes |


### `POST` /v1/notification/inbox/mark_read_all
- **Summary:** Set Read Status of All Notifications
- **Limit:** 10 requests per 60
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `flag` | query | number | Yes |


### `POST` /v1/algo/order
- **Summary:** Create Algo Order
- **Limit:** 5 requests per 1
- **Parameters:** None


### `PUT` /v1/algo/order
- **Summary:** Edit Algo Order
- **Limit:** 5 requests per 1
- **Parameters:** None


### `DELETE` /v1/algo/order
- **Summary:** Cancel Algo Order
- **Limit:** 5 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `order_id` | query | number | Yes |


### `DELETE` /v1/algo/client/order
- **Summary:** Cancel Algo Order By client_order_id
- **Limit:** 5 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `client_order_id` | query | string | Yes |


### `GET` /v1/algo/orders
- **Summary:** Get Algo Orders
- **Limit:** 5 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `order_type` | query | string | No |
  | `status` | query | string | No |
  | `order_tag` | query | string | No |
  | `start_t` | query | number | No |
  | `end_t` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `side` | query | string | No |
  | `algo_type` | query | string | No |
  | `is_triggered` | query | string | No |


### `DELETE` /v1/algo/orders
- **Summary:** Cancel All Pending Algo Orders
- **Limit:** 5 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `algo_type` | query | string | No |


### `GET` /v1/algo/order/{order_id}
- **Summary:** Get Algo Order by order_id
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `order_id` | path | string | Yes |


### `GET` /v1/algo/client/order/{client_order_id}
- **Summary:** Get Algo Order by client_order_id
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `client_order_id` | path | string | Yes |


### `GET` /v1/algo/order/{order_id}/trades
- **Summary:** Get All Trades of Specific Algo Order
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `order_id` | path | number | Yes |


### `GET` /v1/volume/broker/daily
- **Summary:** Get Builder's Users' Volumes
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `address` | query | string | No |
  | `order_tag` | query | string | No |
  | `aggregateBy` | query | string | No |
  | `sort` | query | string | No |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |


### `GET` /v1/broker/leaderboard/daily
- **Summary:** Get Builder's Leaderboard
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `order_tag` | query | string | No |
  | `broker_id` | query | string | No |
  | `sort` | query | string | No |
  | `address` | query | string | No |
  | `aggregateBy` | query | string | No |


### `GET` /v1/client/statistics/daily
- **Summary:** Get User Daily Statistics
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `include_historical_data` | query | boolean | No |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |


### `GET` /v1/broker/user_info
- **Summary:** Get User Fee Rates
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `account_id` | query | string | No |
  | `address` | query | string | No |
  | `page` | query | string | No |
  | `size` | query | string | No |


### `POST` /v1/broker/fee_rate/set
- **Summary:** Update User Fee Rate
- **Limit:** 1 requests per second
- **Parameters:** None


### `POST` /v1/broker/fee_rate/set_default
- **Summary:** Reset User Fee Rate
- **Limit:** 1 requests per second
- **Parameters:** None


### `GET` /v1/broker/fee_rate/default
- **Summary:** Get Default Builder Fee
- **Limit:** 1 requests per second
- **Parameters:** None


### `POST` /v1/broker/fee_rate/default
- **Summary:** Update Default Builder Fee
- **Limit:** 1 requests per second
- **Parameters:** None


### `POST` /v1/referral/create
- **Summary:** Create Referral Code
- **Limit:** 1 requests per second
- **Parameters:** None


### `GET` /v1/referral/admin_info
- **Summary:** Get Referral Code Info
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | string | No |
  | `size` | query | string | No |
  | `user_address` | query | string | No |
  | `account_id` | query | string | No |
  | `sort_by` | query | string | No |


### `POST` /v1/referral/update
- **Summary:** Update Referral Code
- **Limit:** 1 requests per second
- **Parameters:** None


### `POST` /v1/referral/bind
- **Summary:** Bind Referral Code
- **Limit:** 1 requests per second
- **Parameters:** None


### `GET` /v1/referral/info
- **Summary:** Get Referral Info
- **Limit:** 10 requests per second
- **Parameters:** None


### `GET` /v1/referral/referral_history
- **Summary:** Get Referral History
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | No |
  | `end_date` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `GET` /v1/referral/rebate_summary
- **Summary:** Get Referral Rebate Summary
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | No |
  | `end_date` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `GET` /v1/referral/referee_history
- **Summary:** Get Referee History
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | No |
  | `end_date` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `GET` /v1/referral/referee_info
- **Summary:** Get Referee Info
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | integer | No |
  | `size` | query | integer | No |
  | `sort` | query | string | No |


### `GET` /v1/referral/referee_rebate_summary
- **Summary:** Get Referee Rebate Summary
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |


### `POST` /v1/referral/edit_split
- **Summary:** Edit Referral Code Split
- **Limit:** 1 requests per second
- **Parameters:** None


### `POST` /v1/referral/auto_referral/update
- **Summary:** Builder admin update auto referral
- **Limit:** 1 requests per second
- **Parameters:** None


### `GET` /v1/referral/auto_referral/info
- **Summary:** Builder admin get auto referral info
- **Limit:** 1 requests per second
- **Parameters:** None


### `GET` /v1/referral/auto_referral/progress
- **Summary:** Get auto referral progress
- **Limit:** 1 requests per second
- **Parameters:** None


### `POST` /v1/referral/edit_referral_code
- **Summary:** Edit Referral Code
- **Limit:** 10 requests per second
- **Parameters:** None


### `GET` /v1/client/points
- **Summary:** Get User's Merits
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/client/points/user_statistics
- **Summary:** Get User Statistics
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `stage` | query | number | Yes |


### `GET` /v1/admin/points/stage
- **Summary:** Get Stage Parameters
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `stage_id` | query | number | No |


### `POST` /v1/admin/points/stage
- **Summary:** Create/Update Stage Parameters
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |


### `POST` /v1/delegate_withdraw_request
- **Summary:** Delegate Signer Withdraw Request
- **Limit:** 1 requests per second
- **Parameters:** None


### `POST` /v1/delegate_settle_pnl
- **Summary:** Delegate Signer Settle PnL
- **Limit:** 1 requests per second
- **Parameters:** None


### `GET` /v1/client/distribution_history
- **Summary:** Get Distribution History
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `status` | query | string | No |
  | `type` | query | string | No |
  | `start_t` | query | string | No |
  | `end_t` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `POST` /v1/client/campaign/sign_up
- **Summary:** Sign Up Campaign
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/staking/balance
- **Summary:** Get Wallet's Current Staked Balance
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/staking/unstake_details
- **Summary:** Get Unstaking ORDER Details
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/staking/valor2/redeem
- **Summary:** Get Valor2 Redeem Info
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/staking/valor/redeem
- **Summary:** Get Valor Redeem Info
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/staking/esorder/vesting_list
- **Summary:** Get esORDER vesting list
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `is_vesting` | query | boolean | Yes |


### `POST` /v1/sv/sp_orderly_key
- **Summary:** Add SP Orderly Key
- **Limit:** 10 requests per second
- **Parameters:** None


### `POST` /v1/sv/sp_settle_pnl
- **Summary:** Request SP PnL Settlement
- **Limit:** Standard
- **Parameters:** None


### `POST` /v1/sv/manual_period_delivery
- **Summary:** Trigger Manual Period Delivery
- **Limit:** Standard
- **Parameters:** None



## Public Endpoints (No Auth Required)
These endpoints can be called without authentication headers.

### `GET` /v1/public/futures
- **Summary:** Get Market Info for All Symbols
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/ip_info
- **Summary:** Get IP Info
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/futures_market
- **Summary:** Get Market Volume by Builder
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `broker_id` | query | string | No |


### `GET` /v1/public/balance/stats
- **Summary:** Get TVL by Builder
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `broker_id` | query | string | No |


### `GET` /v1/public/market_trades
- **Summary:** Get Market Trades
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `limit` | query | integer | No |


### `GET` /v1/public/market_info/price_changes
- **Summary:** Get Price Info for All Symbols
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/market_info/traders_open_interests
- **Summary:** Get Open Interests for All Symbols
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/market_info/history_charts
- **Summary:** Get Historical Price List for All Symbols
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `time_interval` | query | string | No |


### `GET` /v1/public/market_info/funding_history
- **Summary:** Get Funding Rate for All Markets
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/volume/stats
- **Summary:** Get Builder Volume
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `broker_id` | query | string | No |


### `GET` /v1/public/broker/stats
- **Summary:** Get Builder Stats
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `broker_id` | query | string | No |


### `GET` /v1/public/broker/name
- **Summary:** Get Builder List
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `broker_id` | query | string | No |


### `GET` /v1/public/funding_rates
- **Summary:** Get Predicted Funding Rates for All Markets
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/funding_rate/{symbol}
- **Summary:** Get Predicted Funding Rate for One Market
- **Limit:** 30 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes |


### `GET` /v1/public/funding_rate_history
- **Summary:** Get Funding Rate History for One Market
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `start_t` | query | number | No |
  | `end_t` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/public/futures/{symbol}
- **Summary:** Get Market Info for One Symbol
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes |


### `GET` /v1/public/liquidation
- **Summary:** Get Positions Under Liquidation
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_t` | query | number | No |
  | `end_t` | query | number | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `GET` /v1/public/liquidated_positions
- **Summary:** Get Liquidated Positions Info
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `start_t` | query | string | No |
  | `end_t` | query | string | No |
  | `page` | query | string | No |
  | `size` | query | string | No |


### `GET` /v1/public/info/{symbol}
- **Summary:** Get Order Rules per Symbol
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes |


### `GET` /v1/public/token
- **Summary:** Get Supported Collateral Info
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `chain_id` | query | string | No |


### `GET` /v1/public/info
- **Summary:** Get Available Symbols
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/index_price_source
- **Summary:** Get Index Price Source
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/insurancefund
- **Summary:** Get Insurance Fund Info
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/tv/config
- **Summary:** Get TradingView Localized Config Info
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `locale` | query | string | Yes |


### `GET` /v1/tv/history
- **Summary:** Get TradingView History Bars
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `resolution` | query | string | Yes |
  | `from` | query | string | Yes |
  | `to` | query | string | Yes |


### `GET` /v1/tv/symbol_info
- **Summary:** Get TradingView Symbol Info
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `group` | query | string | Yes |


### `GET` /v1/public/leverage
- **Summary:** Get Max Leverage Setting
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/system_info
- **Summary:** Get System Maintenance Status
- **Limit:** 1 requests per 1
- **Parameters:** None


### `GET` /v1/registration_nonce
- **Summary:** Get Registration Nonce
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/get_account
- **Summary:** Check if Wallet is Registered
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `broker_id` | query | string | Yes |
  | `chain_type` | query | string | No |


### `GET` /v1/get_all_accounts
- **Summary:** Check Account Details of an Address
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `broker_id` | query | string | Yes |
  | `chain_type` | query | string | No |


### `GET` /v1/get_broker
- **Summary:** Check if Address is Registered
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `chain_type` | query | string | No |


### `POST` /v1/register_account
- **Summary:** Register Account
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/get_orderly_key
- **Summary:** Get Orderly Key
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `account_id` | query | string | Yes |
  | `orderly_key` | query | string | Yes |


### `GET` /v1/public/config
- **Summary:** Get Leverage Configuration
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/chain_info
- **Summary:** Get Supported Chains per Builder
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `broker_id` | query | string | No |


### `GET` /v1/tv/kline_history
- **Summary:** Get Kline History
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `resolution` | query | string | Yes |
  | `from` | query | string | No |
  | `to` | query | string | No |
  | `limit` | query | number | No |


### `GET` /v1/public/vault_balance
- **Summary:** Get Vault Balance
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `chain_id` | query | string | No |
  | `token` | query | string | No |


### `POST` /v1/faucet/usdc
- **Summary:** Get Faucet USDC(Testnet Only)
- **Limit:** Standard
- **Parameters:** None


### `GET` /v1/public/account
- **Summary:** Check if Account Exists
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `account_id` | query | string | No |


### `GET` /v1/public/announcement
- **Summary:** Get Announcements
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/referral/check_ref_code
- **Summary:** Check Referral Code
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `account_id` | query | string | Yes |


### `GET` /v1/public/referral/verify_ref_code
- **Summary:** Verify Referral Code
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `referral_code` | query | string | Yes |


### `GET` /v1/public/points/leaderboard
- **Summary:** Get Merits Leaderboard
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_r` | query | integer | No |
  | `end_r` | query | integer | No |
  | `epoch_id` | query | integer | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `GET` /v1/public/points/epoch_dates
- **Summary:** Get Start and End Date of All Epochs
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/points/rankings
- **Summary:** Get Stage Rankings
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `stage` | query | number | Yes |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `period` | query | string | Yes |


### `GET` /v1/public/points/epoch
- **Summary:** Get Number of Merits for Distribution
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/points/stages
- **Summary:** Get Information About Stages
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `stage_id` | query | number | No |
  | `status` | query | string | No |
  | `broker_id` | query | string | Yes |


### `DELETE` /v1/admin/points/stage
- **Summary:** Delete Stage
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `orderly` | header | string | Yes |
  | `stage_id` | query | number | Yes |


### `POST` /v1/delegate_signer
- **Summary:** Delegate Signer
- **Limit:** 1 requests per second
- **Parameters:** None


### `POST` /v1/delegate_orderly_key
- **Summary:** Add Delegate Signer Orderly Key
- **Limit:** 1 requests per second
- **Parameters:** None


### `GET` /v1/public/campaigns
- **Summary:** Get List of Campaigns
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_t` | query | integer | No |
  | `end_t` | query | integer | No |
  | `only_show_alive` | query | string | No |
  | `address` | query | string | No |


### `GET` /v1/public/campaign/stats
- **Summary:** Get Campaign Statistics
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | integer | Yes |
  | `broker_id` | query | string | No |
  | `symbol` | query | string | No |


### `GET` /v1/public/campaign/stats/details
- **Summary:** Get Detailed Campaign Info
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | integer | Yes |
  | `broker_id` | query | string | No |
  | `symbols` | query | string | No |
  | `group_by` | query | string | No |


### `GET` /v1/public/campaign/ranking
- **Summary:** Get Campaign Ranking
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | integer | Yes |
  | `broker_id` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |
  | `sort_by` | query | string | No |
  | `min_pnl` | query | integer | No |
  | `min_volume` | query | integer | No |
  | `aggregate_by` | query | string | No |


### `GET` /v1/public/campaign/user
- **Summary:** Get Campaign User Info
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | integer | Yes |
  | `account_id` | query | string | No |
  | `address` | query | string | No |
  | `broker_id` | query | string | No |
  | `symbol` | query | array | No |
  | `order_tag` | query | string | No |


### `GET` /v1/public/campaign/check
- **Summary:** Get Campaign Verification
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | number | Yes |
  | `type` | query | string | Yes |
  | `address` | query | string | Yes |
  | `lower_boundary` | query | number | Yes |
  | `cmp` | query | string | No |


### `GET` /v1/staking/overview
- **Summary:** Get Staking Overview
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/staking/valor/batch_info
- **Summary:** Get Valor Batch Info
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/staking/valor/pool_info
- **Summary:** Get Valor Pool Info
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/staking/valor2/pool_info
- **Summary:** Get Valor2 Pool Info
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/staking/valor2/batch_info
- **Summary:** Get Valor2 Batch Info
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/staking/valor2/revenue_buyback
- **Summary:** Get Valor2 Revenue Buyback
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/trading_rewards/epoch_info
- **Summary:** Get Parameters of Each Epoch for All Epochs
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/trading_rewards/epoch_data
- **Summary:** Get Epochs Data
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/trading_rewards/broker_allocation_history
- **Summary:** Get Broker Allocation History
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/trading_rewards/wallet_rewards_history
- **Summary:** Get Wallet Trading Rewards History
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `chain_type` | query | string | No |


### `GET` /v1/public/trading_rewards/account_rewards_history
- **Summary:** Get Account Trading Rewards History
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `chain_type` | query | string | No |


### `GET` /v1/public/trading_rewards/status
- **Summary:** Get the Status of Trading Rewards Programme
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/trading_rewards/symbol_category
- **Summary:** Get Symbol Rewards Category
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/market_making_rewards/epoch_info
- **Summary:** Get Parameters of Each Epoch
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/market_making_rewards/status
- **Summary:** Get the Status of Market Making Rewards Programme
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/market_making_rewards/symbol_params
- **Summary:** Get Symbol Rewards Parameters
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/market_making_rewards/leaderboard
- **Summary:** Leaderboard for market maker rewards
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `epoch` | query | number | Yes |
  | `market` | query | string | No |


### `GET` /v1/public/market_making_rewards/current_epoch_estimate
- **Summary:** Get Market Maker Current Epoch Estimate
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `chain_type` | query | string | No |


### `GET` /v1/public/trading_rewards/current_epoch_broker_estimate
- **Summary:** Get Current Epoch Estimate by Broker
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/public/trading_rewards/current_epoch_estimate
- **Summary:** Get Current Epoch Estimate
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/public/market_making_rewards/group_rewards_history
- **Summary:** Get Wallet Group Market Making Rewards History
- **Limit:** 10 requests per 1
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `symbol` | query | string | No |


### `GET` /v1/public/sv_nonce
- **Summary:** Get Strategy Vault Nonce for Account Transaction
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/account_sv_transaction_history
- **Summary:** Get Account’s Strategy Vault Transaction history
- **Limit:** Standard
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `vault_id` | query | string | No |
  | `start_t` | query | string | No |
  | `end_t` | query | string | No |
  | `page` | query | string | No |
  | `size` | query | string | No |
  | `type` | query | string | No |


### `POST` /v1/sv_operation_request
- **Summary:** Create Strategy Vault Deposit/Withdrawal Request with Account
- **Limit:** 10 requests per 1
- **Parameters:** None


### `GET` /v1/sv/venue_transfer_history
- **Summary:** Get Fund Inflow Allocation
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `period_number` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/sv/venue_withdrawal_history
- **Summary:** Get Fund Outflow Allocation
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `period_number` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/sv/protocol_revenue_share_history
- **Summary:** Get Orderly Protocol Revenue Sharing History
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/sv/liquidation_fees_share_history
- **Summary:** Get Liquidation Fees Sharing History
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/sv/internal_transfer_history
- **Summary:** Get Internal Transfer History
- **Limit:** 10 requests per second
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | number | No |
  | `size` | query | number | No |


