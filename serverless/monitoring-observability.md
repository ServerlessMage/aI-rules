# Serverless Monitoring and Observability

## Logging Strategies

### AWS Powertools Logger
```typescript
// src/shared/utils/logger.ts
import { Logger } from '@aws-lambda-powertools/logger';

// Create logger instance with configuration
export const logger = new Logger({
  serviceName: process.env.SERVICE_NAME || 'serverless-app',
  logLevel: (process.env.LOG_LEVEL as 'DEBUG' | 'INFO' | 'WARN' | 'ERROR') || 'INFO',
  environment: process.env.STAGE || 'dev',
  persistentLogAttributes: {
    version: process.env.SERVICE_VERSION || '1.0.0',
    region: process.env.AWS_REGION || 'us-west-2'
  }
});

// Enhanced logger with business context
export class EnhancedLogger {
  private baseLogger: Logger;
  
  constructor(logger: Logger) {
    this.baseLogger = logger;
  }
  
  // Add correlation ID to all logs
  withCorrelationId(correlationId: string): Logger {
    return this.baseLogger.createChild({
      persistentLogAttributes: {
        correlationId
      }
    });
  }
  
  // Add user context to logs
  withUserContext(userId: string, email?: string): Logger {
    return this.baseLogger.createChild({
      persistentLogAttributes: {
        userId,
        ...(email && { userEmail: email })
      }
    });
  }
  
  // Performance logging with structured data
  logPerformance(operation: string, duration: number, additionalData?: Record<string, any>): void {
    this.baseLogger.info('Performance metric', {
      operation,
      duration,
      performanceMetric: true,
      ...additionalData
    });
  }
  
  // Business event logging
  logBusinessEvent(event: string, data?: Record<string, any>): void {
    this.baseLogger.info('Business event', {
      businessEvent: event,
      businessMetric: true,
      ...data
    });
  }
  
  // Security event logging
  logSecurityEvent(event: string, severity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL', data?: Record<string, any>): void {
    this.baseLogger.warn('Security event', {
      securityEvent: event,
      severity,
      securityMetric: true,
      ...data
    });
  }
  
  // Error logging with context
  logError(message: string, error: Error, context?: Record<string, any>): void {
    this.baseLogger.error(message, {
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack
      },
      ...context
    });
  }
}

export const enhancedLogger = new EnhancedLogger(logger);
```

### Log Aggregation and Analysis
```yaml
# serverless.yml - CloudWatch Logs configuration
provider:
  name: aws
  runtime: nodejs20.x
  
  # CloudWatch Logs configuration
  logs:
    restApi: true
    frameworkLambda: true
  
  # Log retention
  logRetentionInDays: ${self:custom.logRetention.${self:provider.stage}}

custom:
  logRetention:
    dev: 7
    staging: 30
    prod: 90

resources:
  Resources:
    # Custom log group with encryption
    ApplicationLogGroup:
      Type: AWS::Logs::LogGroup
      Properties:
        LogGroupName: /aws/lambda/${self:service}-${self:provider.stage}
        RetentionInDays: ${self:custom.logRetention.${self:provider.stage}}
        KmsKeyId: !GetAtt LogsKMSKey.Arn
    
    # KMS key for log encryption
    LogsKMSKey:
      Type: AWS::KMS::Key
      Properties:
        Description: KMS key for CloudWatch Logs encryption
        KeyPolicy:
          Version: '2012-10-17'
          Statement:
            - Sid: Enable IAM User Permissions
              Effect: Allow
              Principal:
                AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
              Action: 'kms:*'
              Resource: '*'
            - Sid: Allow CloudWatch Logs
              Effect: Allow
              Principal:
                Service: !Sub 'logs.${AWS::Region}.amazonaws.com'
              Action:
                - kms:Encrypt
                - kms:Decrypt
                - kms:ReEncrypt*
                - kms:GenerateDataKey*
                - kms:DescribeKey
              Resource: '*'
    
    # Log subscription filter for error alerting
    ErrorLogFilter:
      Type: AWS::Logs::SubscriptionFilter
      Properties:
        LogGroupName: !Ref ApplicationLogGroup
        FilterPattern: '{ $.level = "error" }'
        DestinationArn: !GetAtt ErrorProcessorFunction.Arn
    
    # Lambda function to process error logs
    ErrorProcessorFunction:
      Type: AWS::Lambda::Function
      Properties:
        FunctionName: ${self:service}-${self:provider.stage}-error-processor
        Runtime: nodejs20.x
        Handler: index.handler
        Code:
          ZipFile: |
            exports.handler = async (event) => {
              // Process error logs and send alerts
              console.log('Error log received:', JSON.stringify(event, null, 2));
            };
```

