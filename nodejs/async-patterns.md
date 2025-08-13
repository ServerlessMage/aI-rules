---
description: 
globs: 
alwaysApply: true
---
Prefer async/await over callbacks and raw Promises for better readability and error handling.

```js
// BAD - callback hell
const processData = (callback) => {
  fs.readFile('input.txt', (err, data) => {
    if (err) return callback(err);
    
    processText(data, (err, processed) => {
      if (err) return callback(err);
      
      fs.writeFile('output.txt', processed, (err) => {
        if (err) return callback(err);
        callback(null, 'Success');
      });
    });
  });
};
```

```js
// GOOD - async/await
const processData = async (): Promise<string> => {
  try {
    const data = await fs.promises.readFile('input.txt', 'utf8');
    const processed = await processText(data);
    await fs.promises.writeFile('output.txt', processed);
    return 'Success';
  } catch (error) {
    throw new Error(`Processing failed: ${error.message}`);
  }
};
```

Use Promise.all() for concurrent operations:

```js
// BAD - sequential execution
const fetchAllData = async () => {
  const users = await fetchUsers();
  const products = await fetchProducts();
  const orders = await fetchOrders();
  return { users, products, orders };
};
```

```js
// GOOD - concurrent execution
const fetchAllData = async () => {
  const [users, products, orders] = await Promise.all([
    fetchUsers(),
    fetchProducts(),
    fetchOrders(),
  ]);
  return { users, products, orders };
};
```

Use Promise.allSettled() when you want all promises to complete regardless of failures:

```js
const fetchDataWithFallbacks = async () => {
  const results = await Promise.allSettled([
    fetchPrimaryData(),
    fetchSecondaryData(),
    fetchTertiaryData(),
  ]);
  
  return results
    .filter(result => result.status === 'fulfilled')
    .map(result => result.value);
};
```

Avoid mixing async/await with .then():

```js
// BAD - mixing patterns
const badExample = async () => {
  const data = await fetchData();
  return processData(data).then(result => result.value);
};
```

```js
// GOOD - consistent async/await
const goodExample = async () => {
  const data = await fetchData();
  const result = await processData(data);
  return result.value;
};
```