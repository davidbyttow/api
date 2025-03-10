# API

We are commited to building an open plateform on which everyone can build its own experience that leverages our assets.

The Cards are stored on the Ethereum blockchain that is publicly available but we also provide a free API with richer information.

## GraphQL

Most of our API uses the [GraphQL](https://graphql.org/) query language. It is documented in our [GraphQL playground](https://api.sorare.com/graphql/playground).

## Authentication

A GQL mutation is available for authentication: `signIn`.

It expects the following payload: `{ "user": { "email": "myemail@mydomain.com", "password": "myhashedpassword" } }`

```gql
mutation SignInMutation($input: signInInput!) {
  signIn(input: $input) {
    currentUser {
      slug
    }
    errors {
      message
    }
  }
}
```

Variables :

```json
{ "input": { "email": "youremail", "password": "yourpassword" } }
```

The password needs to be hash client side using a salt. The salt needs to be retrieved with `GET /api/v1/users/myemail@mydomain.com`.

Once the salt is retrieved the hash password can be computed with bcrypt. In JavaScript:

```javascript
import bcrypt from 'bcryptjs';

bcrypt.hashSync(password, salt);
```

You'll also need to pass a `_sorare_session_id` cookie and a `X-CSRF-TOKEN` token header. They can be retrieved with the same request as the salt.

### With a cookie

To use a cookie authentication you need to store the new `_sorare_session_id` and `X-CSRF-TOKEN` and pass it to all next requests.

### With a JWT token

To authenticate by JWT you need to request a valid JWT token in the `signIn` mutation :

```gql
  mutation SignInMutation($input: signInInput!) {
    signIn(input: $input) {
      currentUser {
        slug
        jwtToken(aud: "YOUR_AUD") {
          token
          expiredAt
        }
      }
      ...
    }
  }
```

`YOUR_AUD` is a mandatory string parameter that should help identify your app

You can then pass it to all next requests through the following headers :

```
Authorization: Bearer YOUR_TOKEN
JWT-AUD: YOUR_AUD
```

## 2FA

For account with 2FA enabled the `signIn` mutation will return an `otpSessionChallenge` instead of the `currentUser`.

You then need to do a second call to the `signIn` mutation providing only the `otpSessionChallenge` and a valid `otpAttempt` :

```json
{
  "input": {
    "otpSessionChallenge": "eca010be19a80de5c134c324af24c36f",
    "otpAttempt": "788143"
  }
}
```

## OAuth

If you want to make requests on behalf of other users you can pass an OAuth access token. We'll need to create the OAuth Application for you first. This is done manually for now, on request.

## Rate limit

The API is rate limited by default. We can provide an API Key on demand that raises the limits. The API key should be passed in an http `APIKEY` header.

## Signing auction bids and offers

Every operation that involves money transfers should be signed with your Starkware private key. It can be exported from sorare.com using your wallet.

![Private key export](./private_key_export.png)

You can sign payloads using `signLimitOrder` from https://github.com/sorare/crypto.

### Signing auction bids

1. Get the signable payload

```gql
query BidLimitOrder($auctionSlug: String!, $amount: String!) {
  englishAuction(slug: $auctionSlug) {
    ... on EnglishAuctionInterface {
      limitOrders(amount: $amount) {
        vaultIdSell
        vaultIdBuy
        amountSell
        amountBuy
        tokenSell
        tokenBuy
        nonce
        expirationTimestamp
      }
    }
  }
}
```

Those limit orders should be signed with your Starkware private key.

2. Post the bid with the signature

```gql
mutation Bid($input: bidInput!) {
  bid(input: $input) {
    bid {
      id
    }
  }
}
```

### Creating offers

1. Get the signable payload

```gql
mutation NewOfferLimitOrders($input: prepareOfferInput!) {
  prepareOffer(input: $input) {
    limitOrders {
      vaultIdSell
      vaultIdBuy
      amountSell
      amountBuy
      tokenSell
      tokenBuy
      nonce
      expirationTimestamp
    }
  }
}
```

Those limit orders should be signed with your Starkware private key.

2. Post the new offer with the signature(s)

You can then use any of `createSingleSaleOffer`, `createDirectOffer` or `createSingleBuyOffer` mutations and provide the signature. Note that you need to provide a dealId. It can be generated in the browser using: `window.crypto.getRandomValues(new Uint32Array(4)).join('')`.

### Accepting offers

1. Get the signable payload

```gql
query OfferLimitOrders($offerId: String!) {
  transferMarket {
    id
    offer(id: $offerId) {
      ... on OfferInterface {
        receiverLimitOrders {
          vaultIdSell
          vaultIdBuy
          amountSell
          amountBuy
          tokenSell
          tokenBuy
          nonce
          expirationTimestamp
        }
      }
    }
  }
}
```

Those limit orders should be signed with your Starkware private key.

2. Post accept offer with the signatures(s)

Use the `acceptOffer` mutation providing the signature.
