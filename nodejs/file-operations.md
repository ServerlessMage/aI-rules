---
description: 
globs: 
alwaysApply: true
---
Use the promises-based fs API instead of callback-based or synchronous operations for better performance and error handling.

```js
// BAD - synchronous operations block the event loop
import fs from 'fs';

const data = fs.readFileSync('large-file.txt', 'utf8');
```

```js
// GOOD - asynchronous operations
import { readFile } from 'fs/promises';

const data = await readFile('large-file.txt', 'utf8');
```

Always specify encoding when working with text files:

```js
// BAD - returns Buffer by default
const data = await readFile('config.json');
```

```js
// GOOD - explicit encoding
const data = await readFile('config.json', 'utf8');
```

Use streams for large files to avoid memory issues:

```js
// BAD - loads entire file into memory
const processLargeFile = async () => {
  const data = await readFile('huge-file.txt', 'utf8');
  return data.split('\n').map(line => line.trim());
};
```

```js
// GOOD - streaming for large files
import { createReadStream } from 'fs';
import { createInterface } from 'readline';

const processLargeFile = async () => {
  const fileStream = createReadStream('huge-file.txt');
  const rl = createInterface({
    input: fileStream,
    crlfDelay: Infinity,
  });
  
  const lines = [];
  for await (const line of rl) {
    lines.push(line.trim());
  }
  return lines;
};
```

Always handle file operation errors:

```js
// GOOD - proper error handling
const safeReadFile = async (filePath) => {
  try {
    const data = await readFile(filePath, 'utf8');
    return { success: true, data };
  } catch (error) {
    if (error.code === 'ENOENT') {
      return { success: false, error: 'File not found' };
    }
    if (error.code === 'EACCES') {
      return { success: false, error: 'Permission denied' };
    }
    return { success: false, error: error.message };
  }
};
```

Use path.join() for cross-platform file paths:

```js
// BAD - platform-specific path separators
const filePath = './data/users/profile.json';
```

```js
// GOOD - cross-platform paths
import { join } from 'path';

const filePath = join('data', 'users', 'profile.json');
```

Check file existence before operations when necessary:

```js
import { access, constants } from 'fs/promises';

const fileExists = async (filePath) => {
  try {
    await access(filePath, constants.F_OK);
    return true;
  } catch {
    return false;
  }
};

const safeFileOperation = async (filePath) => {
  if (await fileExists(filePath)) {
    return await readFile(filePath, 'utf8');
  }
  throw new Error(`File ${filePath} does not exist`);
};
```

Use atomic operations for critical file writes:

```js
// GOOD - atomic write operation
import { writeFile, rename } from 'fs/promises';
import { join } from 'path';

const atomicWriteFile = async (filePath, data) => {
  const tempPath = `${filePath}.tmp`;
  try {
    await writeFile(tempPath, data, 'utf8');
    await rename(tempPath, filePath);
  } catch (error) {
    // Clean up temp file if it exists
    try {
      await unlink(tempPath);
    } catch {
      // Ignore cleanup errors
    }
    throw error;
  }
};
```