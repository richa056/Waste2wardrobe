# Re-Thread AI Technical Design Document

## Architecture Overview

Re-Thread AI is built on a serverless, event-driven AWS architecture that automatically processes fabric waste images and generates sustainable product suggestions. The system leverages AWS managed services for scalability, reliability, and cost-effectiveness.

## System Architecture

### High-Level Architecture Diagram

```
[Web App] → [S3 Upload] → [Lambda Trigger] → [Rekognition] → [Bedrock] → [DynamoDB] → [QuickSight]
    ↓           ↓              ↓               ↓            ↓           ↓            ↓
[CloudFront] [Event Notification] [Processing] [Analysis] [AI Suggestions] [Storage] [Dashboard]
```

### Core Components

1. **Frontend Application** - React-based web application
2. **AWS S3** - Image storage and static website hosting
3. **AWS Lambda** - Serverless processing functions
4. **Amazon Rekognition** - AI-powered image analysis
5. **Amazon Bedrock** - Generative AI for product suggestions
6. **Amazon DynamoDB** - NoSQL database for metadata storage
7. **Amazon QuickSight** - Business intelligence dashboard
8. **Amazon CloudFront** - Content delivery network

## Detailed Component Design

### 1. Frontend Application

#### Technology Stack
- **Framework**: React 18 with TypeScript
- **State Management**: Redux Toolkit
- **UI Library**: Material-UI (MUI)
- **Build Tool**: Vite
- **Hosting**: S3 Static Website + CloudFront

#### Key Features
- Drag-and-drop image upload interface
- Real-time processing status updates
- Fabric inventory management
- Product suggestion gallery
- Sustainability metrics dashboard

#### File Structure
```
src/
├── components/
│   ├── Upload/
│   ├── Dashboard/
│   ├── Inventory/
│   └── Suggestions/
├── services/
│   ├── api.ts
│   ├── s3Upload.ts
│   └── websocket.ts
├── store/
│   ├── slices/
│   └── index.ts
└── utils/
```

### 2. AWS S3 Configuration

#### Bucket Structure
```
re-thread-ai-images/
├── uploads/
│   └── {factory-id}/
│       └── {year}/{month}/{day}/
│           └── {timestamp}-{filename}
├── processed/
│   └── {factory-id}/
│       └── thumbnails/
└── static-website/
    └── build/
```

#### S3 Event Configuration
- **Event Type**: `s3:ObjectCreated:*`
- **Prefix Filter**: `uploads/`
- **Suffix Filter**: `.jpg, .jpeg, .png, .webp`
- **Destination**: Lambda function ARN

#### Security Settings
- **Bucket Policy**: Restrict access to authenticated users
- **CORS Configuration**: Allow frontend domain access
- **Encryption**: AES-256 server-side encryption
- **Versioning**: Enabled for data protection

### 3. AWS Lambda Functions

#### 3.1 Image Processing Function (`processImage`)

**Runtime**: Node.js 18.x
**Memory**: 1024 MB
**Timeout**: 5 minutes

```javascript
// Function structure
exports.handler = async (event) => {
  // 1. Parse S3 event
  // 2. Extract image metadata
  // 3. Call Rekognition for analysis
  // 4. Call Bedrock for suggestions
  // 5. Store results in DynamoDB
  // 6. Send WebSocket notification
};
```

**Environment Variables**:
- `REKOGNITION_REGION`
- `BEDROCK_REGION`
- `DYNAMODB_TABLE_NAME`
- `WEBSOCKET_API_ENDPOINT`

**IAM Permissions**:
- S3: GetObject, PutObject
- Rekognition: DetectLabels, DetectText
- Bedrock: InvokeModel
- DynamoDB: PutItem, UpdateItem
- API Gateway: POST (WebSocket)

#### 3.2 API Gateway Functions

**Function**: `fabricAPI`
- GET `/fabrics` - List fabric inventory
- GET `/fabrics/{id}` - Get fabric details
- PUT `/fabrics/{id}` - Update fabric status
- GET `/suggestions/{fabricId}` - Get product suggestions
- GET `/analytics` - Get sustainability metrics

### 4. Amazon Rekognition Integration

#### Custom Labels Model
```json
{
  "labels": [
    "cotton", "polyester", "silk", "denim", "wool",
    "linen", "canvas", "leather", "synthetic",
    "solid_color", "striped", "floral", "geometric",
    "small_piece", "medium_piece", "large_piece",
    "excellent_condition", "good_condition", "fair_condition"
  ]
}
```

#### Analysis Configuration
- **Max Labels**: 20
- **Min Confidence**: 75%
- **Features**: Labels, Text Detection
- **Image Size**: Auto-resize to optimize costs

