---
description: 
globs: 
alwaysApply: true
---
Use ES modules (import/export) over CommonJS (require/module.exports) for new Node.js projects.

Ensure your package.json has `"type": "module"` or use `.mjs` file extensions.

```js
// BAD - CommonJS
const fs = require('fs');
const path = require('path');
const { promisify } = require('util');

module.exports = {
  readFileAsync: promisify(fs.readFile),
};
```

```js
// GOOD - ES modules
import fs from 'fs/promises';
import path from 'path';
import { fileURLToPath } from 'url';

export const readFileAsync = fs.readFile;
```

Use named imports for better tree-shaking and clarity:

```js
// BAD - default import when named imports are available
import fs from 'fs';
import util from 'util';
```

```js
// GOOD - named imports
import { readFile, writeFile } from 'fs/promises';
import { promisify } from 'util';
```

For Node.js built-in modules, use the `node:` prefix for clarity:

```js
// GOOD - explicit Node.js built-ins
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';
import { createServer } from 'node:http';
```

Group imports logically:

```js
// Node.js built-ins first
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';

// Third-party packages
import express from 'express';
import { z } from 'zod';

// Local modules
import { config } from './config.js';
import { logger } from './utils/logger.js';
```

Use dynamic imports for conditional loading:

```js
// GOOD - dynamic import for conditional modules
const loadDevTools = async () => {
  if (process.env.NODE_ENV === 'development') {
    const { setupDevTools } = await import('./dev-tools.js');
    return setupDevTools();
  }
  return null;
};
```

Always include file extensions in relative imports:

```js
// BAD - missing file extension
import { helper } from './utils/helper';
```

```js
// GOOD - explicit file extension
import { helper } from './utils/helper.js';
```