## Metrics and Monitoring

### Custom CloudWatch Metrics
```typescript
// src/shared/services/metrics.service.ts
import { CloudWatchClient, PutMetricDataCommand, MetricDatum } from '@aws-sdk/client-cloudwatch';

export class MetricsService {
  private cloudwatch: CloudWatchClient;
  private namespace: string;
  private defaultDimensions: Record<string, string>;
  
  constructor() {
    this.cloudwatch = new CloudWatchClient({
      region: process.env.AWS_REGION
    });
    this.namespace = `${process.env.SERVICE_NAME}/${process.env.STAGE}`;
    this.defaultDimensions = {
      Service: process.env.SERVICE_NAME || 'serverless-app',
      Stage: process.env.STAGE || 'dev',
      Version: process.env.SERVICE_VERSION || '1.0.0'
    };
  }
  
  async putMetric(
    metricName: string,
    value: number,
    unit: string = 'Count',
    dimensions: Record<string, string> = {}
  ): Promise<void> {
    const metricData: MetricDatum = {
      MetricName: metricName,
      Value: value,
      Unit: unit,
      Timestamp: new Date(),
      Dimensions: Object.entries({ ...this.defaultDimensions, ...dimensions })
        .map(([Name, Value]) => ({ Name, Value }))
    };
    
    try {
      await this.cloudwatch.send(new PutMetricDataCommand({
        Namespace: this.namespace,
        MetricData: [metricData]
      }));
    } catch (error) {
      console.error('Failed to put metric:', error);
    }
  }
  
  async putMultipleMetrics(metrics: Array<{
    name: string;
    value: number;
    unit?: string;
    dimensions?: Record<string, string>;
  }>): Promise<void> {
    const metricData: MetricDatum[] = metrics.map(metric => ({
      MetricName: metric.name,
      Value: metric.value,
      Unit: metric.unit || 'Count',
      Timestamp: new Date(),
      Dimensions: Object.entries({ ...this.defaultDimensions, ...metric.dimensions })
        .map(([Name, Value]) => ({ Name, Value }))
    }));
    
    try {
      await this.cloudwatch.send(new PutMetricDataCommand({
        Namespace: this.namespace,
        MetricData: metricData
      }));
    } catch (error) {
      console.error('Failed to put metrics:', error);
    }
  }
  
  // Business metrics
  async recordUserAction(action: string, userId?: string): Promise<void> {
    await this.putMetric('UserAction', 1, 'Count', {
      Action: action,
      ...(userId && { UserId: userId })
    });
  }
  
  async recordApiCall(endpoint: string, method: string, statusCode: number, duration: number): Promise<void> {
    await this.putMultipleMetrics([
      {
        name: 'ApiCall',
        value: 1,
        dimensions: { Endpoint: endpoint, Method: method, StatusCode: statusCode.toString() }
      },
      {
        name: 'ApiLatency',
        value: duration,
        unit: 'Milliseconds',
        dimensions: { Endpoint: endpoint, Method: method }
      }
    ]);
  }
  
  async recordError(errorType: string, functionName?: string): Promise<void> {
    await this.putMetric('ApplicationError', 1, 'Count', {
      ErrorType: errorType,
      ...(functionName && { FunctionName: functionName })
    });
  }
  
  async recordPerformance(operation: string, duration: number, success: boolean): Promise<void> {
    await this.putMultipleMetrics([
      {
        name: 'OperationDuration',
        value: duration,
        unit: 'Milliseconds',
        dimensions: { Operation: operation }
      },
      {
        name: 'OperationSuccess',
        value: success ? 1 : 0,
        dimensions: { Operation: operation }
      }
    ]);
  }
}

export const metricsService = new MetricsService();
```

