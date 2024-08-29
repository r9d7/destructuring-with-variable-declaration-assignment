# ECMAScript Proposal: Destructuring with Variable Declaration Assignment

> [!WARNING]
> This proposal is under development and has not yet been championed to TC39. Contributions and feedback are welcome - please see [contribution](#contribute) for details.

- [Overview](#overview)
- [Motivation](#motivation)
- [Current Syntax](#current-syntax)
- [Proposed Syntax](#proposed-syntax)
  - [Syntax Examples](#syntax-examples)
  - [Rules and Exceptions](#rules-and-exceptions)
- [Usage Example](#usage-example)
- [Similar / Prior art](#similar--prior-art)
- [Contribute](#contribute)

## Overview

This proposal aims to extend the current array destructuring syntax in JavaScript by allowing variable declarations (`var`, `let`, `const`) to be assigned directly to individual elements within a destructuring block. This enhancement is particularily useful in scenarios when destructuring from a tuple like `[error, result]`, and the `error` needs to be reassigned in the code that comes after.

## Motivation

- **Simplified Error Handling**: This syntax facilitates error handling patterns where a variable (e.g. `error`) may need to be reassigned multiple times within the same scope
- **Cleaner Code**: It reduces the need for extra lines of code to declare and assign variables separately. The advantages are better in a TypeScript codebase.

This proposal complements ongoing discussions around enhances the [Safe Assignment Operator proposal](https://github.com/arthurfiorette/proposal-safe-assignment-operator), providing developers with more robust tools for managing `Promise` errors.

## Current Syntax

JavaScript's existing destructuring syntax is widely used for its ability to make code more readable and concise. Here are some examples of the current syntax:

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

## Proposed Syntax

This proposal introduces the ability to specify variable declarations (`var`, `let`, `const`) directly within a destructuring assignment. The declaration specified for each element takes precedence over the default declaration for the destructuring block.

### Syntax Examples

```typescript
// Example 1
// `foo` -> `const` (default)
// `bar` -> `let` (overrides default)
// `rest` -> `const` (default)
const [foo, let bar, ...rest] = [1, 2, 3, 4];

// Example 2
// `foo` -> `const`
// `bar` -> `let`
// `baz` -> `var`
const [foo, let bar, var baz] = [1, 2, 3, 4];

// Example 3
// `foo` -> `let`
// `bar` -> `let`
// `baz` -> `const`
let [foo, bar, const baz] = [1, 2, 3, 4];
```

In these examples, the declaration type (`var`, `let`, `const`) closest to the variable name takes precedence. The following syntax should behave identically:

```typescript
// Example 1 (equivalent)
[const foo, let bar, const ...rest] = [1, 2, 3, 4];
[const foo, let bar, ...const rest] = [1, 2, 3, 4]; // TODO: Figure out where to place the `...`

// Example 2 (equivalent)
[const foo, let bar, var baz] = [1, 2, 3, 4];
```

### Rules and Exceptions

1. **Default Declaration Requirement**: A default variable type must be declared at the beginning of the destructuring block, or each element within the block must specify its own declaration. The following example would throw an error since `rest` lacks a declaration:

```typescript
// `_` -> `var`
// `bar` -> `const`
// `rest` -> ???
[var _, const bar, ...rest] = [1, 2, 3, 4]; // Error: `rest` lacks a declaration
```

2. **Existing Variables in Scope**: If a variable is already declared in the current scope, the proposal allows using that variable within the destructuring assignment without redeclaring its type (`var` and `let` only):

```typescript
// Example 1
let bar; // `bar` is declared as `let`

[const foo, bar] = [1, 2, 3, 4]; // `bar` retains its original `let` declaration

// Example 2
[const _, let bar] = [1, 2, 3, 4]; // `bar` is declared as `let`

const [hello, bar, world] = ["hello", "bar", "world"]; // `bar` retains it's original `let` declaration, but the value is reassigned
```

3. **Type Mismatch Prevention**: The code should throw a `SyntaxError` if there is an attempt to redeclare a variable in the current scope with a different type (`var`, `let`, `const`):

```typescript
let bar; // Declared as `let`

[const foo, const bar, let world] = ["hello", "bar", "world"]; // Error: `bar` cannot be redeclared as `const`
```

## Usage Example

This syntax is particularly useful when combined with a utility like the [`to`](https://github.com/scopsy/await-to-js/blob/master/src/await-to-js.ts) function, as shown below: 

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

// Usage in a real-world scenario
// - SolidStart code
// - Using the `to` utility
// - Some pseudo code to showcase the use case
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

  // Since the variable `err` was declared with `let` above, here it can just be reassigned
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

- **Go**'s error-handling patterns allow reusing the `err` variable across multiple operations, and was the inspiration behind this proposal. More details can be found [here](https://go.dev/blog/error-handling-and-go).

## Contribute

This proposal is in its early stages, and we are actively seeking feedback and contributions to refine the idea. Please feel free to open an issue or submit a PR with your suggestions & improvements.

Every contribution is valuable and will help shape this proposal into a robust champion.