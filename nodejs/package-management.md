---
description: 
globs: 
alwaysApply: true
---
Use modern package management practices for better dependency management and security.

Always use exact versions for critical dependencies in production:

```json
// BAD - loose version constraints
{
  "dependencies": {
    "express": "^4.18.0",
    "lodash": "~4.17.0"
  }
}
```

```json
// GOOD - exact versions for critical dependencies
{
  "dependencies": {
    "express": "4.18.2",
    "lodash": "4.17.21"
  }
}
```

Use package-lock.json or yarn.lock and commit them:

```bash
# GOOD - always commit lock files
git add package-lock.json
git commit -m "Update dependencies"
```

Separate development and production dependencies:

```json
// GOOD - proper dependency separation
{
  "dependencies": {
    "express": "4.18.2",
    "helmet": "6.0.1"
  },
  "devDependencies": {
    "@types/node": "18.15.0",
    "nodemon": "2.0.20",
    "jest": "29.5.0"
  }
}
```

Use npm scripts for common tasks:

```json
// GOOD - comprehensive npm scripts
{
  "scripts": {
    "start": "node dist/index.js",
    "dev": "nodemon src/index.js",
    "build": "tsc",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/**/*.js",
    "lint:fix": "eslint src/**/*.js --fix",
    "security:audit": "npm audit",
    "security:fix": "npm audit fix",
    "clean": "rm -rf dist node_modules",
    "prestart": "npm run build"
  }
}
```

Regularly audit and update dependencies:

```bash
# GOOD - regular security audits
npm audit
npm audit fix

# Update dependencies safely
npm update
npm outdated
```

Use .npmrc for project-specific configuration:

```ini
# .npmrc - project configuration
save-exact=true
engine-strict=true
fund=false
audit-level=moderate
```

Specify Node.js version requirements:

```json
// GOOD - specify engine requirements
{
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=8.0.0"
  }
}
```

Use private registries for internal packages:

```json
// GOOD - scoped packages with private registry
{
  "name": "@company/my-package",
  "publishConfig": {
    "registry": "https://npm.company.com"
  }
}
```

Implement proper package.json metadata:

```json
// GOOD - comprehensive package.json
{
  "name": "my-app",
  "version": "1.0.0",
  "description": "A Node.js application",
  "main": "dist/index.js",
  "type": "module",
  "author": "Your Name <email@example.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/username/repo.git"
  },
  "bugs": {
    "url": "https://github.com/username/repo/issues"
  },
  "homepage": "https://github.com/username/repo#readme",
  "keywords": ["node", "express", "api"],
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ]
}
```

Use workspaces for monorepos:

```json
// GOOD - workspace configuration
{
  "name": "my-monorepo",
  "workspaces": [
    "packages/*",
    "apps/*"
  ],
  "scripts": {
    "build:all": "npm run build --workspaces",
    "test:all": "npm run test --workspaces",
    "lint:all": "npm run lint --workspaces"
  }
}
```

Implement dependency validation:

```js
// GOOD - validate critical dependencies at startup
const validateDependencies = () => {
  const requiredPackages = ['express', 'helmet', 'cors'];
  const missingPackages = [];
  
  for (const pkg of requiredPackages) {
    try {
      require.resolve(pkg);
    } catch (error) {
      missingPackages.push(pkg);
    }
  }
  
  if (missingPackages.length > 0) {
    throw new Error(`Missing required packages: ${missingPackages.join(', ')}`);
  }
};

// Run validation at startup
validateDependencies();
```

Use semantic versioning properly:

```json
// GOOD - semantic versioning
{
  "version": "1.2.3",
  "scripts": {
    "version:patch": "npm version patch",
    "version:minor": "npm version minor",
    "version:major": "npm version major"
  }
}
```