# ECMAScript proposal: Destructuring with variable declaration assignment

## Status

This proposal is in WIP mode and has not been championed to TC39 yet.

## Overview / Motivation

Variable destructuring was an exciting addition to the JS spec - it is a useful feature of the language used daily by millions of developers and you can find it being used in most of the modern software:

```typescript
// Solid.js Signals
const [state, setState] = createSignal(...);

const [first, second, ...rest] = [
  "Destructuring",
  "with",
  "Variable",
  "Declaration",
  "Assignment"
];

// Read through this function, we will need it in a bit
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

const [err, result] = await to(prisma.user.findUnique({ ... }));
```

This already looks very good, it's flexible and powerful, and also provides a good developer experience, but there currently is no way to declare the variable type (var/let/const) for each destructured element individually. This makes impossible coding patterns like the following:

```typescript
// Real-world scenario
// - SolidStart code
// - Using the `to` function from the previous code block
// - With pseudo code to showcase the usecase

const signInAction = action(async (payload: SignInPayloadType) => {
  "use server"

  const [let err, matchingUser] = await to(prisma.user.findUnique({ ... }));
  // Or
  [let err, const matchingUser] = await to(prisma.user.findUnique({ ... }));

  if (err) {
    return rpcErrorResponse(err);
  }

  [err, const isValidPassword] = await to(fake_validatePassword(
    matchingUser.encryptedPassword,
    payload.password
  ));

  if (err) {
    return rpcErrorResponse(err);
  }

  [err, const emailSentSuccess] = await to(fake_sendConfirmationEmail(
    matchingUser,
  ));

  if (err) {
    return rpcErrorResponse(err);
  }

  ...

  return rpcSuccessResponse(matchingUser);
}, "auth:sign-in");
```

At the moment these are the only ways we can customise the variable type for the destructured arrays:

```typescript
// const
const [foo, bar] = [1, 2];

// let
let [foo, bar] = [1, 2];

let foo, bar
[foo, bar] = [1, 2];

// var
var [foo, bar] = [1, 2];

var foo, bar
[foo, bar] = [1, 2];
```

## Proposed syntax

This proposal is chapioning the expansion of the array destructuring syntax. The extra syntax will allow for individual variable declaration assignment on each destructured array element.

> The following is pseudocode outlining the main features of the proposed syntax

The type assigned in front of the destructuring can be considered the `default` type.

```typescript
const [foo, bar] = [1, 2, 3]; // Both foo & bar are `const`
```

Any declaration that's assigned as part of the destructuring, will take precedence over the default as it is the more specifc variable type.

```typescript
// - Example 1 -
// `const` foo, `let` bar, `const` ...rest
const [foo, let bar, ...rest] = [1, 2, 3, 4];

// - Example 2 -
// `const` foo, `let` bar, `var` baz
const [foo, let bar, var baz] = [1, 2, 3, 4];

// - Example 3 -
// `let` foo, `let` bar, `const` baz
let [foo, bar, const baz] = [1, 2, 3, 4];
```

The following code should behave identically to the code above:

```typescript
// - Example 1 -
// `const` foo, `let` bar, `const` ...rest
// TODO: Need to decide between those
[const foo, let bar, const ...rest] = [1, 2, 3, 4];
[const foo, let bar, ...const rest] = [1, 2, 3, 4];

// - Example 2 -
// `const` foo, `let` bar, `var` baz
[const foo, let bar, var baz] = [1, 2, 3, 4];
```

Either the `default` variable type must be used/declared, or each destructured element must have its own declaration. The following code should not run:

```typescript
// `var` _, `const` bar, `???` ...rest
[var _, const bar, ...rest] = [1, 2, 3, 4];
```