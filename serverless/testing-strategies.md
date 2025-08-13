# Serverless Testing Strategies

## Testing Pyramid

### Unit Tests
```typescript
// tests/unit/services/user.service.test.ts
import { UserService } from '@/shared/services/user.service';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
import { mockClient } from 'aws-sdk-client-mock';
import { GetCommand, PutCommand, QueryCommand } from '@aws-sdk/lib-dynamodb';

const dynamoMock = mockClient(DynamoDBDocumentClient);

describe('UserService', () => {
  let userService: UserService;
  
  beforeEach(() => {
    dynamoMock.reset();
    userService = new UserService();
  });
  
  describe('getById', () => {
    it('should return user when found', async () => {
      const mockUser = {
        id: '123',
        email: 'test@example.com',
        name: 'Test User',
        createdAt: '2023-01-01T00:00:00.000Z'
      };
      
      dynamoMock.on(GetCommand).resolves({
        Item: mockUser
      });
      
      const result = await userService.getById('123');
      
      expect(result).toEqual(mockUser);
      expect(dynamoMock.commandCalls(GetCommand)).toHaveLength(1);
    });
    
    it('should return null when user not found', async () => {
      dynamoMock.on(GetCommand).resolves({});
      
      const result = await userService.getById('nonexistent');
      
      expect(result).toBeNull();
    });
    
    it('should handle DynamoDB errors', async () => {
      dynamoMock.on(GetCommand).rejects(new Error('DynamoDB error'));
      
      await expect(userService.getById('123')).rejects.toThrow('DynamoDB error');
    });
  });
  
  describe('create', () => {
    it('should create user successfully', async () => {
      const createUserRequest = {
        email: 'new@example.com',
        name: 'New User',
        password: 'password123'
      };
      
      // Mock email check (user doesn't exist)
      dynamoMock.on(QueryCommand).resolves({ Items: [] });
      
      // Mock successful creation
      dynamoMock.on(PutCommand).resolves({});
      
      const result = await userService.create(createUserRequest);
      
      expect(result).toMatchObject({
        email: createUserRequest.email,
        name: createUserRequest.name
      });
      expect(result.id).toBeDefined();
      expect(result.createdAt).toBeDefined();
      expect(result.passwordHash).toBeUndefined(); // Should not be in response
    });
    
    it('should throw ConflictError when user already exists', async () => {
      const createUserRequest = {
        email: 'existing@example.com',
        name: 'Existing User',
        password: 'password123'
      };
      
      // Mock existing user
      dynamoMock.on(QueryCommand).resolves({
        Items: [{ id: '123', email: 'existing@example.com' }]
      });
      
      await expect(userService.create(createUserRequest)).rejects.toThrow('User with this email already exists');
    });
  });
});
```

