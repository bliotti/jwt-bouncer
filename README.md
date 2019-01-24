# jwt-bouncer

![npm version](https://img.shields.io/badge/npm-1.0.1-blue.svg) ![Licence MIT](https://img.shields.io/badge/licence-MIT-yellowgreen.svg) ![Open Issues](https://img.shields.io/github/issues-raw/tripott/jwt-bouncer.svg)

A security guard to protect your APIs from undesirable JWTs by verifying JsonWebTokens. Services, such as CDS Hook services, should whitelist the `iss` and `jku` fields to only clients, such tenants using an EHR, they trust.

JsonWebTokens help secure communication between two systems, such as browser to server, or server to server. Signing a token allows the service to trust the client/sender or "issuer". You can sign a token with several different algorithms. This process provides a stateless approach to authentication.

`jwt-bouncer` is used to create ExpressJS middlewares to help determine whether a JWT is present within a whitelist and valid. `jwt-bouncer` isn't concerned with storing, managing, retrieving, or the shape of the whitelist contents.

> It's up to you to provide the whitelist on the request object using your own ExpressJS middleware prior to using middleware generated by `jwt-bouncer`.

Although not required, `jwt-bouncer` can be used in conjunction with the [`jwt-bouncer-sample-validators`](https://www.npmjs.com/package/jwt-bouncer-sample-validators) module which provides sample whitelist and jwt validator functions. See **Usage**.

> You pass a validator function to `jwt-bouncer` which will invoke the validator to check the jwt. The bulk of the complexity lies within each validator function. Review our [`jwt-bouncer-sample-validators`](https://www.npmjs.com/package/jwt-bouncer-sample-validators) samples for a jump start on creating your own validators. See **jwt-bouncer-sample-validators**.

While we use `jwt-bouncer` in an attempt to **[Trust CDS Clients](https://cds-hooks.org/specification/1.0/#trusting-cds-clients)** as part of the SMART on FHIR CDS Hooks specification, you could use `jwt-bouncer` in other situations where JWTs require validation.

## Installation

```bash
npm install jwt-bouncer
npm install jwt-bouncer-sample-validators
```

## Usage

`jwt-bouncer` isn't concerned with storing or managing the whitelist. It just prevents undesirable JWTs. You'll have to write your own middleware to retrieve the whitelist from your own persisted storage and put the whitelist as a `whitelist` property on the request object.

This sample uses the sample jwt validators from [`jwt-bouncer-sample-validators`](https://www.npmjs.com/package/jwt-bouncer-sample-validators) and passes them to `jwtbouncer` to create jwt validation middlewares.

```js
const express = require("express");
const app = express();

// You'll have to write your own middleware to retrieve the whitelist from
// your own persisted storage and put the whitelist as a `whitelist` property on the request object.
const getWhitelistFromDBMiddleware = async (req, res, next) => {
  // get the whitelist from your persisted data store. That is up to you.
  const whitelist = await getWhitelist();
  req.whitelist = whitelist;
  next();
};

const jwtBouncer = require("jwt-bouncer");

//  We've supplied a sample validators in the `jwt-bouncer-sample-validators` npm module, but
//    you are free to write your own.
//  Feel free to contribute your own validator samples.
const {
  whitelistValidator,
  jwtValidator
} = require("jwt-bouncer-sample-validators");

// create your bouncer middlewares by providing
//  - the request property name (whitelistValidatorResult) for the results of the validator
//  - the validator function
//  - An object containing data that you wish to send to the validator function
//  Downstream middlewares can inspect the results using the provided property name.
const whitelistValidationMiddleWare = jwtBouncer(
  "whitelistValidatorResult",
  whitelistValidator,
  {
    apiErrorDocsURL: `https://your.api.docs/errors`
  }
);

const jwtValidationMiddleWare = jwtBouncer("jwtValidatorResult", jwtValidator, {
  apiErrorDocsURL: `https://your.api.docs/errors`
});

app.get(
  "/:tenant/cds-services",
  getWhitelistFromDBMiddleware,
  whitelistValidationMiddleWare,
  jwtValidationMiddleWare,
  async (req, res, next) => {
    const targetCDSServiceResult = await getCDSServicesFromTarget();

    res.status(200).send(targetCDSServiceResult);
  }
);

app.use(function(err, req, res, next) {
  res.status(err.status || 500).send(err);
});

app.listen(process.env.PORT || 9000, function() {
  console.log("Listening on port %d", server.address().port);
});
```

## Sample Whitelist

You must manage your own whitelist and supply it to the request object early in the flow of middleware calls. See **Usage**. Here is a sample whitelist array that we use to maintain a list of valid clients. You don't have to use our whitelist structure. Your whitelist and validators can be tailored to your needs.

```json
[
  {
    "iss": "https://sandbox.cds-hooks.org",
    "tenant": "48163c5e-88b5-4cb3-92d3-23b800caa927",
    "jku": "https://sandbox.cds-hooks.org/.well-known/jwks.json",
    "uriPathTenant": "labs",
    "enabled": true
  },
  {
    "iss": "https://sandbox.cds-hooks.org",
    "tenant": "48163c5e-88b5-4cb3-92d3-23b800caa927",
    "jku": "https://sandbox.cds-hooks.org/.well-known/jwks.json",
    "uriPathTenant": "foo",
    "enabled": false
  }
]
```

## How it works

`jwt-bouncer` is higher order function that takes 3 arguments:

- `validationResultsProp` - A validation results property name - we use this to pass the results of the validator to the req object before calling next.
- `validator` - A validator function - this is where the work occurs. This function should accept an options object, perform validation, and return a results object.
- `options` - an options object - Pass an object with whatever values you wish. These options will be passed to the validator function when the middleware is invoked.

The function returns an ExpressJS "bouncer" middleware function. The bouncer calls the validator function, passing an options object.

> Each validator function allows you to provide your own validation rules. We've provided sample whitelisting and jwt verification validators within [`jwt-bouncer-sample-validators`](https://www.npmjs.com/package/jwt-bouncer-sample-validators).

Each validator function returns an validation results object. See [`jwt-bouncer-sample-validators`](https://www.npmjs.com/package/jwt-bouncer-sample-validators). If validation is ok then it creates a property on the request object containing the validation results object. If there's a problem, the bouncer calls `next` passing the error.

```js
module.exports = (validationResultsProp, validator, options) => (
  req,
  res,
  next
) => {
  const validationResults = validator({ req, ...options });

  if (validationResults.ok) {
    req[validationResultsProp] = validationResults;
    next();
  } else {
    next(validationResults.err);
  }
};
```

## [jwt-bouncer-sample-validators](https://www.npmjs.com/package/jwt-bouncer-sample-validators)

### Whitelist Validator

The Whitelist validator sample function takes an `options` object as its argument. The `options` object will contain a propery named `req` which represents the ExpressJS request object and a url to the error documentation. The validator determines if a JWT is present on the incoming request as an `Authorization` header containing a `Bearer` token. If so, it decodes and checks the JWT against a whitelist.

> This validator assumes that a whitelist array has already been retrieved by prior middleware and is available as a `whitelist` property on the request object.

If everything is ok, the validator returns an object containing these properties:

- `ok` - a value of `true` denotes success
- `whiteListItem` - the found item in the whitelist
- `token` - the decoded token

```js
{
  ok: true,
  whiteListItem: foundWhiteListItem,
  token: decoded
}
```

If the jwt did not pass whitelist validation then the object returned from the validator will look like:

```js
{
  ok: false, err;
}
```

## JWT Validator

JWT Validator function that takes an options object as its argument containing `req` and `apiErrorDocsURL` properties. The validator then:

- Retrieves a json web key (jwk) from a publicly available JSON Web Keyset resource served from an url endpoint.
- Converts the json web key to PEM format
  > Privacy-Enhanced Mail (PEM) is a de facto file format for storing and sending cryptographic keys, certificates, and other data, based on a set of 1993 IETF standards defining "privacy-enhanced mail."
- Verifies the JWT token and audience using against a publicly available JSON Web Keyset resource served from an url endpoint
