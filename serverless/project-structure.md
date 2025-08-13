# Serverless Framework Project Structure

## Standard Directory Layout

```
serverless-app/
├── src/
│   ├── functions/
│   │   ├── auth/
│   │   │   ├── login.ts
│   │   │   ├── register.ts
│   │   │   └── refresh.ts
│   │   ├── users/
│   │   │   ├── create.ts
│   │   │   ├── get.ts
│   │   │   ├── update.ts
│   │   │   └── delete.ts
│   │   └── shared/
│   │       ├── middleware/
│   │       ├── utils/
│   │       └── types/
│   ├── layers/
│   │   ├── common/
│   │   │   └── nodejs/
│   │   │       └── node_modules/
│   │   └── utils/
│   └── resources/
│       ├── dynamodb.yml
│       ├── s3.yml
│       └── cognito.yml
├── config/
│   ├── dev.yml
│   ├── staging.yml
│   └── prod.yml
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── scripts/
│   ├── deploy.sh
│   └── seed-data.js
├── serverless.yml
├── package.json
├── tsconfig.json
├── jest.config.js
└── README.md
```

## Serverless.yml Configuration

### Framework v4 Configuration
```yaml
# serverless.yml
service: ${self:custom.serviceName}
frameworkVersion: '4'

provider:
  name: aws
  runtime: nodejs20.x
  region: ${opt:region, 'us-west-2'}
  stage: ${opt:stage, 'dev'}
  memorySize: 256
  timeout: 30
  
  # Environment variables
  environment:
    STAGE: ${self:provider.stage}
    REGION: ${self:provider.region}
    SERVICE_NAME: ${self:service}
    NODE_ENV: ${self:custom.nodeEnv.${self:provider.stage}, 'development'}
    LOG_LEVEL: ${self:custom.logLevel.${self:provider.stage}, 'info'}
  
  # IAM role statements
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:Scan
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:DeleteItem
          Resource:
            - "arn:aws:dynamodb:${self:provider.region}:*:table/${self:custom.tableName}"
            - "arn:aws:dynamodb:${self:provider.region}:*:table/${self:custom.tableName}/index/*"
  
  # VPC configuration (if needed)
  vpc:
    securityGroupIds:
      - ${self:custom.securityGroupId}
    subnetIds:
      - ${self:custom.subnetId1}
      - ${self:custom.subnetId2}
  
  # Tracing and monitoring
  tracing:
    lambda: true
    apiGateway: true
  
  logs:
    restApi: true
    frameworkLambda: true

# Custom variables
custom:
  serviceName: my-serverless-app
  tableName: ${self:service}-${self:provider.stage}-table
  
  # Environment-specific configurations
  nodeEnv:
    dev: development
    staging: staging
    prod: production
  
  logLevel:
    dev: debug
    staging: info
    prod: warn
  
  # Webpack configuration
  webpack:
    webpackConfig: webpack.config.js
    includeModules: true
    packager: npm
    excludeFiles: src/**/*.test.ts
  
  # API Gateway configuration
  customDomain:
    domainName: ${self:custom.domainNames.${self:provider.stage}}
    basePath: api
    stage: ${self:provider.stage}
    createRoute53Record: true
  
  domainNames:
    dev: api-dev.example.com
    staging: api-staging.example.com
    prod: api.example.com
  
  # DynamoDB configuration
  dynamodb:
    start:
      port: 8000
      inMemory: true
      migrate: true
      seed: true
    seed:
      domain:
        sources:
          - table: ${self:custom.tableName}
            sources: [./tests/fixtures/users.json]

# Plugins
plugins:
  - serverless-webpack
  - serverless-offline
  - serverless-dynamodb-local
  - serverless-domain-manager
  - serverless-plugin-tracing
  - serverless-plugin-aws-alerts
  - serverless-prune-plugin

# Functions
functions:
  # Auth functions
  login:
    handler: src/functions/auth/login.handler
    events:
      - http:
          path: auth/login
          method: post
          cors: true
          request:
            schemas:
              application/json: ${file(src/schemas/auth/login-request.json)}
  
  register:
    handler: src/functions/auth/register.handler
    events:
      - http:
          path: auth/register
          method: post
          cors: true
  
  # User functions
  getUser:
    handler: src/functions/users/get.handler
    events:
      - http:
          path: users/{id}
          method: get
          cors: true
          authorizer:
            name: authorizerFunc
            resultTtlInSeconds: 300
  
  createUser:
    handler: src/functions/users/create.handler
    events:
      - http:
          path: users
          method: post
          cors: true
          authorizer:
            name: authorizerFunc
  
  # Authorizer function
  authorizerFunc:
    handler: src/functions/auth/authorizer.handler

# Resources
resources:
  - ${file(src/resources/dynamodb.yml)}
  - ${file(src/resources/s3.yml)}
  - ${file(src/resources/cognito.yml)}
```

