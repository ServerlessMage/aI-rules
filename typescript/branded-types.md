---
description: 
globs: 
alwaysApply: true
---
Use branded types to prevent primitive obsession and create type-safe identifiers that cannot be accidentally mixed up.

Branded types add a compile-time-only property to distinguish otherwise identical types:

```ts
// Create branded types for different ID types
type UserId = string & { readonly __brand: 'UserId' };
type ProductId = string & { readonly __brand: 'ProductId' };
type OrderId = string & { readonly __brand: 'OrderId' };

// Helper functions to create branded types
const createUserId = (id: string): UserId => id as UserId;
const createProductId = (id: string): ProductId => id as ProductId;
const createOrderId = (id: string): OrderId => id as OrderId;

// Usage
const userId = createUserId('user-123');
const productId = createProductId('prod-456');

// This prevents accidental mixing
const getUser = (id: UserId): User => { /* ... */ };

getUser(userId); // OK
getUser(productId); // Error - ProductId is not assignable to UserId
```

Use branded types for different units or formats:

```ts
// Different units
type Meters = number & { readonly __brand: 'Meters' };
type Feet = number & { readonly __brand: 'Feet' };

// Different formats
type EmailAddress = string & { readonly __brand: 'EmailAddress' };
type PhoneNumber = string & { readonly __brand: 'PhoneNumber' };
type URL = string & { readonly __brand: 'URL' };

// Validation functions that return branded types
const validateEmail = (input: string): EmailAddress | null => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(input) ? (input as EmailAddress) : null;
};

const validateUrl = (input: string): URL | null => {
  try {
    new globalThis.URL(input);
    return input as URL;
  } catch {
    return null;
  }
};
```

Create a generic brand utility:

```ts
// Generic brand utility
type Brand<T, TBrand extends string> = T & { readonly __brand: TBrand };

// Usage with the utility
type UserId = Brand<string, 'UserId'>;
type Timestamp = Brand<number, 'Timestamp'>;
type SafeHtml = Brand<string, 'SafeHtml'>;

// Factory functions
const createTimestamp = (): Timestamp => Date.now() as Timestamp;
const sanitizeHtml = (html: string): SafeHtml => {
  // Sanitization logic here
  return html as SafeHtml;
};
```

Use branded types in APIs to prevent parameter confusion:

```ts
// Without branded types - easy to mix up parameters
const transferMoney = (fromAccount: string, toAccount: string, amount: number) => {
  // Easy to accidentally swap fromAccount and toAccount
};

// With branded types - compile-time safety
type AccountId = Brand<string, 'AccountId'>;
type Amount = Brand<number, 'Amount'>;

const transferMoney = (
  fromAccount: AccountId,
  toAccount: AccountId,
  amount: Amount,
) => {
  // Cannot accidentally pass wrong types
};
```

Branded types are especially useful for:
- Database IDs that shouldn't be mixed up
- Different units of measurement
- Validated strings (emails, URLs, etc.)
- Security-sensitive values (tokens, passwords)
- Domain-specific values that need type safety

The brand property is erased at runtime, so there's no performance cost.