### Performance Monitoring Middleware
```typescript
// src/shared/middleware/monitoring.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import { MiddlewareObj } from '@middy/core';
import { metricsService } from '@/shared/services/metrics.service';
import { logger, enhancedLogger } from '@/shared/utils/logger';

interface MonitoringContext {
  startTime: number;
  endpoint: string;
  method: string;
  correlationId: string;
}

export const monitoringMiddleware = (): MiddlewareObj<APIGatewayProxyEvent, APIGatewayProxyResult> => {
  return {
    before: async (request) => {
      const { event } = request;
      const startTime = Date.now();
      const correlationId = event.headers['x-correlation-id'] || 
                           event.requestContext.requestId;
      
      // Create child logger with correlation ID
      const contextLogger = enhancedLogger.withCorrelationId(correlationId);
      
      // Store monitoring context
      request.internal = {
        ...request.internal,
        monitoring: {
          startTime,
          endpoint: event.path,
          method: event.httpMethod,
          correlationId
        },
        logger: contextLogger
      };
      
      contextLogger.info('Request started', {
        method: event.httpMethod,
        path: event.path,
        userAgent: event.headers['User-Agent'],
        sourceIp: event.requestContext.identity.sourceIp,
        requestId: event.requestContext.requestId
      });
    },
    
    after: async (request) => {
      const { event, response, internal } = request;
      const monitoring = internal.monitoring as MonitoringContext;
      const contextLogger = internal.logger as Logger;
      const duration = Date.now() - monitoring.startTime;
      
      // Record metrics
      await metricsService.recordApiCall(
        monitoring.endpoint,
        monitoring.method,
        response?.statusCode || 200,
        duration
      );
      
      contextLogger.info('Request completed', {
        statusCode: response?.statusCode,
        duration,
        requestId: event.requestContext.requestId
      });
      
      // Log performance metrics
      enhancedLogger.logPerformance('API Request', duration, {
        endpoint: monitoring.endpoint,
        method: monitoring.method,
        statusCode: response?.statusCode,
        requestId: event.requestContext.requestId
      });
    },
    
    onError: async (request) => {
      const { event, error, internal } = request;
      const monitoring = internal.monitoring as MonitoringContext;
      const contextLogger = internal.logger as Logger;
      const duration = Date.now() - monitoring.startTime;
      
      // Record error metrics
      await metricsService.recordError(error.name, process.env.AWS_LAMBDA_FUNCTION_NAME);
      
      enhancedLogger.logError('Request failed', error, {
        requestId: event.requestContext.requestId,
        duration,
        endpoint: monitoring.endpoint,
        method: monitoring.method
      });
    }
  };
};
```

## Distributed Tracing

### AWS X-Ray Integration
```typescript
// src/shared/utils/tracing.ts
import AWSXRay from 'aws-xray-sdk-core';
import AWS from 'aws-sdk';

// Capture AWS SDK calls
const capturedAWS = AWSXRay.captureAWS(AWS);

// Capture HTTP requests
const capturedHTTP = AWSXRay.captureHTTPs(require('https'));

export class TracingService {
  
  // Create custom subsegment
  static async traceAsyncOperation<T>(
    name: string,
    operation: () => Promise<T>,
    metadata?: Record<string, any>
  ): Promise<T> {
    const segment = AWSXRay.getSegment();
    
    if (!segment) {
      // If no active segment, just execute the operation
      return operation();
    }
    
    const subsegment = segment.addNewSubsegment(name);
    
    if (metadata) {
      subsegment.addMetadata('custom', metadata);
    }
    
    try {
      const result = await operation();
      subsegment.close();
      return result;
    } catch (error) {
      subsegment.addError(error);
      subsegment.close();
      throw error;
    }
  }
  
  // Add annotations for filtering
  static addAnnotation(key: string, value: string | number | boolean): void {
    const segment = AWSXRay.getSegment();
    if (segment) {
      segment.addAnnotation(key, value);
    }
  }
  
  // Add metadata for detailed information
  static addMetadata(namespace: string, data: Record<string, any>): void {
    const segment = AWSXRay.getSegment();
    if (segment) {
      segment.addMetadata(namespace, data);
    }
  }
  
  // Trace database operations
  static async traceDatabaseOperation<T>(
    operation: string,
    query: string,
    params: any[],
    executor: () => Promise<T>
  ): Promise<T> {
    return this.traceAsyncOperation(
      `Database ${operation}`,
      executor,
      {
        query: query.substring(0, 1000), // Truncate long queries
        paramCount: params.length
      }
    );
  }
  
  // Trace external API calls
  static async traceExternalCall<T>(
    serviceName: string,
    endpoint: string,
    method: string,
    executor: () => Promise<T>
  ): Promise<T> {
    return this.traceAsyncOperation(
      `External ${serviceName}`,
      executor,
      {
        endpoint,
        method
      }
    );
  }
}

// Decorator for automatic tracing
export function trace(operationName?: string) {
  return function (target: any, propertyName: string, descriptor: PropertyDescriptor) {
    const method = descriptor.value;
    const name = operationName || `${target.constructor.name}.${propertyName}`;
    
    descriptor.value = async function (...args: any[]) {
      return TracingService.traceAsyncOperation(
        name,
        () => method.apply(this, args),
        {
          className: target.constructor.name,
          methodName: propertyName,
          argCount: args.length
        }
      );
    };
  };
}
```