#### Processing Flow
```javascript
const rekognitionParams = {
  Image: {
    S3Object: {
      Bucket: bucketName,
      Name: objectKey
    }
  },
  MaxLabels: 20,
  MinConfidence: 75
};

const result = await rekognition.detectLabels(rekognitionParams).promise();
```

### 5. Amazon Bedrock Integration

#### Model Selection
- **Primary Model**: Claude 3 Haiku (cost-effective)
- **Fallback Model**: Titan Text Express
- **Use Case**: Product suggestion generation

#### Prompt Engineering
```javascript
const prompt = `
You are a sustainable fashion expert. Based on the following fabric analysis, suggest 5 creative products that can be made from this waste fabric.

Fabric Details:
- Type: ${fabricType}
- Colors: ${colors.join(', ')}
- Patterns: ${patterns.join(', ')}
- Size: ${estimatedSize}
- Condition: ${condition}

For each suggestion, provide:
1. Product name
2. Brief description
3. Estimated material needed
4. Difficulty level (1-5)
5. Production time estimate

Format as JSON array.
`;
```

#### Response Processing
```javascript
const bedrockParams = {
  modelId: 'anthropic.claude-3-haiku-20240307-v1:0',
  contentType: 'application/json',
  accept: 'application/json',
  body: JSON.stringify({
    anthropic_version: 'bedrock-2023-05-31',
    max_tokens: 1000,
    messages: [{ role: 'user', content: prompt }]
  })
};
```

### 6. DynamoDB Schema Design

#### Table: `FabricInventory`
```json
{
  "TableName": "FabricInventory",
  "KeySchema": [
    { "AttributeName": "factoryId", "KeyType": "HASH" },
    { "AttributeName": "fabricId", "KeyType": "RANGE" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "factoryId", "AttributeType": "S" },
    { "AttributeName": "fabricId", "AttributeType": "S" },
    { "AttributeName": "uploadDate", "AttributeType": "S" },
    { "AttributeName": "status", "AttributeType": "S" }
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "StatusIndex",
      "KeySchema": [
        { "AttributeName": "factoryId", "KeyType": "HASH" },
        { "AttributeName": "status", "KeyType": "RANGE" }
      ]
    },
    {
      "IndexName": "DateIndex",
      "KeySchema": [
        { "AttributeName": "factoryId", "KeyType": "HASH" },
        { "AttributeName": "uploadDate", "KeyType": "RANGE" }
      ]
    }
  ]
}
```

#### Sample Record Structure
```json
{
  "factoryId": "factory-123",
  "fabricId": "fabric-456",
  "uploadDate": "2024-01-15T10:30:00Z",
  "imageUrl": "s3://bucket/uploads/factory-123/2024/01/15/image.jpg",
  "status": "available", // available, used, reserved
  "rekognitionResults": {
    "labels": [
      { "name": "cotton", "confidence": 95.2 },
      { "name": "blue", "confidence": 88.7 }
    ],
    "textDetections": []
  },
  "bedrockSuggestions": [
    {
      "productName": "Denim Tote Bag",
      "description": "Sturdy tote bag perfect for shopping",
      "materialNeeded": "0.5 square meters",
      "difficulty": 3,
      "productionTime": "2 hours"
    }
  ],
  "metadata": {
    "fileSize": 2048576,
    "dimensions": { "width": 1920, "height": 1080 },
    "processingTime": 45.2
  }
}
```

#### Table: `SustainabilityMetrics`
```json
{
  "factoryId": "factory-123",
  "date": "2024-01-15",
  "metrics": {
    "fabricsProcessed": 25,
    "totalWeight": 15.5, // kg
    "productsCreated": 12,
    "co2Saved": 8.2, // kg CO2
    "waterSaved": 150 // liters
  }
}
```

### 7. Amazon QuickSight Dashboard

#### Data Source Configuration
- **Primary Source**: DynamoDB tables via QuickSight connector
- **Refresh Schedule**: Every 4 hours
- **Data Preparation**: SPICE for fast querying

#### Dashboard Sections

##### 7.1 Overview Dashboard
- Total fabrics processed (current month)
- Active fabric inventory count
- Products created this month
- Processing success rate

##### 7.2 Sustainability Impact
- CO2 emissions saved (monthly/yearly trends)
- Water usage reduction
- Waste diverted from landfills
- Comparison with previous periods

##### 7.3 Fabric Analytics
- Fabric type distribution (pie chart)
- Color analysis (bar chart)
- Processing volume trends (line chart)
- Quality grade breakdown

