---
description: 
globs: 
alwaysApply: true
---
Always validate and provide defaults for environment variables. Never assume they exist or have valid values.

```js
// BAD - direct access without validation
const port = process.env.PORT;
const dbUrl = process.env.DATABASE_URL;
```

```js
// GOOD - validation with defaults and type conversion
const port = Number(process.env.PORT) || 3000;
const dbUrl = process.env.DATABASE_URL || 'sqlite://./dev.db';

if (!process.env.DATABASE_URL && process.env.NODE_ENV === 'production') {
  throw new Error('DATABASE_URL is required in production');
}
```

Use a configuration module to centralize environment variable handling:

```js
// config.js
const config = {
  port: Number(process.env.PORT) || 3000,
  nodeEnv: process.env.NODE_ENV || 'development',
  database: {
    url: process.env.DATABASE_URL || 'sqlite://./dev.db',
    maxConnections: Number(process.env.DB_MAX_CONNECTIONS) || 10,
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '24h',
  },
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379',
  },
};

// Validate required environment variables
const requiredEnvVars = ['JWT_SECRET'];
if (config.nodeEnv === 'production') {
  requiredEnvVars.push('DATABASE_URL');
}

for (const envVar of requiredEnvVars) {
  if (!process.env[envVar]) {
    throw new Error(`Required environment variable ${envVar} is not set`);
  }
}

export { config };
```

Use environment-specific validation:

```js
// GOOD - environment-specific requirements
const validateConfig = () => {
  const errors = [];
  
  if (process.env.NODE_ENV === 'production') {
    if (!process.env.DATABASE_URL) {
      errors.push('DATABASE_URL is required in production');
    }
    if (!process.env.JWT_SECRET || process.env.JWT_SECRET.length < 32) {
      errors.push('JWT_SECRET must be at least 32 characters in production');
    }
  }
  
  if (process.env.PORT && isNaN(Number(process.env.PORT))) {
    errors.push('PORT must be a valid number');
  }
  
  if (errors.length > 0) {
    throw new Error(`Configuration errors:\n${errors.join('\n')}`);
  }
};
```

Use dotenv for development but not in production:

```js
// GOOD - conditional dotenv loading
if (process.env.NODE_ENV !== 'production') {
  const { config } = await import('dotenv');
  config();
}
```

Create typed environment configuration with validation libraries:

```js
// Using zod for validation
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  LOG_LEVEL: z.enum(['error', 'warn', 'info', 'debug']).default('info'),
});

export const env = envSchema.parse(process.env);
```

Never log sensitive environment variables:

```js
// BAD - might log sensitive data
console.log('Environment:', process.env);
```

```js
// GOOD - selective logging
console.log('Environment:', {
  NODE_ENV: process.env.NODE_ENV,
  PORT: process.env.PORT,
  // Never log secrets, tokens, passwords, etc.
});
```