### Powertools Integration with Middy
```typescript
// src/shared/middleware/powertools.ts
import { injectLambdaContext } from '@aws-lambda-powertools/logger/middleware';
import { captureLambdaHandler } from '@aws-lambda-powertools/tracer/middleware';
import { logMetrics } from '@aws-lambda-powertools/metrics/middleware';
import { Tracer } from '@aws-lambda-powertools/tracer';
import { Metrics } from '@aws-lambda-powertools/metrics';
import { logger } from '@/shared/utils/logger';

// Initialize Powertools components
export const tracer = new Tracer({
  serviceName: process.env.SERVICE_NAME || 'serverless-app',
  captureHTTPsRequests: true,
  captureResponse: true
});

export const metrics = new Metrics({
  namespace: `${process.env.SERVICE_NAME}/${process.env.STAGE}`,
  serviceName: process.env.SERVICE_NAME || 'serverless-app',
  defaultDimensions: {
    environment: process.env.STAGE || 'dev',
    version: process.env.SERVICE_VERSION || '1.0.0'
  }
});

// Enhanced middleware factory with Powertools
export const powertoolsMiddleware = () => {
  return [
    // Inject Lambda context into logger
    injectLambdaContext(logger, {
      logEvent: process.env.LOG_LEVEL === 'DEBUG',
      clearState: true
    }),
    
    // Capture Lambda handler for tracing
    captureLambdaHandler(tracer),
    
    // Log metrics automatically
    logMetrics(metrics, {
      captureColdStartMetric: true,
      throwOnEmptyMetrics: false
    })
  ];
};

// Custom correlation middleware for Powertools
export const correlationMiddleware = () => {
  return {
    before: async (request: any) => {
      const { event } = request;
      
      // Extract correlation ID
      const correlationId = event.headers['x-correlation-id'] || 
                           event.requestContext?.requestId;
      
      // Add to logger context
      logger.appendKeys({
        correlationId,
        requestId: event.requestContext?.requestId,
        userId: event.requestContext?.authorizer?.principalId
      });
      
      // Add to tracer annotations
      tracer.putAnnotation('correlationId', correlationId);
      if (event.requestContext?.authorizer?.principalId) {
        tracer.putAnnotation('userId', event.requestContext.authorizer.principalId);
      }
      
      // Add to metrics dimensions
      metrics.addDimension('correlationId', correlationId);
    },
    
    after: async (request: any) => {
      // Clear context for next invocation
      logger.removeKeys(['correlationId', 'requestId', 'userId']);
    }
  };
};
```

## Health Checks and Synthetic Monitoring

