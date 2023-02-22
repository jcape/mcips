- Feature Name: token_metadata_document
- Start Date: 2022-12-02
- MCIP PR: [mobilecoinfoundation/mcips#0059](https://github.com/mobilecoinfoundation/mcips/pull/0059)
- Tracking Issue: [mobilecoinfoundation/mobilecoin#3049](https://github.com/mobilecoinfoundation/mobilecoin/issues/3049)

# Summary
[summary]: #summary

This MCIP specifies a means for clients to dynamically discover metadata about token IDs
on the MobileCoin blockchain, so that they can then display the balance of a new token to customers
in an appropriate way.

This proposal differs from an earlier proposal [MCIP 40](https://github.com/mobilecoinfoundation/mcips/pull/0040), in that it is a stop-gap mechanism which
does not post metadata to the blockchain.

This proposal specifies how clients should dynamically discover token metadata, authenticate this data,
and how they can go on to use this data.

# Motivation
[motivation]: #motivation

When a client gets a `TxOut`, at the lowest level, both the `value` and `token_id` are `u64`
numbers which can be decrypted from the blockchain records. However, it isn't appropriate to
display the `token_id` number directly to the users, since they won't know what this means.

Metadata about a `token_id` consists of all the data which helps clients figure out how they
can appropriately represent value to the users. Tampering with this metadata can represent an attack
to confuse or steal from the users, so there is considerable interest in ensuring that there
is a robust way for clients to authenticate the metadata.

If we do not support dynamic discovery of token metadata, then it must always be baked into clients,
which is what we do today.

Because some clients have relatively long upgrade times, this means that new
tokens that are launched still cannot actually be used by users. This effectively hurts the
time-to-market on any new token that is created.

There are second order effects to this -- if tokens become "available" in clients at different
points in time, then users can get bitten in a bad way. For instance, if a Moby user sent eUSD
to a Signal user, and at the time, Moby supports eUSD but Signal does not, then the recipient
cannot see the payment in their app, or send it anywhere, or send it back to the Moby user.
This could lead to disputed payments, bad user experiences, etc.

At the same time, putting token metadata on chain will require an enclave upgrade, while creating
new tokens on the chain does not. So there is considerable interest in having an off-chain source
of token metadata that can be validated by clients.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This proposal creates a source of truth for token metadata for tokens on the MobileCoin blockchain.

Two documents are hosted by (or on behalf of) the MobileCoin Foundation:

* `https://config.mobilecoin.foundation/token_metadata.json`: A JSON document containing token metadata
* `https://config.mobilecoin.foundation/token_metadata.sig`: An ed25519 signature over the bytes of the .json document. This is binary data.

The ed25519 signing key is controlled by the MobileCoin foundation.
We propose that the `MINTING_TRUST_ROOT` key is reused for this purpose.

At time of writing the `MINTING_TRUST_ROOT` ed25519 public key (in DER encoding) is:

```
-----BEGIN PUBLIC KEY-----
MCowBQYDK2VwAyEA6rqMXns4wNN+W16Eblsue+gqeXlW5C5WhN3MGCc1Ntw=
-----END PUBLIC KEY-----
```

The `token_metadata.json` document has the following schema (example):

```
{
    "schema_version": 1,
    "signing_timestamp": "1670377814",
    "metadata": [
        {
            "token_id": "0",
            "currency_name": "MobileCoin",
            "short_code": "MOB",
            "decimals": 12,
            "suggested_precision": 4,
            "logo_svg_url": "https://mobilecoin.foundation/mob-logo.svg",
            "logo_svg_blake2b_digest": "044F4A75DC1147629",
            "info_url": "https://mobilecoin.foundation"
        },
        {
            "token_id": "1",
            "currency_name": "Electronic Dollar",
            "short_code": "eUSD",
            "symbol": "$",
            "decimals": 6,
            "suggested_precision": 2,
            "logo_svg_url": "https://www.mobilecoin.com/eusd-logo.svg",
            "logo_svg_blake2b_digest": "0123FFFAB12344567",
            "info_url": "https://mobilecoin.com/blog/mobilecoin-launches-eusd"
        },
        {
            "token_id": "14",
            "currency_name": "Meowblecoin",
            "short_code": "MEOW",
            "decimals": 12,
            "suggested_precision": 4,
            "logo_svg_url": "https://www.meowblecoin.com/meowb-logo.svg",
            "logo_svg_blake2b_digest": "F8876CBD11189422B",
            "info_url": "https://www.meowblecoin.com"
        }
    ]
}
```

This metadata list is conceptually a map from `token_id` integers to the metadata
about the token of that id.

For example:

* A `TxOut` with token id of 0 and a (u64) value of `1.5 * 10^12`, may be displayed as `1.5 MOB` to the user, because the token id corresponds to MOB, and the number of decimals of MOB is 12.
* A `TxOut` with a token id of 1 and a  (u64) value of `5 * 10^8` may be displayed as `500 eUSD` to the user, because the token id corresponds to eUSD, and the number of decimals is 6.
* That `TxOut` might also be displayed as `$500` because the symbol of `eUSD` is specified as `$`.

**Note**: In some locales, the actual presentation of currency amounts is different from this example. For example, per [Microsoft's localization guide](https://learn.microsoft.com/en-us/globalization/locale/currency-formatting):

| **Description**                                                    | **Country/Region** | Formatting |
| ------------------------------------------------------------------ | ------------------ | ---------- |
| The negative sign before both the currency symbol and the number   | UK                 | -£127.54   |
|                                                                    | France             | -127,54 €  |
| The negative sign before the number but behind the currency symbol | Denmark            | kr-127,54  |
| The negative sign after the number                                 | Netherlands        | € 127,54-  |
| Enclosed in parentheses.                                           | US                 | ($127.54)  |

Most currencies use the same decimal and thousands separator that other numbers in the locale use, but this is not always true. In some places in Switzerland, they use the period as a decimal separator for Swiss francs (Sfr. 127.54), but then use commas as the decimal separator everywhere else (127,54).

We take the point of view that these differences are not "token metadata", which are determined by the ID of the token that we are using, but rather localization properties determined by the locale of the user. So, for example the same app might be capable of displaying `kr 127,54` and `127.54 kr`, depending on the locale of the user, and token metadata does not influence this.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The token-metadata json fields have the following semantics:

* `schema_version`: An integer describing the version number of the schema of this document. This is 1 until a future MCIP proposes schema modifications.
* `signing_timestamp`: A string representation of the unix timestamp at the (approximate) time of creating the document's signature. This is an integer number of seconds since the unix epoch.
* `token_id`: A decimal string representation of the `u64` corresponding to the token id.
* `currency_name`: A UTF-8 string which is the full, official name of the currency. This is intended to be an English language noun or noun phrase.
* `short_code`: A short code for the currency, up to 12 ASCII characters matching `/[A-Za-z0-9.-]{1,12}/`. This may be displayed to users in a manner similar to ISO 4217 currency codes, for example, USD, GBP, CAD, in connection to an amount of currency. This is expected to match the ticker symbol used for this asset on cryptocurrency exchanges.
* `symbol`: Optional UTF-8 string. Most fiat currencies have a printable character such as $, £, ¥. Some cryptocurrencies do also. Bitcoin has ₿. Ethereum has Ξ.
* `decimals`: An integer specifying how the `u64` integer in a TxOut in the blockchain is scaled to compute a user-displayed amount of the currency. For example, MOB has 12 decimals, which indicates that `10^12` of the smallest representable units of MOB on the MobileCoin blockchain are equal to one MOB.
* `suggested_precision`: An integer specifying how many digits after the decimal point it is suggested to display when displaying an amount of this currency. For MOB the suggested precision is four digits, so if the user has a precise amount of `1.00449 MOB`, it is suggested to round this to `1.0045 MOB` when displaying it. For `eUSD` it is suggested to round the nearest hundredth instead, and this is indicated by the `suggested_precision` of two. The `suggested_precision` could also be a negative number. If for example the `suggested_precision` is `-2`, then amounts are suggested to be rounded to the nearest 100 units. Clients are free to ignore the `suggested_precision` if they so choose.
* `logo_svg_url`: An optional link to a logo image for the currency. This link produces an SVG document. This is expected to have been sanitized using something like [svg-hush](https://github.com/cloudflare/svg-hush), to remove scripting, hyperlinks to other documents, and references to cross-origin resources. The full extent of such sanitization will not be specified here.
* `logo_svg_blake2b_digest`: This is required if `logo_svg_url` is present. This field is a hex-encoding of the 64-byte blake2b digest of the logo svg document. This permits validation of the logo svg after it is downloaded. The blake2b digest is computed against the raw bytes of the entire svg document. This is the same as the bytes of the HTTP body of the response to a GET request that fetches the image, or the blake2b digest of the bytes of the SVG file as it exists on disk.
* `info_url`: A link to a website containing more information about the token that may be interesting to token holders. This should have basic information about the purpose of the token, its supply, any utility that it has, or links to associated whitepapers.

Clients MUST download and validate the `token_metadata.sig` against the bytes of the `token_metadata.json` and the `MINTING_TRUST_ROOT` public key before attempting to process the `token_metadata.json`. Clients MUST reject the json with an error if the signature checking fails.

Clients SHOULD include a validated `token_metadata.json` baked-in to the app as part of their build, and also at run-time try to fetch a new `token_metadata.json`.
Clients then MUST reject any `token_metadata.json` which has a `signing_timestamp` less than the baked-in `signing_timestamp`, and fallback to the baked-in `token_metadata.json`. This prevents replay attacks where an old `token_metadata.json` document is substituted for the latest one with the goal of confusing the users by making their tokens display differently or not at all. Clients can fetch the latest `token_metadata.json` at build time / as part of their release process.

Clients MAY continue to bake token metadata into their build at their discretion, for instance, in order to support offline wallets or other cases where the client cannot access the internet. When both baked-in metadata and the `token_metadata.json` are available, the validated `token_metadata.json` should be considered a more reliable source of truth.

Clients SHOULD follow all `logo_svg_url` links and download and cache the images when they get the token metadata document.
Clients MUST check the `logo_svg_blake2b_digest` for any svg document that they download this way, and reject it if the hash doesn't match.

Maintainers of the `token_metadata.json` SHOULD NOT:

* Allow duplicate records for a given token id (within a given version of `token_metadata.json`)
* Delete a token id's record
* Modify data such as the `short_code` or `decimals` of a token, which may confuse users, and particularly, any exchanges that use this as a source of truth.
* There may be legitimate reasons to make changes to the `currency_name` or `logo_svg`, but this needs to be carefully considered.
* Allow two token ids with `currency_name`'s which could be confused
* Allow the same `short_code` to appear twice, or, allowing two short codes which differ only in case, or the replacement of letters with similar numbers.
* Sign a new version of the `token_metadata.json` with a signing timestamp that is less or equal to the previous version.

# Drawbacks
[drawbacks]: #drawbacks

This proposal has a few drawbacks relative to [MCIP 40](https://github.com/mobilecoinfoundation/mcips/pull/0040):

* MCIP 40 puts the token metadata on chain. This makes it possible to enforce protocol level rules around how the document can evolve. It also ensures the availability of the document.
* It also would take longer to develop since it would require an enclave upgrade.

This proposal's authentication scheme is a single ed25519 signature.

* As an alternative, we could have a signature chain, such as an x509. This allows for more sophisticated strategies around the keeping the key secure.
* As an alternative, we could have a k-of-n ed25519 multisignature. This would allow for the signing authority to be shared across several parties who can sign asynchronously.

Since this signature can easily be performed off-line anyways, and the document isn't expected to change often, we think a single ed25519 signature is fine for this stop-gap proposal.

Having a single document will not scale well to e.g. millions of tokens, and we expect to transition to on-chain metadata before that becomes a problem.

This proposal does not envision having different token registries per network. It is believed that this complexity is unnecessary.
However, it means that just as clients need to hit Intel's IAS services as part of testing (in prod and dev), they will have to hit
the `mobilecoin.com` URLs as well, so this is another way in which testing a deployed network will not be completely isolated to the servers deployed as part of the network.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We chose JSON instead of protobuf to make it easier for clients (which may be web browsers) to directly consume this document in e.g. Javascript.

We chose to separate the signature from the document to avoid the complexity of storing serialized json within json.
This also allows that we can modify the signature scheme in the future while maintaining backwards compatibility, as long as we continue to serve both the old signature and the new signature,
at different URLs.

We chose not to attempt to standardize the localization of any cryptocurrency names, since it does not seem conventional to do so in cryptocurrency wallets.

We chose to allow numbers and some punctuation to be used in ticker symbols.

We chose not to inline the SVG logo images into the document, because it will make the document bigger and more unwieldy, and harder to edit, particularly if the SVG has to be escaped and stored as a string. Base64-encoded SVG is considered an anti-pattern which increases the size of the document and makes it harder to consume. The logos are expected to be online at reachable URLs anyways, so simply storing their blake2b hash in the signed metadata document reduces the size and complexity of the signed metadata document. There are blake2b libraries available in javascript. This pattern is also extensible if we need to have more than one version of the logo appropriate for different form factors, we can simply add more `logo_svg_size_url...` fields.

This is based on looking at token lists on various cryptocurrency exchanges:

* [Binance.US token list](https://support.binance.us/hc/en-us/articles/360049417674-List-of-Supported-Assets)
* Kraken's exchange includes the following symbols, where suffixes such as `.S` suffix are used with staked tokens and similar.

```
[
  "1INCH",
  "AAVE",
  "ACA",
  "ACH",
  "ADA",
  "ADA.S",
  "ADX",
  "AED",
  "AED.HOLD",
  ...
  "DOT",
  "DOT.P",
  "DOT.S",
  "DYDX",
  "EGLD",
  "ENJ",
  "ENS",
  "EOS",
  "ETH2",
  "ETH2.S",
  "ETHW",
  "EUR.HOLD",
  "EUR.M",
  ...
]
```

Coinbase lists over 9000 tokens with letters, numbers, mixed case, and as many as 8 characters: `BabyDoge, LYXe, vBUSD, 7S, DRAGACE, TRINITY`.
Coinbase also lists the tickers `OXD V2`, `$ZOOM`, which would not be allowed in the above specification due to the space character and `$` character.

We do not recommend that space or `$` become allowed characters, as this creates the opportunity for confusion.

# Prior art
[prior-art]: #prior-art

We considered the metadata used in both Cardano native assets and Algorand standard assets when considering this proposal.
We also considered how tokens are assigned metadata in the Ethereum ecosystem.

In the Ethereum ecosystem, most tokens (fungible and non-fungible) have logos, but they are typically stored on centralized servers, and referred to via URLs.

In [EIS-2569](https://eips.ethereum.org/EIPS/eip-2569) it was proposed to actually store SVG data on chain.

Typically, wallets like MetaMask use services like Infura to access names and logos of tokens. There is not a standard way outside of these
services to get this data for an arbitrary ERC-20.

In Algorand, any standard asset that is created has [required parameters](https://developer.algorand.org/docs/get-details/asa/#asset-parameters)

> These eight parameters can *only* be specified when an asset is created.
>
>   * Creator (required)
>   * AssetName (optional, but recommended)
>   * UnitName (optional, but recommended)
>   * Total (required)
>   * Decimals (required)
>   * DefaultFrozen (required)
>   * URL (optional)
>   * MetaDataHash (optional)

In Cardano, native assets can be created before being [registered with the token registry](https://developers.cardano.org/docs/native-tokens/token-registry/How-to-prepare-an-entry-for-the-registry-NA-policy-script). Only a name is required to create the asset. When subsequently registering an asset, a description is required. Then a ticker, url, logo, and decimals are all considered optional. In the Cardano registry, metadata can be deleted and updated, but it requires a signature from the creator.

In a [blog post](https://moxie.org/2022/01/07/web3-first-impressions.html), Moxie Marlinspike criticized web3 technologies that commit url's pointing to images to blockchains, but don't include e.g. a hash of the image on the chain, which would allow the user to verify that the url resolved to the correct image. In this proposal, SVG images are stored online, but a blake2b digest of their contents is stored in the metadata document for validation.

We examined [Microsoft's localization guide](https://learn.microsoft.com/en-us/globalization/locale/currency-formatting) and [Shopify's localization guide](https://polaris.shopify.com/foundations/formatting-localized-currency) when considering how currency localization should work and how that connects to token metadata.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

In the future we hope to do on-chain token metadata as envisioned in [MCIP 40](https://github.com/mobilecoinfoundation/mcips/pull/0040).

This is likely required for doing user-created tokens, and will be required before there are too many tokens for this document to be manageable by humans.
