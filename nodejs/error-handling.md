---
description: 
globs: 
alwaysApply: true
---
Always handle errors properly in Node.js applications. Use async/await with try-catch blocks for better error handling.

```js
// BAD - unhandled promise rejection
const fetchData = async () => {
  const response = await fetch('/api/data');
  return response.json();
};
```

```js
// GOOD - proper error handling
const fetchData = async (): Promise<ApiResponse | null> => {
  try {
    const response = await fetch('/api/data');
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    console.error('Failed to fetch data:', error);
    return null;
  }
};
```

For callback-based APIs, always check for errors first:

```js
// BAD - not checking error
fs.readFile('file.txt', (err, data) => {
  console.log(data.toString());
});
```

```js
// GOOD - error-first callback pattern
fs.readFile('file.txt', (err, data) => {
  if (err) {
    console.error('Error reading file:', err);
    return;
  }
  console.log(data.toString());
});
```

Use custom error classes for different error types:

```js
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}

class DatabaseError extends Error {
  constructor(message, query) {
    super(message);
    this.name = 'DatabaseError';
    this.query = query;
  }
}
```

Always handle uncaught exceptions and unhandled rejections:

```js
process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error);
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  process.exit(1);
});
```