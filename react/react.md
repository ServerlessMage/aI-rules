---
description: General guidelines for writing modern React components and applications.
globs: ["*.js", "*.jsx", "*.ts", "*.tsx"]
alwaysApply: true
---

This file provides general React guidelines. For specific patterns and detailed examples, see the individual standard files in this directory.

## Quick Reference

### Components
- Use functional components with hooks
- PascalCase for component names
- Keep components small and focused
- Use TypeScript interfaces for props

### State Management
- `useState` for simple local state
- `useReducer` for complex state logic
- Context API for moderate shared state
- External libraries for complex global state

### Performance
- Use `React.memo` for expensive components
- `useCallback` and `useMemo` for optimization
- Code splitting with `React.lazy`

### Accessibility
- Semantic HTML elements
- Proper ARIA attributes
- Keyboard navigation support

For detailed examples and specific patterns, refer to the individual standard files in this directory.
