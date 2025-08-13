---
description: 
globs: 
alwaysApply: true
---
Always enable strict mode and related compiler options for maximum type safety.

When possible, a new codebase should always turn these strictness checks on.

```json
// tsconfig.json should include:
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

These flags help you maintain a clean codebase:

- `strict`: Enables all strict type checking options
- `noImplicitAny`: Error on expressions with an implied `any` type
- `noImplicitReturns`: Error when not all code paths return a value
- `noFallthroughCasesInSwitch`: Error on fallthrough cases in switch statements
- `noUncheckedIndexedAccess`: Add `undefined` to index signature results
- `exactOptionalPropertyTypes`: Differentiate between `undefined` and missing properties
- `noUnusedLocals`: Error on unused local variables
- `noUnusedParameters`: Error on unused function parameters

The more typed your program is, the more validation and tooling you'll get, meaning you'll run into fewer bugs as you code.