- Start Date: 2019-07-15
- Target Major Version: 2.x

# Dinero.js v2 RFC

This document outlines the proposal for the v2 of Dinero.js.

## Purpose

### 🐞 Problems with v1

Dinero.js v1.x currently has **several design issues**:

- It's written in JavaScript, which makes it hard to use in TypeScript projects. There's a [hand-maintained declaration file on DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/dinero.js), but it's bound to get out of date all the time.
- It uses numbers to represent amounts, which makes it unable to handle amounts above `Number.MAX_SAFE_INTEGER`.
- It doesn't automatically handle different exponents and uses a precision of `2` by default ([#9](https://github.com/sarahdayan/dinero.js/issues/9)).
- It relies on the ECMAScript I18n API to format objects into human-readable strings, which makes it inconsistent across different runtimes and forces it only to accept ISO 4217 currencies ([#36](https://github.com/sarahdayan/dinero.js/issues/36)).
- It requires passing integers as amounts and doesn't parse decimals ([#58](https://github.com/sarahdayan/dinero.js/issues/58)).
- It's architected in a way that makes it hard to create independent services ([#58](https://github.com/sarahdayan/dinero.js/issues/58)).

### 👩🏿‍💻 Who it's for

Dinero.js aims at becoming a **go-to library for any project that requires monetary manipulation**. It goes from personal projects to banking and stock exchange apps.

The goal is for it to provide a robust and developer-friendly API that scales.

### ❓ Why

All the current problems can be an issue for adopting Dinero.js in a project.

- The robustness issues (lack of typing, lack of big integers support) can be problematic for projects with high standards around dependencies, and projects that manipulate significant monetary values.
- The current developer experience (DX) is far from what it could be, and some usages require that developers write their own factories.

## Features

### 🏠 Architecture & high-level design

Dinero.js v2 should be written in TypeScript, and transpiled into JavaScript along with a declaration file.

It should be organized in a monorepo to make it easier to create companion services and factories.

### 📐 API

Dinero.js v2 should expose the following:

- The core API, similar to v1. It should exclude deprecated v1 methods such as [`hasCents`](https://sarahdayan.github.io/dinero.js/module-Dinero.html#~hasCents) and fix design issues such as the dependency on the I18n API ([`toFormat`](https://sarahdayan.github.io/dinero.js/module-Dinero.html#~toFormat)) or on a REST API ([`convert`](https://sarahdayan.github.io/dinero.js/module-Dinero.html#~convert)).
- An optional service which allows currency assertions and automatic exponent inclusion.
- A set of helpers to create specific `Dinero` objects more easily.

#### Core API

##### Current design flaws

The main flaws in Dinero.js v1 are how it encapsulates certain behaviors, locking users into a specific usage, and not allowing for flexibility.

Two good examples are `toFormat` and `convert`:

- `toFormat` wraps around the I18n API. It creates two issues:
  - The API doesn't output an exact desired format, but a localized currency string. This behavior is confusing, because Dinero lets users pass a mask, so they reasonably expect an exact projection.
  - The I18n API has inconsistent implementations across different environments, which makes the API unpredictable and even more confusing.
- `convert` wraps around a provided REST API to hide the request logic. It creates two issues:
  - It forces users to consume a REST API, preventing them from querying rates from another source (in memory, a file) or use a different protocol (SOAP, GraphQL).
  - It performs a new network request at each call, preventing users from caching results.

`toFormat` should no longer rely on the I18n API, and instead format the object exactly as specified in the mask. The symbol would either be replaced by the currency in full or come from using the currency service. Users would still be able to use the I18n API through the usage of [`toRoundedUnit`](https://sarahdayan.github.io/dinero.js/module-Dinero.html#~toRoundedUnit).

`convert` should no longer expect a REST API and encapsulate the fetching behavior. Instead, it should expect a Promise which resolves to a `Rates` object.

```js
const rates = new Promise(resolve =>
  resolve({
    EUR: 0.81162
  })
);

Dinero({ amount: 500 }).convert("EUR", { rates });
```

##### Integers & BigInts

Dinero.js v1 uses integers for `amount` (though the `number` primitive), and this shouldn't change, but Dinero.js v2 should also allow for `bigint` [as soon as the proposal goes to stage 4](https://tc39.es/proposal-bigint/).

In the meantime, and to support environments with no `bigint` support, Dinero.js v2 should let users provide their own `Calculator`. This way, they can use whatever library they want and map the calculator functions. This behavior should be made available through a Dinero factory, so it doesn't "pollute" the API.

```js
import { createDinero } from "@dinero.js/helpers";
import Big from "some-bigint-library";

const Dinero = createDinero({
  calculator: {
    add: Big.add,
    subtract: Big.subtract
    // ...
  }
});
```

A `Calculator` type should be provided to ease the creation with TypeScript:

```ts
const BigIntCalculator: Calculator = {
  add: Big.add,
  subtract: Big.subtract
  // ...
};
```

#### Currency service

Currently, users must specify the `precision` of their `Dinero` objects, which is set to `2` by default. When a user creates objects with a currency which exponent is different from `2` (e.g., Iraqi dinars, Japanese yens, etc.), they must specify the `precision` by hand. One way to alleviate it is by setting a `defaultPrecision`, but this only works when one is using a single currency throughout their project.

The currency service lets the user pass a `Currency` object instead of a plain string to the `currency` property when creating `Dinero` objects. `Currency` objects embark the currency's exponent, allowing the `Dinero` object to set the `precision` property automatically.

```js
import Dinero from "dinero.js";
import { IQD } from "@dinero.js/currencies";

// `precision` is set to `2` (default)
const d1 = Dinero({ amount: 1000, currency: "IQD" });

// `precision` is automatically set to `3`
const d2 = Dinero({ amount: 1000, currency: IQD });
```

`Currency` objects could also embark the currency's symbol, allowing the library to fully own the currency formatting within the [`toFormat`](https://sarahdayan.github.io/dinero.js/module-Dinero.html#~toFormat) method and liberate itself from its dependency on the I18n API.

```js
import Dinero from "dinero.js";
import { EUR } from "@dinero.js/currencies";

const d1 = Dinero({ amount: 100, currency: "EUR" });
const d2 = Dinero({ amount: 100, currency: EUR });

d1.toFormat("$0,0.00"); // EUR100.00
d2.toFormat("$0,0.00"); // €100.00
```

`Dinero` objects created with a string `currency` instead of a `Currency` object don't have any information on the currency symbol. Consequently, they would output the currency code in full instead of the symbol.

The I18n API could still be used on Dinero objects as with v1, by chaining it to a `toRoundedUnit` method call: `Dinero().toRoundedUnit(digits, roundingMode).toLocaleString(locale, options)`.

Users would also be able to create their own, custom currencies, breaking out of the ISO 4217 limit.

```js
const FOO = {
  name: "foo",
  symbol: "F",
  code: "FOO",
  exponent: 4
};
```

TypeScript would further facilitate it, as they would have access to a `Currency` type.

```ts
const FOO: Currency = {
  name: "foo",
  symbol: "F",
  code: "FOO",
  exponent: 4
};
```

#### Helpers

Dinero v1 constrains users always to create `Dinero` objects the same way, using the main `Dinero` factory. It is limiting for several reasons:

- It requires passing amounts as integers, which requires that users perform manual conversion when their initial data isn't in the right format.
- It relies on globals to facilitate the creation of many objects with the same currency. It creates a limit to a single currency at a time and makes changing the default imperative.

Helpers solve that problem by letting the user pass information in various formats, or use currency factories as shortcuts.

```js
import { fromFloat, fromFormat, DineroUSD } from "@dinero.js/helpers";

// returns a Dinero object with `amount` equal to 49999
// and `precision` equal to `2`
fromFloat({ amount: 499.99 });

// returns a Dinero object with `amount` equal to 15000,
// `precision` equal to `2` and `currency` equal to `USD`
// (using the `Currency` service under the hood)
fromFormat("$150", "$0");

// returns a Dinero object with `amount` equal to 4500
// and precision equal to `2` (using the `Currency` service under the hood)
DineroUSD(45);
```
