# Serverless Performance Optimization

## Cold Start Optimization

### Minimizing Cold Start Impact
```typescript
// src/shared/utils/connection-pool.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';

// Initialize clients outside the handler to reuse connections
const dynamoClient = new DynamoDBClient({
  region: process.env.AWS_REGION,
  maxAttempts: 3,
  requestTimeout: 5000
});

const documentClient = DynamoDBDocumentClient.from(dynamoClient, {
  marshallOptions: {
    removeUndefinedValues: true,
    convertEmptyValues: false
  }
});

// Export singleton instances
export { documentClient as dynamoDocumentClient };
```

```typescript
// src/functions/users/get.ts
import { APIGatewayProxyHandler } from 'aws-lambda';
import { dynamoDocumentClient } from '@/shared/utils/connection-pool';
import { GetCommand } from '@aws-sdk/lib-dynamodb';

// Pre-compile regex patterns
const UUID_REGEX = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

// Cache environment variables
const TABLE_NAME = process.env.USERS_TABLE_NAME!;

export const handler: APIGatewayProxyHandler = async (event) => {
  const userId = event.pathParameters?.id;
  
  // Fast validation
  if (!userId || !UUID_REGEX.test(userId)) {
    return {
      statusCode: 400,
      body: JSON.stringify({ message: 'Invalid user ID format' })
    };
  }
  
  try {
    const result = await dynamoDocumentClient.send(new GetCommand({
      TableName: TABLE_NAME,
      Key: { id: userId }
    }));
    
    if (!result.Item) {
      return {
        statusCode: 404,
        body: JSON.stringify({ message: 'User not found' })
      };
    }
    
    return {
      statusCode: 200,
      body: JSON.stringify({ user: result.Item })
    };
  } catch (error) {
    console.error('Error getting user:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ message: 'Internal server error' })
    };
  }
};
```

### Provisioned Concurrency Configuration
```yaml
# serverless.yml
functions:
  api:
    handler: src/handler.api
    # Provisioned concurrency for production
    provisionedConcurrency: ${self:custom.provisionedConcurrency.${self:provider.stage}, 0}
    events:
      - http:
          path: /{proxy+}
          method: ANY

custom:
  provisionedConcurrency:
    prod: 50      # Keep 50 instances warm
    staging: 10   # Keep 10 instances warm
    dev: 0        # No provisioned concurrency for dev
```

### Bundle Size Optimization
```javascript
// webpack.config.js
const path = require('path');
const nodeExternals = require('webpack-node-externals');

module.exports = {
  mode: 'production',
  entry: './src/handler.ts',
  target: 'node',
  externals: [
    nodeExternals({
      // Bundle AWS SDK v3 clients (they're not in Lambda runtime)
      allowlist: [/^@aws-sdk/]
    })
  ],
  module: {
    rules: [
      {
        test: /\.ts$/,
        use: 'ts-loader',
        exclude: /node_modules/
      }
    ]
  },
  resolve: {
    extensions: ['.ts', '.js'],
    alias: {
      '@': path.resolve(__dirname, 'src')
    }
  },
  output: {
    filename: 'handler.js',
    path: path.resolve(__dirname, 'dist'),
    libraryTarget: 'commonjs2'
  },
  optimization: {
    minimize: true,
    usedExports: true,
    sideEffects: false
  },
  // Exclude unnecessary files
  externalsPresets: { node: true }
};
```

## Memory and Timeout Optimization

### Right-Sizing Functions
```yaml
# serverless.yml - Function-specific configurations
functions:
  # Lightweight API functions
  getUser:
    handler: src/functions/users/get.handler
    memorySize: 256
    timeout: 10
    events:
      - http:
          path: users/{id}
          method: get
  
  # CPU-intensive processing
  processImage:
    handler: src/functions/media/process.handler
    memorySize: 3008  # Maximum memory for CPU-intensive tasks
    timeout: 300      # 5 minutes for processing
    events:
      - s3:
          bucket: ${self:custom.bucketName}
          event: s3:ObjectCreated:*
  
  # Database operations
  batchUpdate:
    handler: src/functions/batch/update.handler
    memorySize: 1024  # More memory for database operations
    timeout: 900      # 15 minutes for batch operations
    reservedConcurrency: 5  # Limit concurrent executions
```