## Function Organization

### Function Handler Pattern
```typescript
// src/functions/users/get.ts
import { APIGatewayProxyHandler, APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import { middyfy } from '@/shared/middleware/middy';
import { UserService } from '@/shared/services/user.service';
import { createResponse } from '@/shared/utils/response';
import { logger } from '@/shared/utils/logger';

interface GetUserEvent extends APIGatewayProxyEvent {
  pathParameters: {
    id: string;
  };
}

const getUserHandler: APIGatewayProxyHandler = async (
  event: GetUserEvent
): Promise<APIGatewayProxyResult> => {
  try {
    const { id } = event.pathParameters;
    
    logger.info('Getting user', { userId: id });
    
    const userService = new UserService();
    const user = await userService.getById(id);
    
    if (!user) {
      return createResponse(404, { message: 'User not found' });
    }
    
    return createResponse(200, { user });
  } catch (error) {
    logger.error('Error getting user', { error, userId: event.pathParameters?.id });
    return createResponse(500, { message: 'Internal server error' });
  }
};

export const handler = middyfy(getUserHandler);
```

### Middleware Configuration
```typescript
// src/shared/middleware/middy.ts
import middy from '@middy/core';
import middyJsonBodyParser from '@middy/http-json-body-parser';
import middyHttpErrorHandler from '@middy/http-error-handler';
import middyValidator from '@middy/validator';
import middyCors from '@middy/http-cors';
import { APIGatewayProxyHandler } from 'aws-lambda';

export const middyfy = (handler: APIGatewayProxyHandler) => {
  return middy(handler)
    .use(middyJsonBodyParser())
    .use(middyValidator({
      eventSchema: {
        type: 'object',
        properties: {
          body: { type: 'object' }
        }
      }
    }))
    .use(middyCors({
      origin: process.env.ALLOWED_ORIGINS?.split(',') || ['*'],
      headers: 'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token',
      methods: 'GET,HEAD,OPTIONS,POST,PUT,DELETE'
    }))
    .use(middyHttpErrorHandler());
};
```

## Environment Configuration

### Stage-Specific Configuration
```yaml
# config/dev.yml
tableName: ${self:service}-dev-table
domainName: api-dev.example.com
logLevel: debug
memorySize: 256
timeout: 30

cors:
  origin: 'http://localhost:3000'
  credentials: true

monitoring:
  enabled: false

vpc:
  enabled: false
```

```yaml
# config/prod.yml
tableName: ${self:service}-prod-table
domainName: api.example.com
logLevel: warn
memorySize: 512
timeout: 30

cors:
  origin: 'https://app.example.com'
  credentials: true

monitoring:
  enabled: true
  
alerts:
  - functionErrors
  - functionDuration
  - functionThrottles

vpc:
  enabled: true
  securityGroupIds:
    - sg-12345678
  subnetIds:
    - subnet-12345678
    - subnet-87654321
```

## Resource Definitions

### DynamoDB Resources
```yaml
# src/resources/dynamodb.yml
Resources:
  UsersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: ${self:custom.tableName}
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
        - AttributeName: email
          AttributeType: S
        - AttributeName: createdAt
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      GlobalSecondaryIndexes:
        - IndexName: EmailIndex
          KeySchema:
            - AttributeName: email
              KeyType: HASH
          Projection:
            ProjectionType: ALL
        - IndexName: CreatedAtIndex
          KeySchema:
            - AttributeName: createdAt
              KeyType: HASH
          Projection:
            ProjectionType: ALL
      StreamSpecification:
        StreamViewType: NEW_AND_OLD_IMAGES
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      SSESpecification:
        SSEEnabled: true
      Tags:
        - Key: Environment
          Value: ${self:provider.stage}
        - Key: Service
          Value: ${self:service}

Outputs:
  UsersTableName:
    Value: !Ref UsersTable
    Export:
      Name: ${self:service}-${self:provider.stage}-UsersTableName
  
  UsersTableArn:
    Value: !GetAtt UsersTable.Arn
    Export:
      Name: ${self:service}-${self:provider.stage}-UsersTableArn
```

