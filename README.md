# ECMAScript Proposal: Destructuring with Variable Declaration Assignment

> [!WARNING]
> This proposal is under development and has not yet been championed to TC39. Contributions and feedback are welcome - please see [contribution](#contribute) for details.

- [Overview](#overview)
- [Motivation](#motivation)
- [Current syntax](#current-syntax)
- [Proposed syntax](#proposed-syntax)
- [Usage example](#usage-example)
- [Similar / Prior art](#similar--prior-art)
- [Contribute](#contribute)

## Overview

This proposal aims to extend the current array destructuring syntax in JavaScript by allowing variable declarations (`var`, `let`, `const`) to be assigned directly to individual elements within a destructuring block. This enhancement is particularily useful in scenarios when destructuring from a tuple like `[error, result]`, and the `error` needs to be reassigned in the code that comes after.

## Motivation

- **Simplified Error Handling**: This syntax facilitates error handling patterns where a variable (e.g. `error`) may need to be reassigned multiple times within the same scope
- **Cleaner Code**: It reduces the need for extra lines of code to declare and assign variables separately. The advantages are better in a TypeScript codebase.

This proposal complements ongoing discussions around enhances the [Safe Assignment Operator proposal](https://github.com/arthurfiorette/proposal-safe-assignment-operator), providing developers with more robust tools for managing `Promise` errors.

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
// Syntax Overview
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

Either a `default` variable type must be declared at the beginning of the destructuring block, or each destructured element must have its own declaration. The following code should execute to avoid unexpected errors:

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

Lastly, if there's an attempt to redeclare a variable's type within the current scope to a different one (`var/let/const`), the code should not execute and should instead throw a SyntaxError:

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

This syntax would integrate well with another ongoing proposal - https://github.com/arthurfiorette/proposal-safe-assignment-operator. I've been using the [`to`](https://github.com/scopsy/await-to-js/blob/master/src/await-to-js.ts) util to achieve a similar result, and I often wanted to reassign the `error` in the same scope, but I'd have to declare both variables with `let`.

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
// - With pseudo code to showcase the use case

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