### Memory Profiling
```typescript
// src/shared/utils/profiler.ts
export class MemoryProfiler {
  private startMemory: NodeJS.MemoryUsage;
  
  constructor() {
    this.startMemory = process.memoryUsage();
  }
  
  getMemoryUsage(): {
    heapUsed: number;
    heapTotal: number;
    external: number;
    rss: number;
    heapDelta: number;
  } {
    const current = process.memoryUsage();
    
    return {
      heapUsed: Math.round(current.heapUsed / 1024 / 1024), // MB
      heapTotal: Math.round(current.heapTotal / 1024 / 1024), // MB
      external: Math.round(current.external / 1024 / 1024), // MB
      rss: Math.round(current.rss / 1024 / 1024), // MB
      heapDelta: Math.round((current.heapUsed - this.startMemory.heapUsed) / 1024 / 1024) // MB
    };
  }
  
  logMemoryUsage(label: string): void {
    const usage = this.getMemoryUsage();
    console.log(`[${label}] Memory Usage:`, usage);
  }
}

// Usage in function
export const handler: APIGatewayProxyHandler = async (event) => {
  const profiler = new MemoryProfiler();
  
  try {
    // Your function logic here
    const result = await processData(event);
    
    profiler.logMemoryUsage('Function completed');
    
    return {
      statusCode: 200,
      body: JSON.stringify(result)
    };
  } catch (error) {
    profiler.logMemoryUsage('Function error');
    throw error;
  }
};
```

## Database Performance

### Connection Pooling and Optimization
```typescript
// src/shared/services/database.service.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, BatchGetCommand, BatchWriteCommand } from '@aws-sdk/lib-dynamodb';

class DatabaseService {
  private static instance: DatabaseService;
  private client: DynamoDBDocumentClient;
  
  private constructor() {
    const dynamoClient = new DynamoDBClient({
      region: process.env.AWS_REGION,
      maxAttempts: 3,
      requestTimeout: 5000,
      // Connection pooling configuration
      maxSockets: 50,
      keepAlive: true,
      keepAliveMsecs: 1000
    });
    
    this.client = DynamoDBDocumentClient.from(dynamoClient, {
      marshallOptions: {
        removeUndefinedValues: true,
        convertEmptyValues: false,
        convertClassInstanceToMap: false
      },
      unmarshallOptions: {
        wrapNumbers: false
      }
    });
  }
  
  static getInstance(): DatabaseService {
    if (!DatabaseService.instance) {
      DatabaseService.instance = new DatabaseService();
    }
    return DatabaseService.instance;
  }
  
  // Batch operations for better performance
  async batchGet(tableName: string, keys: any[]): Promise<any[]> {
    const chunks = this.chunkArray(keys, 100); // DynamoDB batch limit
    const results: any[] = [];
    
    for (const chunk of chunks) {
      const params = {
        RequestItems: {
          [tableName]: {
            Keys: chunk
          }
        }
      };
      
      const result = await this.client.send(new BatchGetCommand(params));
      
      if (result.Responses?.[tableName]) {
        results.push(...result.Responses[tableName]);
      }
      
      // Handle unprocessed keys
      if (result.UnprocessedKeys && Object.keys(result.UnprocessedKeys).length > 0) {
        // Exponential backoff retry logic here
        await this.sleep(100);
        // Retry unprocessed keys...
      }
    }
    
    return results;
  }
  
  async batchWrite(tableName: string, items: any[]): Promise<void> {
    const chunks = this.chunkArray(items, 25); // DynamoDB batch write limit
    
    for (const chunk of chunks) {
      const params = {
        RequestItems: {
          [tableName]: chunk.map(item => ({
            PutRequest: { Item: item }
          }))
        }
      };
      
      await this.client.send(new BatchWriteCommand(params));
    }
  }
  
  private chunkArray<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

export const databaseService = DatabaseService.getInstance();
```

