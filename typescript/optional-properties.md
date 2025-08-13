---
description: 
globs: 
alwaysApply: true
---
Use optional properties extremely sparingly. Only use them when the property is truly optional, and consider whether bugs may be caused by a failure to pass the property.

In the example below we always want to pass user ID to `AuthOptions`. This is because if we forget to pass it somewhere in the code base, it will cause our function to be not authenticated.

```ts
// BAD
type AuthOptions = {
  userId?: string;
};

const func = (options: AuthOptions) => {
  const userId = options.userId;
};
```

```ts
// GOOD
type AuthOptions = {
  userId: string | undefined;
};

const func = (options: AuthOptions) => {
  const userId = options.userId;
};
```

Optional properties ARE appropriate for:
- Configuration objects with sensible defaults
- Partial updates to existing objects
- API responses where fields may be missing

```ts
// OK - partial update
type UpdateUserRequest = {
  readonly id: string;
  readonly name?: string; // Optional for partial update
  readonly email?: string; // Optional for partial update
};

// OK - configuration with defaults
type ApiConfig = {
  readonly baseUrl: string;
  readonly timeout?: number; // Has sensible default
  readonly retries?: number; // Has sensible default
};
```