### Integration Tests
```typescript
// tests/integration/functions/users.test.ts
import { APIGatewayProxyEvent, Context } from 'aws-lambda';
import { handler as createUserHandler } from '@/functions/users/create';
import { handler as getUserHandler } from '@/functions/users/get';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand, DeleteCommand } from '@aws-sdk/lib-dynamodb';

describe('User Functions Integration', () => {
  let dynamoClient: DynamoDBDocumentClient;
  const tableName = process.env.USERS_TABLE_NAME || 'test-users-table';
  
  beforeAll(() => {
    const client = new DynamoDBClient({
      endpoint: 'http://localhost:8000', // Local DynamoDB
      region: 'us-west-2'
    });
    dynamoClient = DynamoDBDocumentClient.from(client);
  });
  
  beforeEach(async () => {
    // Clean up test data
    await cleanupTestData();
  });
  
  afterEach(async () => {
    await cleanupTestData();
  });
  
  describe('Create User', () => {
    it('should create user successfully', async () => {
      const event: Partial<APIGatewayProxyEvent> = {
        httpMethod: 'POST',
        path: '/users',
        body: JSON.stringify({
          email: 'test@example.com',
          name: 'Test User',
          password: 'password123'
        }),
        headers: {
          'Content-Type': 'application/json'
        },
        requestContext: {
          requestId: 'test-request-id',
          authorizer: {
            principalId: 'test-user-id'
          }
        } as any
      };
      
      const context: Partial<Context> = {
        awsRequestId: 'test-request-id'
      };
      
      const result = await createUserHandler(
        event as APIGatewayProxyEvent,
        context as Context,
        () => {}
      );
      
      expect(result.statusCode).toBe(201);
      
      const responseBody = JSON.parse(result.body);
      expect(responseBody.user).toMatchObject({
        email: 'test@example.com',
        name: 'Test User'
      });
      expect(responseBody.user.id).toBeDefined();
      expect(responseBody.user.passwordHash).toBeUndefined();
    });
    
    it('should return 400 for invalid request', async () => {
      const event: Partial<APIGatewayProxyEvent> = {
        httpMethod: 'POST',
        path: '/users',
        body: JSON.stringify({
          email: 'invalid-email',
          name: ''
        }),
        headers: {
          'Content-Type': 'application/json'
        },
        requestContext: {
          requestId: 'test-request-id'
        } as any
      };
      
      const result = await createUserHandler(
        event as APIGatewayProxyEvent,
        {} as Context,
        () => {}
      );
      
      expect(result.statusCode).toBe(400);
    });
  });
  
  describe('Get User', () => {
    it('should get user successfully', async () => {
      // First create a user
      const testUser = {
        id: 'test-user-123',
        email: 'test@example.com',
        name: 'Test User',
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
        isActive: true
      };
      
      await dynamoClient.send(new PutCommand({
        TableName: tableName,
        Item: testUser
      }));
      
      const event: Partial<APIGatewayProxyEvent> = {
        httpMethod: 'GET',
        path: '/users/test-user-123',
        pathParameters: {
          id: 'test-user-123'
        },
        requestContext: {
          requestId: 'test-request-id',
          authorizer: {
            principalId: 'test-user-id'
          }
        } as any
      };
      
      const result = await getUserHandler(
        event as APIGatewayProxyEvent,
        {} as Context,
        () => {}
      );
      
      expect(result.statusCode).toBe(200);
      
      const responseBody = JSON.parse(result.body);
      expect(responseBody.user).toMatchObject({
        id: 'test-user-123',
        email: 'test@example.com',
        name: 'Test User'
      });
    });
    
    it('should return 404 for non-existent user', async () => {
      const event: Partial<APIGatewayProxyEvent> = {
        httpMethod: 'GET',
        path: '/users/nonexistent',
        pathParameters: {
          id: 'nonexistent'
        },
        requestContext: {
          requestId: 'test-request-id'
        } as any
      };
      
      const result = await getUserHandler(
        event as APIGatewayProxyEvent,
        {} as Context,
        () => {}
      );
      
      expect(result.statusCode).toBe(404);
    });
  });
  
  async function cleanupTestData() {
    // Clean up any test data
    const testUserIds = ['test-user-123'];
    
    for (const id of testUserIds) {
      try {
        await dynamoClient.send(new DeleteCommand({
          TableName: tableName,
          Key: { id }
        }));
      } catch (error) {
        // Ignore errors for non-existent items
      }
    }
  }
});
```

### End-to-End Tests
```typescript
// tests/e2e/user-workflow.test.ts
import axios, { AxiosInstance } from 'axios';
import { v4 as uuidv4 } from 'uuid';

describe('User Workflow E2E', () => {
  let api: AxiosInstance;
  let authToken: string;
  let createdUserId: string;
  
  beforeAll(async () => {
    const baseURL = process.env.API_BASE_URL || 'http://localhost:3000';
    
    api = axios.create({
      baseURL,
      timeout: 10000,
      validateStatus: () => true // Don't throw on HTTP errors
    });
    
    // Get auth token for tests
    authToken = await getAuthToken();
  });
  
  afterAll(async () => {
    // Clean up created user
    if (createdUserId) {
      await api.delete(`/users/${createdUserId}`, {
        headers: { Authorization: `Bearer ${authToken}` }
      });
    }
  });
  
  describe('Complete User Lifecycle', () => {
    it('should complete full user workflow', async () => {
      const userEmail = `test-${uuidv4()}@example.com`;
      const userData = {
        email: userEmail,
        name: 'E2E Test User',
        password: 'password123'
      };
      
      // 1. Create user
      const createResponse = await api.post('/users', userData, {
        headers: { Authorization: `Bearer ${authToken}` }
      });
      
      expect(createResponse.status).toBe(201);
      expect(createResponse.data.user.email).toBe(userEmail);
      
      createdUserId = createResponse.data.user.id;
      
      // 2. Get user
      const getResponse = await api.get(`/users/${createdUserId}`, {
        headers: { Authorization: `Bearer ${authToken}` }
      });
      
      expect(getResponse.status).toBe(200);
      expect(getResponse.data.user.id).toBe(createdUserId);
      expect(getResponse.data.user.email).toBe(userEmail);
      
      // 3. Update user
      const updateData = { name: 'Updated E2E Test User' };
      const updateResponse = await api.put(`/users/${createdUserId}`, updateData, {
        headers: { Authorization: `Bearer ${authToken}` }
      });
      
      expect(updateResponse.status).toBe(200);
      expect(updateResponse.data.user.name).toBe('Updated E2E Test User');
      
      // 4. List users (should include our user)
      const listResponse = await api.get('/users', {
        headers: { Authorization: `Bearer ${authToken}` }
      });
      
      expect(listResponse.status).toBe(200);
      expect(listResponse.data.users).toContainEqual(
        expect.objectContaining({ id: createdUserId })
      );
      
      // 5. Delete user
      const deleteResponse = await api.delete(`/users/${createdUserId}`, {
        headers: { Authorization: `Bearer ${authToken}` }
      });
      
      expect(deleteResponse.status).toBe(204);
      
      // 6. Verify user is deleted
      const getDeletedResponse = await api.get(`/users/${createdUserId}`, {
        headers: { Authorization: `Bearer ${authToken}` }
      });
      
      expect(getDeletedResponse.status).toBe(404);
      
      createdUserId = ''; // Mark as cleaned up
    });
    
    it('should handle authentication errors', async () => {
      const response = await api.get('/users/123');
      expect(response.status).toBe(401);
    });
    
    it('should handle validation errors', async () => {
      const invalidUserData = {
        email: 'invalid-email',
        name: '',
        password: '123' // Too short
      };
      
      const response = await api.post('/users', invalidUserData, {
        headers: { Authorization: `Bearer ${authToken}` }
      });
      
      expect(response.status).toBe(400);
      expect(response.data.message).toContain('Validation error');
    });
  });
  
  async function getAuthToken(): Promise<string> {
    // Mock authentication - in real tests, this would authenticate with your auth service
    const authResponse = await api.post('/auth/login', {
      email: 'admin@example.com',
      password: 'admin123'
    });
    
    if (authResponse.status !== 200) {
      throw new Error('Failed to authenticate for E2E tests');
    }
    
    return authResponse.data.token;
  }
});
```

