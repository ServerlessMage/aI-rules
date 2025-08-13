---
description: 
globs: 
alwaysApply: true
---
Use `ReadonlyArray<T>` or `readonly T[]` when the array should not be mutated, and regular `T[]` when mutation is expected.

Default to readonly arrays unless mutation is specifically needed:

```ts
// GOOD - readonly by default
const processItems = (items: readonly string[]): string[] => {
  // items.push('new'); // Error - cannot mutate
  return items.map(item => item.toUpperCase());
};

// GOOD - mutable when needed
const addItem = (items: string[], newItem: string): void => {
  items.push(newItem); // OK - mutation expected
};
```

Use `ReadonlyArray<T>` for better readability with complex types:

```ts
// Prefer ReadonlyArray for complex types
type UserList = ReadonlyArray<User>;
type NestedData = ReadonlyArray<ReadonlyArray<string>>;

// Use readonly modifier for simple arrays
const names: readonly string[] = ['Alice', 'Bob'];
const numbers: readonly number[] = [1, 2, 3];
```

Return readonly arrays from functions that shouldn't expose mutability:

```ts
// GOOD - prevents accidental mutation of internal state
class UserService {
  private users: User[] = [];

  getUsers(): readonly User[] {
    return this.users; // Readonly view of internal array
  }

  addUser(user: User): void {
    this.users.push(user); // Internal mutation is OK
  }
}
```

Use tuple types for fixed-length arrays:

```ts
// Tuple for fixed structure
type Coordinates = readonly [number, number];
type RGB = readonly [number, number, number];

// Function that returns a tuple
const getCoordinates = (): Coordinates => [10, 20];

// Destructuring works as expected
const [x, y] = getCoordinates();
```

Consider using `as const` for literal arrays:

```ts
// Creates readonly tuple with literal types
const colors = ['red', 'green', 'blue'] as const;
// Type: readonly ["red", "green", "blue"]

// Useful for creating union types
type Color = typeof colors[number]; // "red" | "green" | "blue"
```

Array vs ReadonlyArray guidelines:

- Use `readonly T[]` or `ReadonlyArray<T>` by default
- Use `T[]` only when you need to mutate the array
- Return readonly arrays from public APIs
- Use tuples for fixed-length, structured data
- Use `as const` for literal arrays that should be immutable