##### 7.4 Product Suggestions
- Most popular product categories
- Average difficulty levels
- Production time estimates
- Success rate of implemented suggestions

#### QuickSight Calculated Fields
```sql
-- CO2 Savings Calculation
ifelse(
  {fabric_weight} > 0,
  {fabric_weight} * 2.1, // 2.1 kg CO2 per kg fabric
  0
)

-- Water Savings Calculation
ifelse(
  {fabric_weight} > 0,
  {fabric_weight} * 10.85, // 10.85 liters per kg fabric
  0
)
```

## API Design

### REST API Endpoints

#### Authentication
- **Method**: JWT tokens with AWS Cognito
- **Headers**: `Authorization: Bearer <token>`

#### Fabric Management
```
GET    /api/v1/fabrics
GET    /api/v1/fabrics/{id}
PUT    /api/v1/fabrics/{id}
DELETE /api/v1/fabrics/{id}
POST   /api/v1/fabrics/{id}/suggestions/refresh
```

#### Analytics
```
GET    /api/v1/analytics/overview
GET    /api/v1/analytics/sustainability
GET    /api/v1/analytics/trends?period={month|quarter|year}
```

#### Upload Management
```
POST   /api/v1/upload/presigned-url
GET    /api/v1/upload/status/{uploadId}
```

### WebSocket API
- **Purpose**: Real-time processing updates
- **Endpoint**: `wss://api.rethread.ai/ws`
- **Events**: `processing_started`, `analysis_complete`, `suggestions_ready`

## Security Architecture

### Authentication & Authorization
- **User Authentication**: AWS Cognito User Pools
- **API Authorization**: Cognito Identity Pools + IAM roles
- **Fine-grained Access**: Custom authorizer Lambda functions

### Data Protection
- **Encryption in Transit**: TLS 1.3 for all communications
- **Encryption at Rest**: 
  - S3: AES-256 server-side encryption
  - DynamoDB: AWS managed encryption
- **API Security**: Rate limiting, input validation, CORS policies

### Network Security
- **VPC Configuration**: Lambda functions in private subnets
- **Security Groups**: Restrictive inbound/outbound rules
- **WAF**: Web Application Firewall for API Gateway

## Performance Optimization

### Caching Strategy
- **CloudFront**: Static assets and API responses (5-minute TTL)
- **DynamoDB DAX**: Microsecond latency for frequent queries
- **Lambda**: Provisioned concurrency for critical functions

### Cost Optimization
- **S3 Lifecycle**: Move old images to IA/Glacier after 90 days
- **DynamoDB**: On-demand billing for variable workloads
- **Lambda**: ARM-based Graviton2 processors for better price/performance

### Monitoring & Observability
- **CloudWatch**: Metrics, logs, and alarms
- **X-Ray**: Distributed tracing for Lambda functions
- **Custom Metrics**: Processing times, success rates, cost per analysis

## Deployment Architecture

### Infrastructure as Code
- **Tool**: AWS CDK (TypeScript)
- **Environments**: dev, staging, production
- **CI/CD**: GitHub Actions with AWS deployment

### Environment Configuration
```typescript
// CDK Stack structure
export class ReThreadAIStack extends Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    // S3 buckets
    // Lambda functions
    // DynamoDB tables
    // API Gateway
    // CloudFront distribution
    // IAM roles and policies
  }
}
```

### Deployment Pipeline
1. **Code Commit** → GitHub repository
2. **Build Stage** → Compile TypeScript, run tests
3. **Deploy Stage** → CDK deploy to target environment
4. **Integration Tests** → Automated API testing
5. **Production Deploy** → Blue/green deployment strategy

## Scalability Considerations

### Auto-scaling Configuration
- **Lambda**: Concurrent execution limit (1000)
- **DynamoDB**: Auto-scaling read/write capacity
- **API Gateway**: Throttling limits per client

### Performance Targets
- **Image Processing**: < 2 minutes per image
- **API Response Time**: < 500ms for 95th percentile
- **Dashboard Load Time**: < 3 seconds
- **Concurrent Users**: 500+ simultaneous users

## Disaster Recovery

### Backup Strategy
- **S3**: Cross-region replication for critical images
- **DynamoDB**: Point-in-time recovery enabled
- **Code**: Multi-region GitHub repository

### Recovery Procedures
- **RTO**: 4 hours (Recovery Time Objective)
- **RPO**: 1 hour (Recovery Point Objective)
- **Failover**: Automated DNS failover to backup region

This technical design provides a robust, scalable, and cost-effective solution for Re-Thread AI, leveraging AWS managed services to minimize operational overhead while maximizing performance and reliability.