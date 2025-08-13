# Serverless Deployment Strategies

## Environment Management

### Multi-Stage Configuration
```yaml
# serverless.yml
service: ${self:custom.serviceName}
frameworkVersion: '4'

provider:
  name: aws
  runtime: nodejs20.x
  region: ${opt:region, self:custom.defaultRegion}
  stage: ${opt:stage, 'dev'}
  
  # Stage-specific configurations
  memorySize: ${self:custom.memorySize.${self:provider.stage}, 256}
  timeout: ${self:custom.timeout.${self:provider.stage}, 30}
  
  environment:
    STAGE: ${self:provider.stage}
    NODE_ENV: ${self:custom.nodeEnv.${self:provider.stage}}
    LOG_LEVEL: ${self:custom.logLevel.${self:provider.stage}}
    API_BASE_URL: ${self:custom.apiBaseUrl.${self:provider.stage}}

custom:
  serviceName: my-serverless-app
  defaultRegion: us-west-2
  
  # Stage-specific memory allocation
  memorySize:
    dev: 256
    staging: 512
    prod: 1024
  
  # Stage-specific timeout
  timeout:
    dev: 30
    staging: 60
    prod: 300
  
  # Environment configurations
  nodeEnv:
    dev: development
    staging: staging
    prod: production
  
  logLevel:
    dev: debug
    staging: info
    prod: warn
  
  # API base URLs
  apiBaseUrl:
    dev: https://api-dev.example.com
    staging: https://api-staging.example.com
    prod: https://api.example.com
  
  # Custom domain configuration
  customDomain:
    domainName: ${self:custom.domainNames.${self:provider.stage}}
    basePath: api
    stage: ${self:provider.stage}
    createRoute53Record: true
    certificateName: ${self:custom.certificateNames.${self:provider.stage}}
  
  domainNames:
    dev: api-dev.example.com
    staging: api-staging.example.com
    prod: api.example.com
  
  certificateNames:
    dev: "*.dev.example.com"
    staging: "*.staging.example.com"
    prod: "*.example.com"
  
  # Alerts configuration
  alerts:
    stages:
      - staging
      - prod
    topics:
      alarm:
        topic: ${self:service}-${self:provider.stage}-alerts
        notifications:
          - protocol: email
            endpoint: alerts@example.com
    alarms:
      - functionErrors
      - functionDuration
      - functionThrottles
    definitions:
      functionErrors:
        threshold: 5
        treatMissingData: notBreaching
      functionDuration:
        threshold: 10000
        treatMissingData: notBreaching

plugins:
  - serverless-domain-manager
  - serverless-plugin-aws-alerts
  - serverless-prune-plugin
```

### Environment-Specific Resource Configuration
```yaml
# config/dev.yml
# Development environment configuration
vpc:
  enabled: false

rds:
  enabled: false
  # Use DynamoDB for development

monitoring:
  enabled: false
  xrayTracing: false

backup:
  enabled: false

scaling:
  minCapacity: 1
  maxCapacity: 5

cors:
  origin: 'http://localhost:3000'
  credentials: true

rateLimiting:
  enabled: false
```

```yaml
# config/prod.yml
# Production environment configuration
vpc:
  enabled: true
  securityGroupIds:
    - sg-prod-lambda-sg
  subnetIds:
    - subnet-prod-private-1a
    - subnet-prod-private-1b

rds:
  enabled: true
  instanceClass: db.r5.large
  multiAZ: true
  backupRetentionPeriod: 30

monitoring:
  enabled: true
  xrayTracing: true
  enhancedMonitoring: true

backup:
  enabled: true
  retentionPeriod: 90

scaling:
  minCapacity: 10
  maxCapacity: 100
  targetUtilization: 70

cors:
  origin: 'https://app.example.com'
  credentials: true

rateLimiting:
  enabled: true
  requestsPerSecond: 1000
  burstLimit: 2000
```

## Blue-Green Deployment

### Alias-Based Blue-Green Deployment
```yaml
# serverless.yml - Blue-Green configuration
provider:
  name: aws
  runtime: nodejs20.x
  
  # Versioning configuration
  versionFunctions: true

functions:
  api:
    handler: src/handler.api
    events:
      - http:
          path: /{proxy+}
          method: ANY
    # Provisioned concurrency for production
    provisionedConcurrency: ${self:custom.provisionedConcurrency.${self:provider.stage}, 0}

custom:
  # Provisioned concurrency by stage
  provisionedConcurrency:
    prod: 50
    staging: 10
  
  # Alias configuration
  aliases:
    live: ${self:provider.stage}
    
plugins:
  - serverless-plugin-aws-aliases
```