### Health Check Endpoint
```typescript
// src/functions/health/check.ts
import { APIGatewayProxyHandler } from 'aws-lambda';
import { databaseService } from '@/shared/services/database.service';
import { redisService } from '@/shared/services/redis.service';
import { logger, enhancedLogger } from '@/shared/utils/logger';
import { tracer, metrics } from '@/shared/middleware/powertools';

interface HealthCheckResult {
  status: 'healthy' | 'unhealthy' | 'degraded';
  timestamp: string;
  version: string;
  checks: {
    database: HealthStatus;
    cache: HealthStatus;
    external: HealthStatus;
  };
  uptime: number;
}

interface HealthStatus {
  status: 'up' | 'down' | 'degraded';
  responseTime?: number;
  error?: string;
}

const startTime = Date.now();

export const handler: APIGatewayProxyHandler = async () => {
  const timestamp = new Date().toISOString();
  const uptime = Date.now() - startTime;
  
  logger.info('Health check requested');
  
  // Create subsegment for health checks
  const subsegment = tracer.getSegment()?.addNewSubsegment('health-checks');
  
  try {
    // Perform health checks with tracing
    const checks = await Promise.allSettled([
      tracer.captureAsyncFunc('database-check', checkDatabase),
      tracer.captureAsyncFunc('cache-check', checkCache),
      tracer.captureAsyncFunc('external-check', checkExternalServices)
    ]);
    
    const [databaseResult, cacheResult, externalResult] = checks;
    
    const healthResult: HealthCheckResult = {
      status: 'healthy',
      timestamp,
      version: process.env.SERVICE_VERSION || '1.0.0',
      uptime,
      checks: {
        database: databaseResult.status === 'fulfilled' ? databaseResult.value : { status: 'down', error: databaseResult.reason?.message },
        cache: cacheResult.status === 'fulfilled' ? cacheResult.value : { status: 'down', error: cacheResult.reason?.message },
        external: externalResult.status === 'fulfilled' ? externalResult.value : { status: 'down', error: externalResult.reason?.message }
      }
    };
    
    // Determine overall status
    const statuses = Object.values(healthResult.checks).map(check => check.status);
    
    if (statuses.includes('down')) {
      healthResult.status = 'unhealthy';
    } else if (statuses.includes('degraded')) {
      healthResult.status = 'degraded';
    }
    
    // Add metrics
    metrics.addMetric('HealthCheckStatus', 'Count', healthResult.status === 'healthy' ? 1 : 0);
    metrics.addMetric('HealthCheckUptime', 'Seconds', uptime / 1000);
    
    // Add tracing annotations
    tracer.putAnnotation('healthStatus', healthResult.status);
    tracer.putMetadata('healthCheck', healthResult.checks);
    
    const statusCode = healthResult.status === 'healthy' ? 200 : 503;
    
    logger.info('Health check completed', {
      status: healthResult.status,
      checks: healthResult.checks,
      uptime
    });
    
    return {
      statusCode,
      headers: {
        'Content-Type': 'application/json',
        'Cache-Control': 'no-cache'
      },
      body: JSON.stringify(healthResult)
    };
  } finally {
    subsegment?.close();
  }
};

async function checkDatabase(): Promise<HealthStatus> {
  const start = Date.now();
  
  try {
    // Simple database connectivity check
    await databaseService.query('SELECT 1');
    
    const responseTime = Date.now() - start;
    
    return {
      status: responseTime > 1000 ? 'degraded' : 'up',
      responseTime
    };
  } catch (error) {
    return {
      status: 'down',
      error: error.message,
      responseTime: Date.now() - start
    };
  }
}

async function checkCache(): Promise<HealthStatus> {
  const start = Date.now();
  
  try {
    await redisService.set('health-check', 'ok', 10);
    const value = await redisService.get('health-check');
    
    const responseTime = Date.now() - start;
    
    if (value !== 'ok') {
      return {
        status: 'degraded',
        responseTime,
        error: 'Cache read/write mismatch'
      };
    }
    
    return {
      status: responseTime > 500 ? 'degraded' : 'up',
      responseTime
    };
  } catch (error) {
    return {
      status: 'down',
      error: error.message,
      responseTime: Date.now() - start
    };
  }
}

async function checkExternalServices(): Promise<HealthStatus> {
  const start = Date.now();
  
  try {
    // Check external API dependencies
    const response = await fetch('https://api.external-service.com/health', {
      method: 'GET',
      timeout: 5000
    });
    
    const responseTime = Date.now() - start;
    
    if (!response.ok) {
      return {
        status: 'degraded',
        responseTime,
        error: `External service returned ${response.status}`
      };
    }
    
    return {
      status: responseTime > 2000 ? 'degraded' : 'up',
      responseTime
    };
  } catch (error) {
    return {
      status: 'down',
      error: error.message,
      responseTime: Date.now() - start
    };
  }
}
```