## Package.json Configuration

### Dependencies and Scripts
```json
{
  "name": "my-serverless-app",
  "version": "1.0.0",
  "description": "Serverless application with TypeScript",
  "main": "handler.js",
  "scripts": {
    "dev": "serverless offline start --stage dev",
    "build": "serverless package",
    "deploy:dev": "serverless deploy --stage dev",
    "deploy:staging": "serverless deploy --stage staging",
    "deploy:prod": "serverless deploy --stage prod",
    "remove:dev": "serverless remove --stage dev",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "type-check": "tsc --noEmit",
    "logs:dev": "serverless logs -f functionName --stage dev -t",
    "invoke:local": "serverless invoke local -f functionName",
    "dynamodb:install": "serverless dynamodb install",
    "dynamodb:start": "serverless dynamodb start"
  },
  "dependencies": {
    "@aws-lambda-powertools/logger": "^2.0.0",
    "@aws-lambda-powertools/metrics": "^2.0.0",
    "@aws-lambda-powertools/tracer": "^2.0.0",
    "@aws-sdk/client-dynamodb": "^3.450.0",
    "@aws-sdk/lib-dynamodb": "^3.450.0",
    "@middy/core": "^4.6.0",
    "@middy/http-cors": "^4.6.0",
    "@middy/http-error-handler": "^4.6.0",
    "@middy/http-json-body-parser": "^4.6.0",
    "@middy/validator": "^4.6.0",
    "aws-lambda": "^1.0.7",
    "joi": "^17.11.0",
    "uuid": "^9.0.1"
  },
  "devDependencies": {
    "@types/aws-lambda": "^8.10.126",
    "@types/jest": "^29.5.8",
    "@types/node": "^20.9.0",
    "@types/uuid": "^9.0.7",
    "@typescript-eslint/eslint-plugin": "^6.11.0",
    "@typescript-eslint/parser": "^6.11.0",
    "eslint": "^8.53.0",
    "jest": "^29.7.0",
    "serverless": "^4.0.0",
    "serverless-domain-manager": "^7.3.0",
    "serverless-dynamodb-local": "^0.2.40",
    "serverless-offline": "^13.3.0",
    "serverless-plugin-aws-alerts": "^1.8.0",
    "serverless-plugin-tracing": "^2.0.0",
    "serverless-prune-plugin": "^2.0.2",
    "serverless-webpack": "^5.13.0",
    "ts-jest": "^29.1.1",
    "ts-loader": "^9.5.0",
    "typescript": "^5.2.2",
    "webpack": "^5.89.0",
    "webpack-node-externals": "^3.0.0"
  }
}
```

## TypeScript Configuration

### TSConfig for Serverless
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "baseUrl": "./",
    "paths": {
      "@/*": ["src/*"],
      "@/shared/*": ["src/shared/*"],
      "@/functions/*": ["src/functions/*"],
      "@/types/*": ["src/shared/types/*"]
    }
  },
  "include": [
    "src/**/*"
  ],
  "exclude": [
    "node_modules",
    "dist",
    "tests",
    "**/*.test.ts",
    "**/*.spec.ts"
  ]
}
```

## Best Practices

### Function Organization
- **Single Responsibility**: Each function should have one clear purpose
- **Shared Code**: Use layers or shared modules for common functionality
- **Environment Variables**: Use stage-specific configuration files
- **Error Handling**: Implement consistent error handling patterns
- **Logging**: Use structured logging with correlation IDs

### Performance Optimization
- **Cold Start Reduction**: Use provisioned concurrency for critical functions
- **Bundle Size**: Optimize webpack configuration to reduce bundle size
- **Memory Allocation**: Right-size memory based on function requirements
- **Connection Pooling**: Reuse database connections across invocations

### Security Considerations
- **IAM Permissions**: Follow least privilege principle
- **API Authorization**: Implement proper authentication and authorization
- **Environment Variables**: Never store secrets in plain text
- **VPC Configuration**: Use VPC for functions that need private network access
- **CORS Configuration**: Configure CORS appropriately for your frontend