### Blue-Green Deployment Script
```bash
#!/bin/bash
# scripts/blue-green-deploy.sh

set -e

STAGE=${1:-prod}
SERVICE_NAME="my-serverless-app"
FUNCTION_NAME="${SERVICE_NAME}-${STAGE}-api"

echo "Starting blue-green deployment for stage: $STAGE"

# Deploy new version
echo "Deploying new version..."
serverless deploy --stage $STAGE

# Get the new version number
NEW_VERSION=$(aws lambda get-function --function-name $FUNCTION_NAME --query 'Configuration.Version' --output text)
echo "New version deployed: $NEW_VERSION"

# Update alias to point to new version with weighted routing
echo "Starting traffic shift to new version..."
aws lambda update-alias \
  --function-name $FUNCTION_NAME \
  --name live \
  --function-version $NEW_VERSION \
  --routing-config "AdditionalVersionWeights={\"$NEW_VERSION\"=0.1}"

echo "10% of traffic now routing to new version"
echo "Monitoring for 5 minutes..."
sleep 300

# Check CloudWatch metrics for errors
ERROR_COUNT=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=$FUNCTION_NAME \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum \
  --query 'Datapoints[0].Sum' \
  --output text)

if [ "$ERROR_COUNT" != "None" ] && [ "$ERROR_COUNT" -gt 5 ]; then
  echo "High error rate detected. Rolling back..."
  # Get previous version
  PREVIOUS_VERSION=$(aws lambda list-aliases \
    --function-name $FUNCTION_NAME \
    --query 'Aliases[?Name==`live`].FunctionVersion' \
    --output text)
  
  # Rollback to previous version
  aws lambda update-alias \
    --function-name $FUNCTION_NAME \
    --name live \
    --function-version $PREVIOUS_VERSION
  
  echo "Rollback completed"
  exit 1
fi

# Gradually increase traffic to new version
echo "Increasing traffic to 50%..."
aws lambda update-alias \
  --function-name $FUNCTION_NAME \
  --name live \
  --function-version $NEW_VERSION \
  --routing-config "AdditionalVersionWeights={\"$NEW_VERSION\"=0.5}"

sleep 300

echo "Switching 100% traffic to new version..."
aws lambda update-alias \
  --function-name $FUNCTION_NAME \
  --name live \
  --function-version $NEW_VERSION

echo "Blue-green deployment completed successfully"
```

## Canary Deployment

### Canary Configuration with API Gateway
```yaml
# serverless.yml - Canary deployment
provider:
  name: aws
  runtime: nodejs20.x
  
  # API Gateway deployment configuration
  deploymentBucket:
    name: ${self:service}-${self:provider.stage}-deployments
    
  apiGateway:
    deploymentConfiguration:
      type: Canary10Percent5Minutes
      alarms:
        - AliasErrorMetricGreaterThanZeroAlarm
        - AliasLatencyMetricGreaterThan2000msAlarm

functions:
  api:
    handler: src/handler.api
    events:
      - http:
          path: /{proxy+}
          method: ANY

resources:
  Resources:
    # CloudWatch Alarms for canary deployment
    AliasErrorMetricGreaterThanZeroAlarm:
      Type: AWS::CloudWatch::Alarm
      Properties:
        AlarmDescription: Lambda function errors
        ComparisonOperator: GreaterThanThreshold
        EvaluationPeriods: 2
        MetricName: Errors
        Namespace: AWS/Lambda
        Period: 60
        Statistic: Sum
        Threshold: 0
        Dimensions:
          - Name: FunctionName
            Value: !Ref ApiLambdaFunction
    
    AliasLatencyMetricGreaterThan2000msAlarm:
      Type: AWS::CloudWatch::Alarm
      Properties:
        AlarmDescription: Lambda function latency
        ComparisonOperator: GreaterThanThreshold
        EvaluationPeriods: 2
        MetricName: Duration
        Namespace: AWS/Lambda
        Period: 60
        Statistic: Average
        Threshold: 2000
        Dimensions:
          - Name: FunctionName
            Value: !Ref ApiLambdaFunction
```