### Synthetic Monitoring
```typescript
// src/functions/synthetic/monitor.ts
import { ScheduledHandler } from 'aws-lambda';
import { logger, enhancedLogger } from '@/shared/utils/logger';
import { tracer, metrics } from '@/shared/middleware/powertools';

interface SyntheticTest {
  name: string;
  url: string;
  method: string;
  expectedStatus: number;
  timeout: number;
  headers?: Record<string, string>;
  body?: string;
}

const tests: SyntheticTest[] = [
  {
    name: 'API Health Check',
    url: `${process.env.API_BASE_URL}/health`,
    method: 'GET',
    expectedStatus: 200,
    timeout: 5000
  },
  {
    name: 'User Login',
    url: `${process.env.API_BASE_URL}/auth/login`,
    method: 'POST',
    expectedStatus: 200,
    timeout: 10000,
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      email: process.env.SYNTHETIC_TEST_EMAIL,
      password: process.env.SYNTHETIC_TEST_PASSWORD
    })
  },
  {
    name: 'Get User Profile',
    url: `${process.env.API_BASE_URL}/users/me`,
    method: 'GET',
    expectedStatus: 200,
    timeout: 5000,
    headers: {
      'Authorization': `Bearer ${process.env.SYNTHETIC_TEST_TOKEN}`
    }
  }
];

export const handler: ScheduledHandler = async (event) => {
  logger.info('Starting synthetic monitoring tests', {
    testCount: tests.length,
    scheduledEvent: event.source
  });
  
  // Create subsegment for synthetic tests
  const subsegment = tracer.getSegment()?.addNewSubsegment('synthetic-tests');
  
  try {
    const results = await Promise.allSettled(
      tests.map(test => tracer.captureAsyncFunc(`test-${test.name}`, () => runSyntheticTest(test)))
    );
    
    let successCount = 0;
    let failureCount = 0;
    
    results.forEach((result, index) => {
      const test = tests[index];
      
      if (result.status === 'fulfilled') {
        successCount++;
        logger.info(`Synthetic test passed: ${test.name}`, {
          testName: test.name,
          duration: result.value.duration,
          statusCode: result.value.statusCode
        });
        
        // Add individual test metrics
        metrics.addMetric(`SyntheticTest_${test.name}_Success`, 'Count', 1);
        metrics.addMetric(`SyntheticTest_${test.name}_Duration`, 'Milliseconds', result.value.duration);
      } else {
        failureCount++;
        enhancedLogger.logError(`Synthetic test failed: ${test.name}`, result.reason, {
          testName: test.name
        });
        
        metrics.addMetric(`SyntheticTest_${test.name}_Failure`, 'Count', 1);
      }
    });
    
    // Record aggregate metrics
    metrics.addMetric('SyntheticTestSuccess', 'Count', successCount);
    metrics.addMetric('SyntheticTestFailure', 'Count', failureCount);
    metrics.addMetric('SyntheticTestSuccessRate', 'Percent', (successCount / tests.length) * 100);
    
    // Add tracing annotations
    tracer.putAnnotation('syntheticTestsTotal', tests.length);
    tracer.putAnnotation('syntheticTestsSuccess', successCount);
    tracer.putAnnotation('syntheticTestsFailure', failureCount);
    
    logger.info('Synthetic monitoring completed', {
      successCount,
      failureCount,
      successRate: (successCount / tests.length) * 100
    });
  } finally {
    subsegment?.close();
  }
};

async function runSyntheticTest(test: SyntheticTest): Promise<{
  duration: number;
  statusCode: number;
}> {
  const start = Date.now();
  
  try {
    // Add test metadata to tracing
    tracer.putMetadata('syntheticTest', {
      name: test.name,
      url: test.url,
      method: test.method,
      expectedStatus: test.expectedStatus
    });
    
    const response = await fetch(test.url, {
      method: test.method,
      headers: test.headers,
      body: test.body,
      timeout: test.timeout
    });
    
    const duration = Date.now() - start;
    
    // Add response metadata
    tracer.putMetadata('syntheticTestResponse', {
      statusCode: response.status,
      duration,
      headers: Object.fromEntries(response.headers.entries())
    });
    
    if (response.status !== test.expectedStatus) {
      throw new Error(`Expected status ${test.expectedStatus}, got ${response.status}`);
    }
    
    return {
      duration,
      statusCode: response.status
    };
  } catch (error) {
    const duration = Date.now() - start;
    
    // Add error to trace
    tracer.addErrorAsMetadata(error as Error);
    
    throw new Error(`Synthetic test failed: ${error.message}`);
  }
}
```

## Alerting and Notifications

