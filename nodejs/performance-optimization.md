---
description: 
globs: 
alwaysApply: true
---
Optimize Node.js applications for better performance and resource utilization.

Use connection pooling for databases:

```js
// BAD - creating new connections for each request
const getUser = async (id) => {
  const connection = await mysql.createConnection(dbConfig);
  const result = await connection.execute('SELECT * FROM users WHERE id = ?', [id]);
  await connection.end();
  return result;
};
```

```js
// GOOD - connection pooling
const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  database: 'myapp',
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
});

const getUser = async (id) => {
  const [rows] = await pool.execute('SELECT * FROM users WHERE id = ?', [id]);
  return rows;
};
```

Implement caching for frequently accessed data:

```js
import NodeCache from 'node-cache';

// GOOD - in-memory caching
const cache = new NodeCache({ stdTTL: 600 }); // 10 minutes TTL

const getCachedUser = async (userId) => {
  const cacheKey = `user:${userId}`;
  
  // Try cache first
  let user = cache.get(cacheKey);
  if (user) {
    return user;
  }
  
  // Fetch from database if not in cache
  user = await db.users.findById(userId);
  if (user) {
    cache.set(cacheKey, user);
  }
  
  return user;
};
```

Use streaming for large data processing:

```js
// BAD - loading all data into memory
const processLargeDataset = async () => {
  const allRecords = await db.query('SELECT * FROM large_table');
  return allRecords.map(record => processRecord(record));
};
```

```js
// GOOD - streaming data processing
import { Transform } from 'stream';
import { pipeline } from 'stream/promises';

const processLargeDataset = async () => {
  const queryStream = db.query('SELECT * FROM large_table').stream();
  
  const processTransform = new Transform({
    objectMode: true,
    transform(record, encoding, callback) {
      const processed = processRecord(record);
      callback(null, processed);
    },
  });
  
  const results = [];
  const collectTransform = new Transform({
    objectMode: true,
    transform(data, encoding, callback) {
      results.push(data);
      callback();
    },
  });
  
  await pipeline(queryStream, processTransform, collectTransform);
  return results;
};
```

Optimize JSON parsing and stringification:

```js
// GOOD - use streaming JSON parser for large payloads
import StreamingJsonParse from 'streaming-json-parse';

const parselargeJson = (stream) => {
  return new Promise((resolve, reject) => {
    const parser = new StreamingJsonParse();
    const results = [];
    
    parser.on('data', (data) => {
      results.push(data);
    });
    
    parser.on('end', () => resolve(results));
    parser.on('error', reject);
    
    stream.pipe(parser);
  });
};
```

Use worker threads for CPU-intensive tasks:

```js
// worker.js
import { parentPort, workerData } from 'worker_threads';

const performHeavyCalculation = (data) => {
  // CPU-intensive work here
  let result = 0;
  for (let i = 0; i < data.iterations; i++) {
    result += Math.sqrt(i);
  }
  return result;
};

const result = performHeavyCalculation(workerData);
parentPort.postMessage(result);
```

```js
// main.js
import { Worker } from 'worker_threads';

// GOOD - offload CPU-intensive work to worker threads
const runWorker = (data) => {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', { workerData: data });
    
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker stopped with exit code ${code}`));
      }
    });
  });
};
```

Implement proper logging levels:

```js
// GOOD - structured logging with levels
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// Only add console transport in development
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple()
  }));
}
```

Use compression middleware:

```js
import compression from 'compression';

// GOOD - compress responses
app.use(compression({
  filter: (req, res) => {
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  },
  level: 6, // Compression level (1-9)
  threshold: 1024, // Only compress if response is larger than 1KB
}));
```

Monitor and profile your application:

```js
// GOOD - performance monitoring
const performanceMonitor = {
  startTime: process.hrtime.bigint(),
  
  measureExecutionTime: (label, fn) => {
    return async (...args) => {
      const start = process.hrtime.bigint();
      try {
        const result = await fn(...args);
        const end = process.hrtime.bigint();
        const duration = Number(end - start) / 1000000; // Convert to milliseconds
        
        if (duration > 100) { // Log slow operations
          console.warn(`Slow operation ${label}: ${duration.toFixed(2)}ms`);
        }
        
        return result;
      } catch (error) {
        const end = process.hrtime.bigint();
        const duration = Number(end - start) / 1000000;
        console.error(`Failed operation ${label}: ${duration.toFixed(2)}ms`, error);
        throw error;
      }
    };
  },
};
```