## Test Configuration

### Jest Configuration
```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: [
    '**/__tests__/**/*.ts',
    '**/?(*.)+(spec|test).ts'
  ],
  transform: {
    '^.+\\.ts$': 'ts-jest'
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts',
    '!src/**/*.spec.ts'
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],
  moduleNameMapping: {
    '^@/(.*)$': '<rootDir>/src/$1'
  },
  testTimeout: 30000,
  globalSetup: '<rootDir>/tests/global-setup.ts',
  globalTeardown: '<rootDir>/tests/global-teardown.ts'
};
```

### Test Setup
```typescript
// tests/setup.ts
import { config } from 'dotenv';

// Load test environment variables
config({ path: '.env.test' });

// Set test environment variables
process.env.NODE_ENV = 'test';
process.env.AWS_REGION = 'us-west-2';
process.env.USERS_TABLE_NAME = 'test-users-table';
process.env.LOG_LEVEL = 'error'; // Reduce log noise in tests

// Mock AWS SDK clients globally if needed
jest.mock('@aws-sdk/client-dynamodb');
jest.mock('@aws-sdk/lib-dynamodb');

// Global test utilities
global.createMockEvent = (overrides = {}) => ({
  httpMethod: 'GET',
  path: '/test',
  headers: {},
  queryStringParameters: null,
  pathParameters: null,
  body: null,
  requestContext: {
    requestId: 'test-request-id',
    identity: {
      sourceIp: '127.0.0.1'
    }
  },
  ...overrides
});

global.createMockContext = (overrides = {}) => ({
  awsRequestId: 'test-request-id',
  functionName: 'test-function',
  functionVersion: '1',
  memoryLimitInMB: '256',
  getRemainingTimeInMillis: () => 30000,
  ...overrides
});
```

### Global Setup and Teardown
```typescript
// tests/global-setup.ts
import { DynamoDBClient, CreateTableCommand } from '@aws-sdk/client-dynamodb';

export default async function globalSetup() {
  console.log('Setting up test environment...');
  
  // Start local DynamoDB if needed
  if (process.env.USE_LOCAL_DYNAMODB === 'true') {
    await setupLocalDynamoDB();
  }
  
  console.log('Test environment setup complete');
}

async function setupLocalDynamoDB() {
  const client = new DynamoDBClient({
    endpoint: 'http://localhost:8000',
    region: 'us-west-2'
  });
  
  // Create test tables
  const createTableParams = {
    TableName: 'test-users-table',
    KeySchema: [
      { AttributeName: 'id', KeyType: 'HASH' }
    ],
    AttributeDefinitions: [
      { AttributeName: 'id', AttributeType: 'S' },
      { AttributeName: 'email', AttributeType: 'S' }
    ],
    GlobalSecondaryIndexes: [
      {
        IndexName: 'EmailIndex',
        KeySchema: [
          { AttributeName: 'email', KeyType: 'HASH' }
        ],
        Projection: { ProjectionType: 'ALL' },
        ProvisionedThroughput: {
          ReadCapacityUnits: 1,
          WriteCapacityUnits: 1
        }
      }
    ],
    ProvisionedThroughput: {
      ReadCapacityUnits: 1,
      WriteCapacityUnits: 1
    }
  };
  
  try {
    await client.send(new CreateTableCommand(createTableParams));
    console.log('Test DynamoDB table created');
  } catch (error) {
    if (error.name !== 'ResourceInUseException') {
      console.error('Error creating test table:', error);
    }
  }
}
```