### CloudWatch Alarms
```yaml
# resources/alarms.yml
Resources:
  # High error rate alarm
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
      TreatMissingData: notBreaching
      Dimensions:
        - Name: FunctionName
          Value: ${self:service}-${self:provider.stage}-api
      AlarmActions:
        - !Ref CriticalAlertsTopic
  
  # High latency alarm
  HighLatencyAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: ${self:service}-${self:provider.stage}-high-latency
      AlarmDescription: High latency detected
      MetricName: Duration
      Namespace: AWS/Lambda
      Statistic: Average
      Period: 300
      EvaluationPeriods: 3
      Threshold: 5000
      ComparisonOperator: GreaterThanThreshold
      TreatMissingData: notBreaching
      Dimensions:
        - Name: FunctionName
          Value: ${self:service}-${self:provider.stage}-api
      AlarmActions:
        - !Ref WarningAlertsTopic
  
  # Throttling alarm
  ThrottlingAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: ${self:service}-${self:provider.stage}-throttling
      AlarmDescription: Function throttling detected
      MetricName: Throttles
      Namespace: AWS/Lambda
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold
      TreatMissingData: notBreaching
      Dimensions:
        - Name: FunctionName
          Value: ${self:service}-${self:provider.stage}-api
      AlarmActions:
        - !Ref CriticalAlertsTopic
  
  # Custom business metric alarm
  LowUserRegistrationAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: ${self:service}-${self:provider.stage}-low-user-registration
      AlarmDescription: User registration rate is below threshold
      MetricName: UserAction
      Namespace: ${self:service}/${self:provider.stage}
      Statistic: Sum
      Period: 3600  # 1 hour
      EvaluationPeriods: 2
      Threshold: 5
      ComparisonOperator: LessThanThreshold
      TreatMissingData: breaching
      Dimensions:
        - Name: Action
          Value: register
      AlarmActions:
        - !Ref BusinessAlertsTopic
  
  # SNS Topics for different alert levels
  CriticalAlertsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: ${self:service}-${self:provider.stage}-critical-alerts
      DisplayName: Critical Alerts
  
  WarningAlertsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: ${self:service}-${self:provider.stage}-warning-alerts
      DisplayName: Warning Alerts
  
  BusinessAlertsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: ${self:service}-${self:provider.stage}-business-alerts
      DisplayName: Business Alerts
  
  # Email subscriptions
  CriticalAlertsEmailSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: email
      TopicArn: !Ref CriticalAlertsTopic
      Endpoint: ${self:custom.alerting.criticalEmail}
  
  # Slack webhook subscription
  CriticalAlertsSlackSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: https
      TopicArn: !Ref CriticalAlertsTopic
      Endpoint: ${self:custom.alerting.slackWebhook}
```