### Automated Canary Deployment
```typescript
// scripts/canary-deploy.ts
import { 
  LambdaClient, 
  UpdateAliasCommand, 
  GetAliasCommand,
  CreateAliasCommand 
} from '@aws-sdk/client-lambda';
import { 
  CloudWatchClient, 
  GetMetricStatisticsCommand 
} from '@aws-sdk/client-cloudwatch';

interface CanaryConfig {
  functionName: string;
  aliasName: string;
  newVersion: string;
  stages: Array<{
    percentage: number;
    durationMinutes: number;
  }>;
  rollbackThreshold: {
    errorRate: number;
    latencyMs: number;
  };
}

class CanaryDeployment {
  private lambda: LambdaClient;
  private cloudwatch: CloudWatchClient;
  
  constructor() {
    this.lambda = new LambdaClient({ region: process.env.AWS_REGION });
    this.cloudwatch = new CloudWatchClient({ region: process.env.AWS_REGION });
  }
  
  async deploy(config: CanaryConfig): Promise<void> {
    console.log(`Starting canary deployment for ${config.functionName}`);
    
    try {
      // Get current alias version for rollback
      const currentAlias = await this.lambda.send(new GetAliasCommand({
        FunctionName: config.functionName,
        Name: config.aliasName
      }));
      
      const previousVersion = currentAlias.FunctionVersion!;
      console.log(`Previous version: ${previousVersion}`);
      console.log(`New version: ${config.newVersion}`);
      
      // Execute canary stages
      for (const stage of config.stages) {
        console.log(`Deploying ${stage.percentage}% traffic to new version`);
        
        await this.updateTrafficSplit(
          config.functionName,
          config.aliasName,
          config.newVersion,
          stage.percentage / 100
        );
        
        // Wait for the specified duration
        await this.sleep(stage.durationMinutes * 60 * 1000);
        
        // Check metrics
        const shouldRollback = await this.checkMetrics(
          config.functionName,
          config.rollbackThreshold,
          stage.durationMinutes
        );
        
        if (shouldRollback) {
          console.log('Metrics indicate issues. Rolling back...');
          await this.rollback(config.functionName, config.aliasName, previousVersion);
          throw new Error('Canary deployment failed - rolled back');
        }
        
        console.log(`Stage ${stage.percentage}% completed successfully`);
      }
      
      // Final switch to 100%
      console.log('Switching 100% traffic to new version');
      await this.lambda.send(new UpdateAliasCommand({
        FunctionName: config.functionName,
        Name: config.aliasName,
        FunctionVersion: config.newVersion
      }));
      
      console.log('Canary deployment completed successfully');
      
    } catch (error) {
      console.error('Canary deployment failed:', error);
      throw error;
    }
  }
  
  private async updateTrafficSplit(
    functionName: string,
    aliasName: string,
    newVersion: string,
    weight: number
  ): Promise<void> {
    const routingConfig = weight < 1 ? {
      AdditionalVersionWeights: {
        [newVersion]: weight
      }
    } : undefined;
    
    await this.lambda.send(new UpdateAliasCommand({
      FunctionName: functionName,
      Name: aliasName,
      FunctionVersion: weight === 1 ? newVersion : undefined,
      RoutingConfig: routingConfig
    }));
  }
  
  private async checkMetrics(
    functionName: string,
    threshold: { errorRate: number; latencyMs: number },
    durationMinutes: number
  ): Promise<boolean> {
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - (durationMinutes * 60 * 1000));
    
    // Check error rate
    const errorMetrics = await this.cloudwatch.send(new GetMetricStatisticsCommand({
      Namespace: 'AWS/Lambda',
      MetricName: 'Errors',
      Dimensions: [
        { Name: 'FunctionName', Value: functionName }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 60,
      Statistics: ['Sum']
    }));
    
    const invocationMetrics = await this.cloudwatch.send(new GetMetricStatisticsCommand({
      Namespace: 'AWS/Lambda',
      MetricName: 'Invocations',
      Dimensions: [
        { Name: 'FunctionName', Value: functionName }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 60,
      Statistics: ['Sum']
    }));
    
    // Calculate error rate
    const totalErrors = errorMetrics.Datapoints?.reduce((sum, dp) => sum + (dp.Sum || 0), 0) || 0;
    const totalInvocations = invocationMetrics.Datapoints?.reduce((sum, dp) => sum + (dp.Sum || 0), 0) || 0;
    const errorRate = totalInvocations > 0 ? (totalErrors / totalInvocations) * 100 : 0;
    
    console.log(`Error rate: ${errorRate.toFixed(2)}%`);
    
    if (errorRate > threshold.errorRate) {
      console.log(`Error rate ${errorRate.toFixed(2)}% exceeds threshold ${threshold.errorRate}%`);
      return true;
    }
    
    // Check latency
    const latencyMetrics = await this.cloudwatch.send(new GetMetricStatisticsCommand({
      Namespace: 'AWS/Lambda',
      MetricName: 'Duration',
      Dimensions: [
        { Name: 'FunctionName', Value: functionName }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 60,
      Statistics: ['Average']
    }));
    
    const avgLatency = latencyMetrics.Datapoints?.reduce((sum, dp) => sum + (dp.Average || 0), 0) || 0;
    const latencyCount = latencyMetrics.Datapoints?.length || 1;
    const averageLatency = avgLatency / latencyCount;
    
    console.log(`Average latency: ${averageLatency.toFixed(2)}ms`);
    
    if (averageLatency > threshold.latencyMs) {
      console.log(`Latency ${averageLatency.toFixed(2)}ms exceeds threshold ${threshold.latencyMs}ms`);
      return true;
    }
    
    return false;
  }
  
  private async rollback(
    functionName: string,
    aliasName: string,
    previousVersion: string
  ): Promise<void> {
    await this.lambda.send(new UpdateAliasCommand({
      FunctionName: functionName,
      Name: aliasName,
      FunctionVersion: previousVersion
    }));
    
    console.log(`Rolled back to version ${previousVersion}`);
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const canaryConfig: CanaryConfig = {
  functionName: process.env.FUNCTION_NAME!,
  aliasName: 'live',
  newVersion: process.env.NEW_VERSION!,
  stages: [
    { percentage: 10, durationMinutes: 5 },
    { percentage: 25, durationMinutes: 10 },
    { percentage: 50, durationMinutes: 15 },
    { percentage: 75, durationMinutes: 10 }
  ],
  rollbackThreshold: {
    errorRate: 1, // 1% error rate
    latencyMs: 2000 // 2 second latency
  }
};

const deployment = new CanaryDeployment();
deployment.deploy(canaryConfig).catch(console.error);
```

