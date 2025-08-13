---
description: 
globs: 
alwaysApply: true
---
Prefer type guards over type assertions for better runtime safety.

Type assertions bypass TypeScript's type checking and can lead to runtime errors if the assertion is incorrect.

```ts
// BAD - type assertion without validation
const user = data as User;
user.email.toLowerCase(); // Runtime error if data doesn't have email
```

```ts
// GOOD - type guard with validation
const isUser = (data: unknown): data is User => {
  return (
    typeof data === 'object' &&
    data !== null &&
    'id' in data &&
    'email' in data &&
    typeof (data as any).id === 'string' &&
    typeof (data as any).email === 'string'
  );
};

if (isUser(data)) {
  // data is now safely typed as User
  data.email.toLowerCase(); // Safe
}
```

When type assertions are necessary, use them sparingly and document why:

```ts
// OK - when you have more information than TypeScript
const element = document.getElementById('my-input') as HTMLInputElement;
// We know this element exists and is an input from the HTML

// OK - in generic functions where TypeScript can't infer correctly
return result as TReturn; // Documented in generic function context
```

Use `satisfies` operator when you want to check assignability without widening:

```ts
// GOOD - satisfies ensures type safety without widening
const config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
} satisfies ApiConfig;

// config.apiUrl is still string literal, not just string
```