### Alert Processing Function
```typescript
// src/functions/alerts/processor.ts
import { SNSHandler } from 'aws-lambda';
import { logger } from '@/shared/utils/logger';

interface CloudWatchAlarm {
  AlarmName: string;
  AlarmDescription: string;
  NewStateValue: 'ALARM' | 'OK' | 'INSUFFICIENT_DATA';
  NewStateReason: string;
  StateChangeTime: string;
  Region: string;
  MetricName: string;
  Namespace: string;
  Statistic: string;
  Dimensions: Array<{ name: string; value: string }>;
  Period: number;
  EvaluationPeriods: number;
  Threshold: number;
  ComparisonOperator: string;
}

export const handler: SNSHandler = async (event) => {
  for (const record of event.Records) {
    try {
      const message = JSON.parse(record.Sns.Message) as CloudWatchAlarm;
      
      logger.info('Processing CloudWatch alarm', {
        alarmName: message.AlarmName,
        state: message.NewStateValue,
        reason: message.NewStateReason
      });
      
      // Process different types of alarms
      if (message.NewStateValue === 'ALARM') {
        await handleAlarmState(message);
      } else if (message.NewStateValue === 'OK') {
        await handleOkState(message);
      }
      
    } catch (error) {
      logger.error('Error processing alarm', error, {
        snsMessage: record.Sns.Message
      });
    }
  }
};

async function handleAlarmState(alarm: CloudWatchAlarm): Promise<void> {
  const severity = determineSeverity(alarm);
  
  // Send to appropriate channels based on severity
  if (severity === 'critical') {
    await sendSlackAlert(alarm);
    await sendPagerDutyAlert(alarm);
  } else if (severity === 'warning') {
    await sendSlackAlert(alarm);
  }
  
  // Log for audit trail
  logger.warn('Alarm triggered', {
    alarmName: alarm.AlarmName,
    severity,
    metric: alarm.MetricName,
    threshold: alarm.Threshold,
    reason: alarm.NewStateReason
  });
}

async function handleOkState(alarm: CloudWatchAlarm): Promise<void> {
  logger.info('Alarm resolved', {
    alarmName: alarm.AlarmName,
    metric: alarm.MetricName
  });
  
  // Send resolution notification
  await sendSlackResolution(alarm);
}

function determineSeverity(alarm: CloudWatchAlarm): 'critical' | 'warning' | 'info' {
  // Determine severity based on alarm name or metric
  if (alarm.AlarmName.includes('critical') || 
      alarm.MetricName === 'Errors' || 
      alarm.MetricName === 'Throttles') {
    return 'critical';
  }
  
  if (alarm.AlarmName.includes('warning') || 
      alarm.MetricName === 'Duration') {
    return 'warning';
  }
  
  return 'info';
}

async function sendSlackAlert(alarm: CloudWatchAlarm): Promise<void> {
  // Implementation for Slack webhook
  const webhookUrl = process.env.SLACK_WEBHOOK_URL;
  if (!webhookUrl) return;
  
  const color = determineSeverity(alarm) === 'critical' ? 'danger' : 'warning';
  
  const payload = {
    attachments: [
      {
        color,
        title: `🚨 ${alarm.AlarmName}`,
        text: alarm.AlarmDescription,
        fields: [
          {
            title: 'Metric',
            value: alarm.MetricName,
            short: true
          },
          {
            title: 'Threshold',
            value: alarm.Threshold.toString(),
            short: true
          },
          {
            title: 'Reason',
            value: alarm.NewStateReason,
            short: false
          }
        ],
        timestamp: Math.floor(new Date(alarm.StateChangeTime).getTime() / 1000)
      }
    ]
  };
  
  try {
    await fetch(webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });
  } catch (error) {
    logger.error('Failed to send Slack alert', error);
  }
}

async function sendSlackResolution(alarm: CloudWatchAlarm): Promise<void> {
  const webhookUrl = process.env.SLACK_WEBHOOK_URL;
  if (!webhookUrl) return;
  
  const payload = {
    attachments: [
      {
        color: 'good',
        title: `✅ ${alarm.AlarmName} - RESOLVED`,
        text: 'Alarm has returned to OK state',
        timestamp: Math.floor(new Date(alarm.StateChangeTime).getTime() / 1000)
      }
    ]
  };
  
  try {
    await fetch(webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });
  } catch (error) {
    logger.error('Failed to send Slack resolution', error);
  }
}

async function sendPagerDutyAlert(alarm: CloudWatchAlarm): Promise<void> {
  // Implementation for PagerDuty integration
  const integrationKey = process.env.PAGERDUTY_INTEGRATION_KEY;
  if (!integrationKey) return;
  
  const payload = {
    routing_key: integrationKey,
    event_action: 'trigger',
    dedup_key: alarm.AlarmName,
    payload: {
      summary: `${alarm.AlarmName}: ${alarm.NewStateReason}`,
      source: 'CloudWatch',
      severity: 'critical',
      component: alarm.MetricName,
      group: alarm.Namespace,
      custom_details: {
        alarm_description: alarm.AlarmDescription,
        threshold: alarm.Threshold,
        comparison_operator: alarm.ComparisonOperator,
        state_change_time: alarm.StateChangeTime
      }
    }
  };
  
  try {
    await fetch('https://events.pagerduty.com/v2/enqueue', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });
  } catch (error) {
    logger.error('Failed to send PagerDuty alert', error);
  }
}
```

## Best Practices

### Monitoring Guidelines
- **Implement comprehensive logging** with structured formats
- **Use correlation IDs** to trace requests across services
- **Monitor business metrics** not just technical metrics
- **Set up proactive alerting** based on SLIs and SLOs
- **Implement health checks** for all critical dependencies
- **Use distributed tracing** for complex workflows
- **Monitor costs** alongside performance metrics

### Observability Strategy
- **Define SLIs and SLOs** for your service
- **Implement the three pillars**: logs, metrics, and traces
- **Use dashboards** for real-time visibility
- **Set up synthetic monitoring** for critical user journeys
- **Implement proper error handling** and reporting
- **Monitor security events** and anomalies
- **Regular review and tuning** of alerts and thresholds

### Performance Monitoring
- **Track cold start metrics** and optimize accordingly
- **Monitor memory usage** and right-size functions
- **Track database performance** and connection pooling
- **Monitor external API dependencies**
- **Implement circuit breakers** for resilience
- **Track business KPIs** alongside technical metrics