## CI/CD Pipeline Integration

### GitHub Actions Deployment Workflow
```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main

env:
  NODE_VERSION: '20'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Run linting
        run: npm run lint

  deploy-dev:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Deploy to dev
        run: npm run deploy:dev
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Run smoke tests
        run: npm run test:smoke
        env:
          API_BASE_URL: ${{ secrets.DEV_API_BASE_URL }}

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Deploy to staging
        run: npm run deploy:staging
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          API_BASE_URL: ${{ secrets.STAGING_API_BASE_URL }}
          AUTH_TOKEN: ${{ secrets.STAGING_AUTH_TOKEN }}

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Deploy to production (canary)
        run: npm run deploy:prod:canary
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
          FUNCTION_NAME: ${{ secrets.PROD_FUNCTION_NAME }}
      
      - name: Run production smoke tests
        run: npm run test:smoke:prod
        env:
          API_BASE_URL: ${{ secrets.PROD_API_BASE_URL }}
      
      - name: Notify deployment success
        uses: 8398a7/action-slack@v3
        with:
          status: success
          text: 'Production deployment completed successfully'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Deployment Scripts
```json
{
  "scripts": {
    "deploy:dev": "serverless deploy --stage dev --verbose",
    "deploy:staging": "serverless deploy --stage staging --verbose",
    "deploy:prod": "serverless deploy --stage prod --verbose",
    "deploy:prod:canary": "npm run deploy:prod && npm run canary:deploy",
    "canary:deploy": "ts-node scripts/canary-deploy.ts",
    "remove:dev": "serverless remove --stage dev",
    "remove:staging": "serverless remove --stage staging",
    "logs:dev": "serverless logs -f api --stage dev -t",
    "logs:prod": "serverless logs -f api --stage prod -t",
    "invoke:local": "serverless invoke local -f api --data '{\"httpMethod\":\"GET\",\"path\":\"/health\"}'",
    "test:smoke": "jest tests/smoke --testTimeout=30000",
    "test:smoke:prod": "jest tests/smoke --testTimeout=30000 --testNamePattern='production'",
    "domain:create:dev": "serverless create_domain --stage dev",
    "domain:create:staging": "serverless create_domain --stage staging",
    "domain:create:prod": "serverless create_domain --stage prod"
  }
}
```

## Rollback Strategies

### Automated Rollback Script
```bash
#!/bin/bash
# scripts/rollback.sh

set -e