### Query Optimization
```typescript
// src/shared/services/user.service.ts
import { databaseService } from './database.service';
import { QueryCommand, ScanCommand } from '@aws-sdk/lib-dynamodb';

export class UserService {
  private tableName = process.env.USERS_TABLE_NAME!;
  
  // Efficient query using GSI
  async getUsersByStatus(status: string, limit = 50): Promise<User[]> {
    const params = {
      TableName: this.tableName,
      IndexName: 'StatusIndex',
      KeyConditionExpression: '#status = :status',
      ExpressionAttributeNames: {
        '#status': 'status'
      },
      ExpressionAttributeValues: {
        ':status': status
      },
      Limit: limit,
      ScanIndexForward: false // Get newest first
    };
    
    const result = await databaseService.client.send(new QueryCommand(params));
    return result.Items as User[];
  }
  
  // Paginated query for large datasets
  async getUsersPaginated(
    lastEvaluatedKey?: any,
    limit = 50
  ): Promise<{ users: User[]; lastEvaluatedKey?: any }> {
    const params = {
      TableName: this.tableName,
      Limit: limit,
      ExclusiveStartKey: lastEvaluatedKey
    };
    
    const result = await databaseService.client.send(new ScanCommand(params));
    
    return {
      users: result.Items as User[],
      lastEvaluatedKey: result.LastEvaluatedKey
    };
  }
  
  // Parallel queries for better performance
  async getUsersFromMultipleStatuses(statuses: string[]): Promise<User[]> {
    const promises = statuses.map(status => this.getUsersByStatus(status));
    const results = await Promise.all(promises);
    
    // Flatten and deduplicate results
    const allUsers = results.flat();
    const uniqueUsers = allUsers.filter((user, index, self) => 
      index === self.findIndex(u => u.id === user.id)
    );
    
    return uniqueUsers;
  }
}
```

## Caching Strategies

### In-Memory Caching
```typescript
// src/shared/utils/cache.ts
interface CacheItem<T> {
  data: T;
  timestamp: number;
  ttl: number;
}

class InMemoryCache {
  private cache = new Map<string, CacheItem<any>>();
  private maxSize: number;
  
  constructor(maxSize = 1000) {
    this.maxSize = maxSize;
  }
  
  set<T>(key: string, data: T, ttlSeconds = 300): void {
    // Implement LRU eviction if cache is full
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl: ttlSeconds * 1000
    });
  }
  
  get<T>(key: string): T | null {
    const item = this.cache.get(key);
    
    if (!item) {
      return null;
    }
    
    // Check if item has expired
    if (Date.now() - item.timestamp > item.ttl) {
      this.cache.delete(key);
      return null;
    }
    
    return item.data as T;
  }
  
  delete(key: string): boolean {
    return this.cache.delete(key);
  }
  
  clear(): void {
    this.cache.clear();
  }
  
  size(): number {
    return this.cache.size;
  }
}

// Global cache instance (persists across warm invocations)
export const cache = new InMemoryCache(500);
```

### ElastiCache Integration
```typescript
// src/shared/services/redis.service.ts
import { createClient, RedisClientType } from 'redis';

class RedisService {
  private static instance: RedisService;
  private client: RedisClientType;
  private isConnected = false;
  
  private constructor() {
    this.client = createClient({
      url: process.env.REDIS_URL,
      socket: {
        connectTimeout: 5000,
        lazyConnect: true
      }
    });
    
    this.client.on('error', (err) => {
      console.error('Redis Client Error:', err);
      this.isConnected = false;
    });
    
    this.client.on('connect', () => {
      console.log('Redis Client Connected');
      this.isConnected = true;
    });
  }
  
  static getInstance(): RedisService {
    if (!RedisService.instance) {
      RedisService.instance = new RedisService();
    }
    return RedisService.instance;
  }
  
  async connect(): Promise<void> {
    if (!this.isConnected) {
      await this.client.connect();
    }
  }
  
  async get(key: string): Promise<string | null> {
    try {
      await this.connect();
      return await this.client.get(key);
    } catch (error) {
      console.error('Redis GET error:', error);
      return null;
    }
  }
  
  async set(key: string, value: string, ttlSeconds = 300): Promise<void> {
    try {
      await this.connect();
      await this.client.setEx(key, ttlSeconds, value);
    } catch (error) {
      console.error('Redis SET error:', error);
    }
  }
  
  async getJSON<T>(key: string): Promise<T | null> {
    const value = await this.get(key);
    if (!value) return null;
    
    try {
      return JSON.parse(value) as T;
    } catch (error) {
      console.error('JSON parse error:', error);
      return null;
    }
  }
  
  async setJSON<T>(key: string, data: T, ttlSeconds = 300): Promise<void> {
    try {
      const value = JSON.stringify(data);
      await this.set(key, value, ttlSeconds);
    } catch (error) {
      console.error('JSON stringify error:', error);
    }
  }
  
  async del(key: string): Promise<void> {
    try {
      await this.connect();
      await this.client.del(key);
    } catch (error) {
      console.error('Redis DEL error:', error);
    }
  }
}

export const redisService = RedisService.getInstance();
```