## Load Testing

### Artillery Configuration
```yaml
# tests/load/artillery.yml
config:
  target: 'https://api.example.com'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Ramp up load"
    - duration: 300
      arrivalRate: 100
      name: "Sustained load"
  variables:
    authToken: "{{ $processEnvironment.AUTH_TOKEN }}"
  
scenarios:
  - name: "User CRUD Operations"
    weight: 70
    flow:
      - post:
          url: "/users"
          headers:
            Authorization: "Bearer {{ authToken }}"
            Content-Type: "application/json"
          json:
            email: "load-test-{{ $randomString() }}@example.com"
            name: "Load Test User"
            password: "password123"
          capture:
            - json: "$.user.id"
              as: "userId"
      - get:
          url: "/users/{{ userId }}"
          headers:
            Authorization: "Bearer {{ authToken }}"
      - put:
          url: "/users/{{ userId }}"
          headers:
            Authorization: "Bearer {{ authToken }}"
            Content-Type: "application/json"
          json:
            name: "Updated Load Test User"
      - delete:
          url: "/users/{{ userId }}"
          headers:
            Authorization: "Bearer {{ authToken }}"
  
  - name: "Read-only Operations"
    weight: 30
    flow:
      - get:
          url: "/users"
          headers:
            Authorization: "Bearer {{ authToken }}"
          qs:
            limit: 10
            offset: "{{ $randomInt(0, 100) }}"
```

### Performance Test Script
```typescript
// tests/load/performance.test.ts
import { spawn } from 'child_process';
import { promisify } from 'util';

describe('Performance Tests', () => {
  const timeout = 10 * 60 * 1000; // 10 minutes
  
  it('should handle sustained load', async () => {
    const artilleryProcess = spawn('artillery', [
      'run',
      'tests/load/artillery.yml',
      '--output',
      'tests/load/results.json'
    ]);
    
    let stdout = '';
    let stderr = '';
    
    artilleryProcess.stdout.on('data', (data) => {
      stdout += data.toString();
    });
    
    artilleryProcess.stderr.on('data', (data) => {
      stderr += data.toString();
    });
    
    const exitCode = await new Promise((resolve) => {
      artilleryProcess.on('close', resolve);
    });
    
    console.log('Artillery output:', stdout);
    
    if (stderr) {
      console.error('Artillery errors:', stderr);
    }
    
    expect(exitCode).toBe(0);
    
    // Parse results and assert performance metrics
    const results = require('./results.json');
    
    expect(results.aggregate.counters['http.codes.200']).toBeGreaterThan(0);
    expect(results.aggregate.rates['http.request_rate']).toBeGreaterThan(50);
    expect(results.aggregate.latency.p95).toBeLessThan(1000); // 95th percentile < 1s
    expect(results.aggregate.counters['errors.ECONNREFUSED']).toBeFalsy();
  }, timeout);
});
```

## CI/CD Integration

### GitHub Actions Workflow
```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

  integration-tests:
    runs-on: ubuntu-latest
    
    services:
      dynamodb:
        image: amazon/dynamodb-local
        ports:
          - 8000:8000
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Setup test environment
        run: |
          npm run dynamodb:install
          npm run test:setup
        env:
          USE_LOCAL_DYNAMODB: true
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          USERS_TABLE_NAME: test-users-table
          AWS_REGION: us-west-2

  e2e-tests:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Deploy to test environment
        run: npm run deploy:test
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          API_BASE_URL: ${{ secrets.TEST_API_BASE_URL }}
          TEST_AUTH_TOKEN: ${{ secrets.TEST_AUTH_TOKEN }}
      
      - name: Cleanup test environment
        if: always()
        run: npm run remove:test
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Best Practices

### Testing Guidelines
- **Test Pyramid**: More unit tests, fewer integration tests, minimal E2E tests
- **Isolation**: Each test should be independent and not rely on external state
- **Mocking**: Mock external dependencies in unit tests
- **Real Services**: Use real AWS services in integration tests when possible
- **Test Data**: Use factories or fixtures for consistent test data
- **Cleanup**: Always clean up resources created during tests

### Performance Testing
- **Baseline**: Establish performance baselines for critical functions
- **Load Testing**: Test with realistic load patterns
- **Monitoring**: Monitor key metrics during load tests
- **Bottlenecks**: Identify and address performance bottlenecks
- **Scaling**: Test auto-scaling behavior under load

### CI/CD Integration
- **Fast Feedback**: Run unit tests on every commit
- **Staged Testing**: Run integration tests on pull requests
- **E2E Testing**: Run E2E tests on main branch deployments
- **Parallel Execution**: Run tests in parallel to reduce CI time
- **Test Reports**: Generate and store test reports and coverage data