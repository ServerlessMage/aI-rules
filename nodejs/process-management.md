---
description: 
globs: 
alwaysApply: true
---
Handle process lifecycle events properly for graceful shutdowns and proper resource cleanup.

```js
// GOOD - graceful shutdown handling
const gracefulShutdown = (server) => {
  const shutdown = (signal) => {
    console.log(`Received ${signal}. Starting graceful shutdown...`);
    
    server.close(() => {
      console.log('HTTP server closed.');
      
      // Close database connections, cleanup resources, etc.
      process.exit(0);
    });
    
    // Force shutdown after timeout
    setTimeout(() => {
      console.error('Could not close connections in time, forcefully shutting down');
      process.exit(1);
    }, 10000);
  };
  
  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
};
```

Always handle uncaught exceptions and unhandled rejections:

```js
// GOOD - global error handling
process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error);
  // Log to external service, cleanup, then exit
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  // Log to external service, cleanup, then exit
  process.exit(1);
});
```

Use process.env.NODE_ENV to control behavior:

```js
// GOOD - environment-based configuration
const isDevelopment = process.env.NODE_ENV === 'development';
const isProduction = process.env.NODE_ENV === 'production';
const isTest = process.env.NODE_ENV === 'test';

if (isDevelopment) {
  // Enable debug logging, hot reloading, etc.
}

if (isProduction) {
  // Enable performance optimizations, error reporting, etc.
}
```

Handle worker processes and clustering properly:

```js
import cluster from 'cluster';
import { cpus } from 'os';

if (cluster.isPrimary) {
  const numCPUs = cpus().length;
  
  console.log(`Primary ${process.pid} is running`);
  
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    // Restart worker
    cluster.fork();
  });
} else {
  // Worker process - start your server here
  startServer();
  console.log(`Worker ${process.pid} started`);
}
```

Use proper exit codes:

```js
// GOOD - meaningful exit codes
const ExitCodes = {
  SUCCESS: 0,
  GENERAL_ERROR: 1,
  INVALID_ARGUMENT: 2,
  CONFIG_ERROR: 78,
  CANNOT_EXECUTE: 126,
};

// Usage
if (!config.isValid()) {
  console.error('Invalid configuration');
  process.exit(ExitCodes.CONFIG_ERROR);
}
```

Monitor memory usage and handle memory leaks:

```js
// GOOD - memory monitoring
const monitorMemory = () => {
  const usage = process.memoryUsage();
  console.log({
    rss: `${Math.round(usage.rss / 1024 / 1024)} MB`,
    heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)} MB`,
    heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)} MB`,
    external: `${Math.round(usage.external / 1024 / 1024)} MB`,
  });
  
  // Alert if memory usage is too high
  if (usage.heapUsed > 500 * 1024 * 1024) { // 500MB
    console.warn('High memory usage detected');
  }
};

// Monitor every 30 seconds in production
if (process.env.NODE_ENV === 'production') {
  setInterval(monitorMemory, 30000);
}
```

Handle process arguments properly:

```js
// GOOD - argument parsing
const parseArgs = () => {
  const args = process.argv.slice(2);
  const options = {};
  
  for (let i = 0; i < args.length; i++) {
    if (args[i].startsWith('--')) {
      const key = args[i].slice(2);
      const value = args[i + 1];
      options[key] = value;
      i++; // Skip next argument as it's the value
    }
  }
  
  return options;
};

const options = parseArgs();
console.log('Options:', options);
```