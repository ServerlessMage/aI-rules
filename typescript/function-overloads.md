---
description: 
globs: 
alwaysApply: true
---
Use function overloads to provide better type safety when a function can accept different parameter types and return different types based on the input.

Function overloads allow you to define multiple signatures for the same function:

```ts
// Function overloads - define multiple signatures
function processData(data: string): string;
function processData(data: number): number;
function processData(data: boolean): boolean;

// Implementation signature - must handle all overload cases
function processData(data: string | number | boolean): string | number | boolean {
  if (typeof data === 'string') {
    return data.toUpperCase();
  }
  if (typeof data === 'number') {
    return data * 2;
  }
  return !data;
}

// Usage - TypeScript knows the exact return type
const result1 = processData('hello'); // string
const result2 = processData(42); // number
const result3 = processData(true); // boolean
```

Use overloads for different parameter combinations:

```ts
// Different parameter combinations
function createUser(name: string): User;
function createUser(name: string, email: string): User;
function createUser(config: UserConfig): User;

function createUser(
  nameOrConfig: string | UserConfig,
  email?: string,
): User {
  if (typeof nameOrConfig === 'string') {
    return {
      id: generateId(),
      name: nameOrConfig,
      email: email || `${nameOrConfig}@example.com`,
    };
  }
  
  return {
    id: generateId(),
    ...nameOrConfig,
  };
}
```

Prefer discriminated unions over complex overloads when possible:

```ts
// GOOD - discriminated union is often clearer
type ProcessRequest = 
  | { type: 'string'; data: string }
  | { type: 'number'; data: number }
  | { type: 'boolean'; data: boolean };

const processData = (request: ProcessRequest): string | number | boolean => {
  switch (request.type) {
    case 'string':
      return request.data.toUpperCase();
    case 'number':
      return request.data * 2;
    case 'boolean':
      return !request.data;
  }
};
```

Use overloads sparingly - only when they significantly improve the developer experience:

```ts
// Good use case - DOM element selection
function querySelector(selector: 'input'): HTMLInputElement | null;
function querySelector(selector: 'button'): HTMLButtonElement | null;
function querySelector(selector: string): Element | null;

function querySelector(selector: string): Element | null {
  return document.querySelector(selector);
}

// Now TypeScript knows the specific element type
const input = querySelector('input'); // HTMLInputElement | null
const button = querySelector('button'); // HTMLButtonElement | null
```