### Multi-Level Caching
```typescript
// src/shared/services/cached-user.service.ts
import { UserService } from './user.service';
import { cache } from '@/shared/utils/cache';
import { redisService } from './redis.service';

export class CachedUserService extends UserService {
  
  async getById(id: string): Promise<User | null> {
    const cacheKey = `user:${id}`;
    
    // Level 1: In-memory cache
    let user = cache.get<User>(cacheKey);
    if (user) {
      console.log('Cache hit: in-memory');
      return user;
    }
    
    // Level 2: Redis cache
    user = await redisService.getJSON<User>(cacheKey);
    if (user) {
      console.log('Cache hit: Redis');
      // Store in in-memory cache for faster access
      cache.set(cacheKey, user, 300);
      return user;
    }
    
    // Level 3: Database
    console.log('Cache miss: fetching from database');
    user = await super.getById(id);
    
    if (user) {
      // Store in both cache levels
      cache.set(cacheKey, user, 300);
      await redisService.setJSON(cacheKey, user, 600);
    }
    
    return user;
  }
  
  async update(id: string, updates: Partial<User>): Promise<User> {
    const user = await super.update(id, updates);
    
    // Invalidate caches
    const cacheKey = `user:${id}`;
    cache.delete(cacheKey);
    await redisService.del(cacheKey);
    
    // Pre-populate cache with updated user
    cache.set(cacheKey, user, 300);
    await redisService.setJSON(cacheKey, user, 600);
    
    return user;
  }
}
```

## Concurrency and Scaling

### Reserved Concurrency Configuration
```yaml
# serverless.yml
functions:
  # Critical function with guaranteed capacity
  processPayment:
    handler: src/functions/payments/process.handler
    reservedConcurrency: 100  # Reserve 100 concurrent executions
    events:
      - sqs:
          arn: ${self:custom.paymentQueueArn}
          batchSize: 1
  
  # Background processing with limited concurrency
  generateReport:
    handler: src/functions/reports/generate.handler
    reservedConcurrency: 5    # Limit to 5 concurrent executions
    timeout: 900              # 15 minutes
    events:
      - schedule: cron(0 2 * * ? *)  # Daily at 2 AM
  
  # High-throughput function
  logProcessor:
    handler: src/functions/logs/process.handler
    reservedConcurrency: 500  # High concurrency for log processing
    events:
      - stream:
          type: kinesis
          arn: ${self:custom.logStreamArn}
          batchSize: 100
          parallelizationFactor: 10
```

### Auto-Scaling Configuration
```yaml
# resources/auto-scaling.yml
Resources:
  # Application Auto Scaling Target
  DynamoDBReadCapacityScalableTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Properties:
      MaxCapacity: 1000
      MinCapacity: 5
      ResourceId: table/${self:custom.tableName}
      RoleARN: !GetAtt ApplicationAutoScalingDynamoDBRole.Arn
      ScalableDimension: dynamodb:table:ReadCapacityUnits
      ServiceNamespace: dynamodb
  
  # Scaling Policy
  DynamoDBReadCapacityScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      PolicyName: ReadAutoScalingPolicy
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref DynamoDBReadCapacityScalableTarget
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 70.0
        ScaleInCooldown: 60
        ScaleOutCooldown: 60
        PredefinedMetricSpecification:
          PredefinedMetricType: DynamoDBReadCapacityUtilization
```

## Monitoring and Alerting

