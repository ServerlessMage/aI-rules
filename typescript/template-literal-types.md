---
description: 
globs: 
alwaysApply: true
---
Use template literal types to create precise string patterns and improve type safety for string-based APIs.

Template literal types allow you to create types based on string patterns:

```ts
// Basic template literal types
type Greeting = `Hello, ${string}!`;
type ApiEndpoint = `/api/${string}`;
type CssClass = `${string}-${string}`;

// Usage
const greeting: Greeting = 'Hello, World!'; // OK
const endpoint: ApiEndpoint = '/api/users'; // OK
const className: CssClass = 'btn-primary'; // OK
```

Combine with union types for specific patterns:

```ts
// HTTP methods with paths
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type ApiRoute = 'users' | 'products' | 'orders';
type ApiCall = `${HttpMethod} /api/${ApiRoute}`;

// Type: "GET /api/users" | "POST /api/users" | ... etc
const apiCall: ApiCall = 'GET /api/users'; // OK
```

Use for CSS-in-JS and styling:

```ts
// CSS properties with units
type CssUnit = 'px' | 'rem' | 'em' | '%' | 'vh' | 'vw';
type CssSize = `${number}${CssUnit}`;

// Color formats
type HexColor = `#${string}`;
type RgbColor = `rgb(${number}, ${number}, ${number})`;
type CssColor = HexColor | RgbColor | 'transparent';

// Usage in styled components
const styles = {
  width: '100px' as CssSize,
  color: '#ff0000' as HexColor,
};
```

Create event name patterns:

```ts
// Event naming convention
type EventCategory = 'user' | 'product' | 'order';
type EventAction = 'created' | 'updated' | 'deleted';
type EventName = `${EventCategory}.${EventAction}`;

// Type: "user.created" | "user.updated" | "user.deleted" | ...
const handleEvent = (eventName: EventName) => {
  // TypeScript knows all possible event names
};
```

Use with mapped types for object keys:

```ts
// Create getter/setter method names
type User = {
  name: string;
  email: string;
  age: number;
};

type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

// Results in:
// {
//   getName: () => string;
//   getEmail: () => string;
//   getAge: () => number;
//   setName: (value: string) => void;
//   setEmail: (value: string) => void;
//   setAge: (value: number) => void;
// }
type UserAccessors = Getters<User> & Setters<User>;
```

Use for database table/column naming:

```ts
// Database naming patterns
type TableName = 'users' | 'products' | 'orders';
type ColumnSuffix = 'id' | 'name' | 'created_at' | 'updated_at';
type ColumnName<T extends TableName> = `${T}.${ColumnSuffix}`;

// SQL query builder with type safety
const buildQuery = <T extends TableName>(
  table: T,
  columns: ColumnName<T>[],
) => {
  return `SELECT ${columns.join(', ')} FROM ${table}`;
};

// Usage with autocomplete and type checking
const query = buildQuery('users', ['users.id', 'users.name']);
```

Template literal types are powerful for:
- API endpoint patterns
- CSS class naming conventions
- Event naming systems
- Database query builders
- Configuration key patterns
- File path patterns

Use them when you need precise string patterns with compile-time validation.