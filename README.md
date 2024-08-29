# ECMAScript proposal: Destructuring with variable declaration assignment

> [!WARNING]
> This proposal is under development and has not been championed to TC39 yet. Any [contribution](#contribute) is welcome!

- [Overview](#overview)
- [Motiovation](#motiovation)
- [Current syntax](#current-syntax)
- [Proposed syntax](#proposed-syntax)
- [Usage example](#usage-example)
- [Similar / Prior art](#similar--prior-art)
- [Contribute](#contribute)

## Overview

This proposal is chapioning the expansion of the current array destructuring syntax. The new extra syntax will allow for individual variable declaration assignment (`var/let/const`) on each destructured array element. This proves especially useful when destructuring a tuple of similar shape `[error, result]` and there is a need to re-assign the value of `error` in the code below.

## Motiovation

- Better readability
- Flatter code
- Simplified error handling
- Combo with https://github.com/arthurfiorette/proposal-safe-assignment-operator?tab=readme-ov-file#help-us-improve-this-proposal

## Current syntax

Variable destructuring was an exciting addition to the JS language - millions of developers use it daily and you can find it being used in most of the modern software as it makes the code more readable:

```typescript
// Solid.js Signals example
const [state, setState] = createSignal(...);

// Simple array destructuring with `...rest` example
const [first, second, ...rest] = [
  "Destructuring",
  "with",
  "Variable",
  "Declaration",
  "Assignment"
];
```

```typescript
// Syntax overview
const [foo, bar] = [1, 2];

let [foo, bar] = [1, 2];

let foo, bar;
[foo, bar] = [1, 2];

var [foo, bar] = [1, 2];

var foo, bar;
[foo, bar] = [1, 2];
```

## Proposed syntax

Any declaration that's specified within the destructuring block will take precedence over the default.

```typescript
// - Example 1 -
// `foo` -> `const` since that is the default
// `bar` -> `let` will override the `const` as it's closer to the variable
// `rest` -> `const`
const [foo, let bar, ...rest] = [1, 2, 3, 4];

// - Example 2 -
// `foo` -> `const`
// `bar` -> `let`
// `baz` -> `var`
const [foo, let bar, var baz] = [1, 2, 3, 4];

// - Example 3 -
// `foo` -> `let`
// `bar` -> `let`
// `baz` -> `const`
let [foo, bar, const baz] = [1, 2, 3, 4];
```

The following code should behave identically to the code above:

```typescript
// - Example 1 -
// `foo` -> `const` since that is the default
// `bar` -> `let` will override the `const` as it's closer to the variable
// `rest` -> `const`
[const foo, let bar, const ...rest] = [1, 2, 3, 4];
[const foo, let bar, ...const rest] = [1, 2, 3, 4]; // TODO: Figure out where to place the `...`

// - Example 2 -
// `foo` -> `const`
// `bar` -> `let`
// `baz` -> `var`
[const foo, let bar, var baz] = [1, 2, 3, 4];
```

Either a `default` variable type must be declared at the beginning of the destructuring block, or each destructured element must have its own declaration. The following code should not run to avoid unexpected errors:

```typescript
// `_` -> `var`
// `bar` -> `const`
// `rest` -> ???
[var _, const bar, ...rest] = [1, 2, 3, 4];
```

The only exception to the above is when the variable is already declared and is available in the current scope:

```typescript
// Declared `let bar` here
//          |
//          V
[const _, let bar] = [1, 2, 3, 4];

// - Example 1 -
// `hello` -> `const`
// `bar` is declared with `let` above
// `world` -> `const`
const [hello, bar, world] = ["hello", "bar", "world"];
```

Lastly, if there's an attempt to re-declare the variable type for a variable in the current scope to a different one (`var/let/const`), the code should not run and instead throw a `SyntaxError`:

```typescript
// Declared `let bar` here
//          |
//          V
[const _, let bar] = [1, 2, 3, 4];

// Can't change from existing `let bar` to `const bar`
//             |
//             V
[const hello, const bar, let world] = ["hello", "bar", "world"];
```

## Usage example

This syntax would play nicely with another ongoing proposal - https://github.com/arthurfiorette/proposal-safe-assignment-operator. I've been using the [`to`](https://github.com/scopsy/await-to-js/blob/master/src/await-to-js.ts) util to achieve a similar result, and often I wanted to reassign the `error` in the same scope, but I'd have to declare both variables with `let`.

```typescript
// https://github.com/scopsy/await-to-js/blob/master/src/await-to-js.ts
export function to<T, U = Error>(
  promise: Promise<T>,
  errorExt?: object,
): Promise<[U, undefined] | [null, T]> {
  return promise
    .then<[null, T]>((data: T) => [null, data])
    .catch<[U, undefined]>((err: U) => {
      if (errorExt) {
        const parsedError = Object.assign({}, err, errorExt);
        return [parsedError, undefined];
      }

      return [err, undefined];
    });
}

// Usage
// Instead of the try/catch block
try {
  const matchingUser = await prisma.user.findUnique(...);
} catch (error) {
  // handle `error`
}

// Use the `to` util
const [error, matchingUser] = await to(prisma.user.findUnique({ ... }));
```

```typescript
// Real-world scenario
// - SolidStart code
// - Using the `to` function from the previous code block
// - With pseudo code to showcase the usecase

const signInAction = action(async (payload: SignInPayloadType) => {
  "use server"

  const [let err, matchingUser] = await to(prisma.user.findUnique({ ... }));
  // or
  // [let err, const matchingUser] = await to(prisma.user.findUnique({ ... }));

  // The error from the `prisma.user.findUnique` above
  if (err) {
    return rpcErrorResponse(err);
  }

  // do stuff with `matchingUser`

  // Since the variable `err` was declared with `let` above, here it doesn't need to be declared specifically
  [err, const isValidPassword] = await to(fake_validatePassword(
    matchingUser.encryptedPassword,
    payload.password
  ));

  // The error from the `fake_validatePassword` above
  if (err) {
    return rpcErrorResponse(err);
  }

  // do stuff with `isValidPassword`

  [err, const emailSentSuccess] = await to(fake_sendConfirmationEmail(
    matchingUser,
  ));

  // The error from the `fake_sendConfirmationEmail` above
  if (err) {
    return rpcErrorResponse(err);
  }

  // do stuff with `emailSentSuccess`

  ...

  return rpcSuccessResponse(matchingUser);
}, "auth:sign-in");
```

## Similar / Prior art

- **Go**'s error handling allows re-using the `err` variable (https://go.dev/blog/error-handling-and-go)

## Contribute

This proposal is in its very early stages as I'm still finding more use cases. Feel free to open an issue or submit a PR with your suggestions/improvements.

Any contribution is valuable!
