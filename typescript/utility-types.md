---
description: 
globs: 
alwaysApply: true
---
Leverage TypeScript's built-in utility types for type transformations instead of manually recreating them.

Common utility types to use:

```ts
// Partial - makes all properties optional
type PartialUser = Partial<User>;

// Required - makes all properties required
type RequiredConfig = Required<Config>;

// Pick - select specific properties
type UserEmail = Pick<User, 'email'>;
type UserBasics = Pick<User, 'id' | 'name' | 'email'>;

// Omit - exclude specific properties
type UserWithoutId = Omit<User, 'id'>;
type PublicUser = Omit<User, 'password' | 'internalId'>;

// Record - create object type with specific keys and values
type StatusMap = Record<string, boolean>;
type UserRoles = Record<'admin' | 'user' | 'guest', Permission[]>;

// Extract - extract types from union
type StringKeys<T> = Extract<keyof T, string>;

// Exclude - exclude types from union
type NonNullable<T> = Exclude<T, null | undefined>;

// ReturnType - get function return type
type ApiResponse = ReturnType<typeof fetchUser>;

// Parameters - get function parameter types
type FetchUserParams = Parameters<typeof fetchUser>;
```

Combine utility types for complex transformations:

```ts
// Create update type that makes all fields optional except id
type UpdateUser = Partial<Omit<User, 'id'>> & Pick<User, 'id'>;

// Create a type with readonly properties from another type
type ReadonlyUser = Readonly<User>;

// Create API response wrapper
type ApiWrapper<T> = {
  readonly data: T;
  readonly status: 'success' | 'error';
  readonly message?: string;
};

type UserResponse = ApiWrapper<User>;
```

Prefer utility types over manual type definitions:

```ts
// BAD - manually recreating Partial
type PartialUser = {
  id?: string;
  name?: string;
  email?: string;
};

// GOOD - using built-in utility
type PartialUser = Partial<User>;
```