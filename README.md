# lrc-20

This is only an experiment. Assume all lrc-20s will have no value.

## Idea

The idea is to create a locking-native fungible token protocol in the spirit of brc-20.

It uses [1sat ordinals](https://docs.1satordinals.com/) for token ownership and [locks](https://github.com/shruggr/lock) to mint, and locks also for tickers and signalling generally.

There are 3 kinds of transactions:

- *Deploy* - Define a new lrc-20
- *Mint* - Create new tokens of a deployed lrc-20
- *Transfer* - Reassign lrc-20 tokens to another user

## Transactions

Transactions in lrc-20 use 1sat inscriptions to denote their operation.

Inscriptions follow the [1sat ordinals spec](https://docs.1satordinals.com/). Typically they will be `1SAT_P2PKH` scripts:

```
OP_DUP OP_HASH160 <pubkeyhash> OP_EQUALVERIFY OP_CHECKSIG
OP_FALSE OP_IF "ord" OP_1 <content-type> OP_0 <data> OP_ENDIF
```

The content-type is `application/json`. The data is a [valid JSON object](https://www.json.org/json-en.html). 

In addition, mint transactions are required to have at least one [sat lockup script](https://github.com/shruggr/lockup). Mints must lock at least the amount defined in deploy to be valid. The order of outputs does not matter.

Combinations of deploys and transfers in a transaction are ok. If one lrc-20 operation is invalid, it does not invalidate the others. Mints however must always be in their own transaction.

#### Notes

* Json is encoded in utf-8
* Json spacing and the order of entries do not matter
* Strings are not case-sensitive. (HODL = hodl)
* Spacing within strings matters. ("lrc-20" != "lrc - 20 ")
* The user may add custom fields in addition to those required
* If an operation is invalid, its tokens are burned

## Operations

### Deploy

Define a new lrc-20.

[Example transaction](https://whatsonchain.com/tx/bfd3bfe2d65a131e9792ee04a2da9594d9dc8741a7ab362c11945bfc368d2063)

```json
{
    "p": "lrc-20",
    "op": "deploy",
    "tick": "hodl",
    "max": 21000000,
    "lim": 10000,
    "sats": 100000000,
    "blocks": 21000
}
```

| Key    | Required | JSON type  | Description                                                                   |
| ------ | -------- | ---------- | ----------------------------------------------------------------------------- |
| p      | yes      | string     | Protocol: `"lrc-20"`                                                          |
| op     | yes      | string     | Operation: `"deploy"`                                                         |
| tick   | yes      | string     | Ticker: 4-letter identifier of the lrc-20                                     |
| max    | yes      | number     | Max supply: Total number of tokens to be minted                               |
| dec    | no       | number     | Decimals: Decimal shift for display. Defaults to 0.                           |
| lim    | yes      | number     | Mint limit: Limit on tokens minted per user mint operation                    |
| sats   | yes      | number     | Lock amount: Minimum amount of satoshis locked to be a valid mint             |
| blocks | yes      | number     | Lock blocks: Minimum number of blocks locked to be a valid mint  		     |

#### Notes

* The ticker is always 4 bytes encoded as utf-8
* `max`, `dec`, `limit`, `sats`, and `blocks` must all parse to non-negative integers
* Compared to bsv-20, `max`, `dec`, and `limit` are numbers, not strings, reducing ambiguity
* Maximum supply cannot exceed 2^64 - 1
* Number of decimals cannot exceed 18
* Some JSON implementations like JavaScript's `JSON.parse` lose precision above 2^53. Using a library is recommended.

### Mint

Mint a deployed lrc-20.

[Example transaction](https://whatsonchain.com/tx/de3f1f7ddce0a6e79f7fa3d576681ccf506bcb0556cdc2cf5e69553a1de6a4f3)

**Important: Spending mint inscriptions without proper transfer inscriptions will result in a burn**

```json
{
    "p": "lrc-20",
    "op": "mint",
    "id": "3b313338fa0555aebeaf91d8db1ffebd74773c67c8ad5181ff3d3f51e21e0000_1",
    "amt": 10000
}
```

| Key    | Required | JSON type  | Description                                                                   |
| ------ | -------- | ---------- | ----------------------------------------------------------------------------- |
| p      | yes      | string     | Protocol: `"lrc-20"`                                                          |
| op     | yes      | string     | Operation: `"mint"`                                                           |
| id     | yes      | string     | Deploy id: `<txid>_<vout>` of the deploy inscription. vout is 0-indexed.      |
| amt    | yes      | number     | Mint amount: Number of tokens to mint                                         |

#### Notes

* Only one mint is allowed per tx otherwise all mints are invalid
* `amt` must parse to a non-negative integer
* Compared to bsv-20, `amt` is always a number, not a string
* If `amt` is more than the deploy `lim`, the operation is invalid
* The first mint to exceed the maximum supply will receive the fraction that is valid. (ex. 21,000,000 maximum supply, 20,999,242 circulating supply, and 1000 mint inscription = 758 received)
* If two mints occur in the same block, prioritization is assigned via order they were confirmed in the block. (first to last).
* The number of blocks locked can only be determined once the tx is in a block. It is recommended users lock up for slightly longer than required in case the tx is not immediately confirmed.

### Transfer

Transfer a deployed lrc-20.

```json
{
    "p": "lrc-20",
    "op": "transfer",
    "id": "3b313338fa0555aebeaf91d8db1ffebd74773c67c8ad5181ff3d3f51e21e0000_1",
    "amt": 10000
}
```

| Key    | Required | JSON type  | Description                                                                   |
| ------ | -------- | ---------- | ----------------------------------------------------------------------------- |
| p      | yes      | string     | Protocol: `"lrc-20"`                                                          |
| op     | yes      | string     | Operation: `"transfer"`                                                       |
| id     | yes      | string     | Deploy id: `<txid>_<vout>` of the deploy inscription. vout is 0-indexed.      |
| amt    | yes      | number     | Transfer amount: Number of tokens to send                                     |

#### Notes

* Tokens transfer like bitcoins from inputs to outputs
* `amt` must parse to a non-negative integer
* Compared to bsv-20, `amt` is always a number, not a string
* If `amt` is more than the remaining input tokens, that operation is invalid

#### Implied Transfers

Implied transfers are transactions that move tokens without an inscription. Under ordinal theory, this is allowed since an inscription is linked to its ordinal. The new output must be a 1sat ordinal. The moved inscription is invalid if it is consumed at all as part of an earlier transfer inscription.

## Determining tickers

The unambiguous identifier for an lrc-20 is its deploy `txid` and `vout`, not its ticker. Ticker duplicates are allowed by design. If an lrc-20 has an unpopular deploy config and nobody mints it, another lrc-20 should be able to take its ticker.

Ultimately wallets and exchanges will assign tickers, but it is strongly recommended they use locks to determine which deploy is the official one. The lrc-20 with the most total [vibes](https://www.hodlocker.com/bitcoiner/post/43d64ec68724b9c53732af427210127d8fbfa040e3940b3a4d8311b2db8a8df7) across all transactions (ideally including social posts, likes, and transfers) should get the ticker.

Acquiring and retaining tickers has a game theory of its own. It encourage larger locks for mints and non-minimal locks on transfers. It also makes established tickers more valuable since they are unlikely to be successfully challenged. Challenging an lrc-20 for its ticker could start a locking arms race between communities which would be exciting.
