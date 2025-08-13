---
description: 
globs: 
alwaysApply: true
---
Think carefully before implementing code that throws errors.

If a thrown error produces a desirable outcome in the system, go for it. For instance, throwing a custom error inside a backend framework's request handler.

However, for code that you would need a manual try catch for, consider using a result type instead:

```ts
type Result<T, E extends Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };
```

Define specific error types for better error handling:

```ts
class ValidationError extends Error {
  constructor(
    message: string,
    public readonly field: string,
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

class NetworkError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
  ) {
    super(message);
    this.name = 'NetworkError';
  }
}

// Use specific error types in Result
type ValidationResult<T> = Result<T, ValidationError>;
type ApiResult<T> = Result<T, NetworkError>;
```

For example, when parsing JSON:

```ts
const parseJson = (
  input: string,
): Result<unknown, ValidationError> => {
  try {
    return { ok: true, value: JSON.parse(input) };
  } catch (error) {
    return { 
      ok: false, 
      error: new ValidationError('Invalid JSON format', 'input') 
    };
  }
};
```

This way you can handle the error in the caller:

```ts
const result = parseJson('{"name": "John"}');

if (result.ok) {
  console.log(result.value);
} else {
  console.error(`${result.error.name}: ${result.error.message}`);
}
```
