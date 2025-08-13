---
description: 
globs: 
alwaysApply: true
---
Always validate and sanitize user input to prevent security vulnerabilities.

```js
// BAD - direct use of user input
const getUserData = (userId) => {
  return db.query(`SELECT * FROM users WHERE id = ${userId}`);
};
```

```js
// GOOD - parameterized queries
const getUserData = (userId) => {
  return db.query('SELECT * FROM users WHERE id = ?', [userId]);
};
```

Use input validation libraries:

```js
import { z } from 'zod';

// GOOD - schema validation
const userSchema = z.object({
  email: z.string().email(),
  age: z.number().min(0).max(120),
  name: z.string().min(1).max(100),
});

const createUser = async (userData) => {
  try {
    const validatedData = userSchema.parse(userData);
    return await db.users.create(validatedData);
  } catch (error) {
    throw new ValidationError('Invalid user data', error.errors);
  }
};
```

Never expose sensitive information in error messages:

```js
// BAD - exposes internal details
app.use((error, req, res, next) => {
  res.status(500).json({ error: error.message, stack: error.stack });
});
```

```js
// GOOD - sanitized error responses
app.use((error, req, res, next) => {
  const isDevelopment = process.env.NODE_ENV === 'development';
  
  res.status(error.status || 500).json({
    message: error.message,
    ...(isDevelopment && { stack: error.stack }),
  });
});
```

Use helmet for security headers:

```js
import helmet from 'helmet';

// GOOD - security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true,
  },
}));
```

Implement rate limiting:

```js
import rateLimit from 'express-rate-limit';

// GOOD - rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP',
  standardHeaders: true,
  legacyHeaders: false,
});

app.use('/api/', limiter);
```

Sanitize file paths to prevent directory traversal:

```js
import path from 'path';

// BAD - vulnerable to directory traversal
const serveFile = (filename) => {
  return fs.readFile(`./uploads/${filename}`);
};
```

```js
// GOOD - sanitized file paths
const serveFile = (filename) => {
  const safePath = path.normalize(filename).replace(/^(\.\.[\/\\])+/, '');
  const fullPath = path.join('./uploads', safePath);
  
  // Ensure the resolved path is within the uploads directory
  if (!fullPath.startsWith(path.resolve('./uploads'))) {
    throw new Error('Invalid file path');
  }
  
  return fs.readFile(fullPath);
};
```

Use HTTPS in production:

```js
import https from 'https';
import fs from 'fs';

// GOOD - HTTPS in production
if (process.env.NODE_ENV === 'production') {
  const options = {
    key: fs.readFileSync('path/to/private-key.pem'),
    cert: fs.readFileSync('path/to/certificate.pem'),
  };
  
  https.createServer(options, app).listen(443, () => {
    console.log('HTTPS Server running on port 443');
  });
} else {
  app.listen(3000, () => {
    console.log('HTTP Server running on port 3000');
  });
}
```

Implement proper authentication and authorization:

```js
import jwt from 'jsonwebtoken';

// GOOD - JWT middleware with proper validation
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) {
    return res.sendStatus(401);
  }
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.sendStatus(403);
    req.user = user;
    next();
  });
};

// Use strong JWT secrets
const generateToken = (user) => {
  return jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET,
    { expiresIn: '1h', issuer: 'your-app-name' }
  );
};
```

Never log sensitive information:

```js
// BAD - logging sensitive data
console.log('User login:', { email, password, token });
```

```js
// GOOD - sanitized logging
console.log('User login:', { 
  email, 
  userId: user.id,
  timestamp: new Date().toISOString()
});
```