### Performance Monitoring
```typescript
// src/shared/utils/metrics.ts
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

class MetricsService {
  private cloudwatch: CloudWatchClient;
  private namespace: string;
  
  constructor() {
    this.cloudwatch = new CloudWatchClient({ region: process.env.AWS_REGION });
    this.namespace = `${process.env.SERVICE_NAME}/${process.env.STAGE}`;
  }
  
  async putMetric(
    metricName: string,
    value: number,
    unit: string = 'Count',
    dimensions: Record<string, string> = {}
  ): Promise<void> {
    try {
      const params = {
        Namespace: this.namespace,
        MetricData: [
          {
            MetricName: metricName,
            Value: value,
            Unit: unit,
            Dimensions: Object.entries(dimensions).map(([Name, Value]) => ({
              Name,
              Value
            })),
            Timestamp: new Date()
          }
        ]
      };
      
      await this.cloudwatch.send(new PutMetricDataCommand(params));
    } catch (error) {
      console.error('Error putting metric:', error);
    }
  }
  
  async recordExecutionTime(functionName: string, duration: number): Promise<void> {
    await this.putMetric('ExecutionTime', duration, 'Milliseconds', {
      FunctionName: functionName
    });
  }
  
  async recordBusinessMetric(metricName: string, value: number): Promise<void> {
    await this.putMetric(metricName, value, 'Count');
  }
}

export const metricsService = new MetricsService();

// Performance decorator
export function measurePerformance(target: any, propertyName: string, descriptor: PropertyDescriptor) {
  const method = descriptor.value;
  
  descriptor.value = async function (...args: any[]) {
    const start = Date.now();
    
    try {
      const result = await method.apply(this, args);
      const duration = Date.now() - start;
      
      await metricsService.recordExecutionTime(propertyName, duration);
      
      return result;
    } catch (error) {
      const duration = Date.now() - start;
      await metricsService.recordExecutionTime(`${propertyName}_error`, duration);
      throw error;
    }
  };
}
```

### CloudWatch Alarms
```yaml
# resources/alarms.yml
Resources:
  HighErrorRateAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: ${self:service}-${self:provider.stage}-high-error-rate
      AlarmDescription: High error rate detected
      MetricName: Errors
      Namespace: AWS/Lambda
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 10
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: FunctionName
          Value: ${self:service}-${self:provider.stage}-api
      AlarmActions:
        - !Ref AlertsTopic
  
  HighLatencyAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: ${self:service}-${self:provider.stage}-high-latency
      AlarmDescription: High latency detected
      MetricName: Duration
      Namespace: AWS/Lambda
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 5000
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: FunctionName
          Value: ${self:service}-${self:provider.stage}-api
      AlarmActions:
        - !Ref AlertsTopic
  
  AlertsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: ${self:service}-${self:provider.stage}-alerts
      Subscription:
        - Protocol: email
          Endpoint: alerts@example.com
```

## Best Practices

### Performance Guidelines
- **Right-size Functions**: Allocate appropriate memory and timeout values
- **Minimize Cold Starts**: Use provisioned concurrency for critical functions
- **Optimize Bundle Size**: Use webpack and tree shaking to reduce package size
- **Connection Reuse**: Initialize clients outside handler functions
- **Implement Caching**: Use multi-level caching strategies
- **Batch Operations**: Use batch APIs when possible
- **Monitor Performance**: Track key metrics and set up alerts

### Scaling Considerations
- **Reserved Concurrency**: Use for critical functions that need guaranteed capacity
- **Auto-scaling**: Configure auto-scaling for downstream services
- **Queue Management**: Use SQS for handling traffic spikes
- **Circuit Breakers**: Implement circuit breakers for external dependencies
- **Graceful Degradation**: Design functions to degrade gracefully under load

### Cost Optimization
- **Memory Optimization**: Find the optimal memory allocation for cost/performance
- **Timeout Tuning**: Set appropriate timeouts to avoid unnecessary charges
- **Provisioned Concurrency**: Use judiciously as it incurs additional costs
- **Resource Cleanup**: Clean up unused resources and old function versions
- **Monitor Costs**: Use AWS Cost Explorer to track serverless costs