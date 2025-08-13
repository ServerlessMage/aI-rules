---
description: 
globs: 
alwaysApply: true
---
- Use kebab-case for file names (e.g., `my-component.ts`)
- Use camelCase for variables and function names (e.g., `myVariable`, `myFunction()`)
- Use UpperCamelCase (PascalCase) for classes, types, and interfaces (e.g., `MyClass`, `MyInterface`)
- Use ALL_CAPS for constants and enum values (e.g., `MAX_COUNT`, `Color.RED`)
- Inside generic types, functions or classes, prefix type parameters with `T` (e.g., `TKey`, `TValue`)
- For single type parameter, plain `T` is acceptable
- For multiple parameters, use descriptive names with `T` prefix

```ts
// Single parameter - plain T is fine
type Container<T> = { value: T };

// Multiple parameters - use descriptive names
type RecordOfArrays<TItem> = Record<string, TItem[]>;
type Mapper<TInput, TOutput> = (input: TInput) => TOutput;
type Repository<TEntity, TKey> = {
  findById(id: TKey): Promise<TEntity | null>;
};
```