STAGE=${1:-prod}
SERVICE_NAME="my-serverless-app"
FUNCTION_NAME="${SERVICE_NAME}-${STAGE}-api"

if [ -z "$2" ]; then
  echo "Usage: $0 <stage> <version>"
  echo "Example: $0 prod 42"
  exit 1
fi

TARGET_VERSION=$2

echo "Rolling back $FUNCTION_NAME to version $TARGET_VERSION"

# Verify the target version exists
aws lambda get-function \
  --function-name $FUNCTION_NAME \
  --qualifier $TARGET_VERSION > /dev/null

if [ $? -ne 0 ]; then
  echo "Error: Version $TARGET_VERSION does not exist"
  exit 1
fi

# Update the alias to point to the target version
aws lambda update-alias \
  --function-name $FUNCTION_NAME \
  --name live \
  --function-version $TARGET_VERSION

echo "Rollback completed. Function is now running version $TARGET_VERSION"

# Run smoke tests to verify rollback
echo "Running smoke tests..."
npm run test:smoke

if [ $? -eq 0 ]; then
  echo "Rollback successful - smoke tests passed"
else
  echo "Warning: Smoke tests failed after rollback"
  exit 1
fi
```

### Database Migration Rollback
```typescript
// scripts/db-rollback.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, ScanCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';

interface MigrationRecord {
  id: string;
  version: string;
  appliedAt: string;
  rollbackScript?: string;
}

class DatabaseRollback {
  private dynamoClient: DynamoDBDocumentClient;
  private migrationsTable: string;
  
  constructor() {
    const client = new DynamoDBClient({ region: process.env.AWS_REGION });
    this.dynamoClient = DynamoDBDocumentClient.from(client);
    this.migrationsTable = process.env.MIGRATIONS_TABLE || 'migrations';
  }
  
  async rollbackToVersion(targetVersion: string): Promise<void> {
    console.log(`Rolling back database to version ${targetVersion}`);
    
    // Get all migrations after the target version
    const migrations = await this.getMigrationsAfterVersion(targetVersion);
    
    // Sort migrations in reverse order for rollback
    migrations.sort((a, b) => b.version.localeCompare(a.version));
    
    for (const migration of migrations) {
      if (migration.rollbackScript) {
        console.log(`Rolling back migration ${migration.version}`);
        await this.executeMigration(migration.rollbackScript);
        await this.removeMigrationRecord(migration.id);
      } else {
        console.warn(`No rollback script for migration ${migration.version}`);
      }
    }
    
    console.log('Database rollback completed');
  }
  
  private async getMigrationsAfterVersion(version: string): Promise<MigrationRecord[]> {
    const result = await this.dynamoClient.send(new ScanCommand({
      TableName: this.migrationsTable,
      FilterExpression: 'version > :version',
      ExpressionAttributeValues: {
        ':version': version
      }
    }));
    
    return result.Items as MigrationRecord[];
  }
  
  private async executeMigration(script: string): Promise<void> {
    // Execute the rollback script
    // This would depend on your specific migration system
    console.log(`Executing rollback script: ${script}`);
  }
  
  private async removeMigrationRecord(id: string): Promise<void> {
    await this.dynamoClient.send(new UpdateCommand({
      TableName: this.migrationsTable,
      Key: { id },
      UpdateExpression: 'SET rolledBack = :true, rolledBackAt = :timestamp',
      ExpressionAttributeValues: {
        ':true': true,
        ':timestamp': new Date().toISOString()
      }
    }));
  }
}
```

## Best Practices

### Deployment Guidelines
- **Environment Parity**: Keep environments as similar as possible
- **Immutable Deployments**: Deploy new versions rather than updating existing ones
- **Automated Testing**: Run comprehensive tests before production deployment
- **Gradual Rollouts**: Use canary or blue-green deployments for production
- **Monitoring**: Monitor key metrics during and after deployments
- **Rollback Plan**: Always have a tested rollback strategy

### Security Considerations
- **Secrets Management**: Use AWS Secrets Manager or Parameter Store
- **IAM Permissions**: Follow least privilege principle
- **Environment Isolation**: Use separate AWS accounts for different environments
- **Code Signing**: Sign deployment packages for integrity
- **Audit Logging**: Log all deployment activities

### Performance Optimization
- **Cold Start Reduction**: Use provisioned concurrency for critical functions
- **Bundle Optimization**: Minimize deployment package size
- **Connection Pooling**: Reuse database connections
- **Caching**: Implement appropriate caching strategies
- **Resource Sizing**: Right-size memory and timeout configurations