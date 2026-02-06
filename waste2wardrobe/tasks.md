# Implementation Plan: Waste2Wardrobe

## Overview

This implementation plan breaks down the Waste2Wardrobe system into discrete coding tasks. The system will be built using TypeScript for AWS Lambda functions, with a focus on incremental development and early validation through testing. The implementation follows a bottom-up approach: core data models → individual Lambda functions → service integration → end-to-end wiring.

## Tasks

- [ ] 1. Set up project structure and core data models
  - Create TypeScript project with AWS Lambda configuration
  - Define TypeScript interfaces for InventoryItem, GarmentAttributes, MarketAnalysis, ReuseStrategy, SustainabilityMetrics
  - Set up AWS SDK clients for S3, DynamoDB, Rekognition, and Bedrock
  - Configure environment variables for AWS resource names
  - Set up Jest testing framework with fast-check for property-based testing
  - _Requirements: All requirements (foundation)_

- [ ] 1.1 Write property test for data model completeness
  - **Property 7: Strategy Completeness**
  - **Validates: Requirements 4.1, 4.2**

- [ ] 2. Implement Upload Lambda function
  - [ ] 2.1 Create input validation logic
    - Validate required fields: category, quantity, region, timeUnsold
    - Validate image format and size (max 5MB)
    - Return specific error messages for missing fields
    - _Requirements: 1.2, 1.3_
  
  - [ ] 2.2 Write property test for input validation
    - **Property 2: Input Validation Completeness**
    - **Validates: Requirements 1.2, 1.3**
  
  - [ ] 2.3 Implement image upload to S3
    - Generate unique UUID for inventory ID
    - Upload base64-decoded image to S3 with key pattern: `inventory/{id}/product.jpg`
    - Handle S3 upload errors with retry logic
    - _Requirements: 1.1, 1.5_
  
  - [ ] 2.4 Write property test for image storage
    - **Property 1: Image Upload and Storage**
    - **Validates: Requirements 1.1, 1.4**
  
  - [ ] 2.5 Write property test for unique ID generation
    - **Property 3: Unique Identifier Assignment**
    - **Validates: Requirements 1.5**
  
  - [ ] 2.6 Create DynamoDB record with initial status
    - Create inventory item record with status "pending"
    - Store user-provided metadata
    - Store S3 key and timestamps
    - _Requirements: 1.4_
  
  - [ ] 2.7 Invoke Analysis Lambda asynchronously
    - Use AWS Lambda SDK to invoke Analysis Lambda
    - Pass inventory ID and S3 key as payload
    - Handle invocation errors
    - _Requirements: 1.4_
  
  - [ ] 2.8 Write unit tests for Upload Lambda
    - Test validation error cases
    - Test S3 upload failure handling
    - Test DynamoDB write failure handling
    - _Requirements: 1.2, 1.3, 9.3_

- [ ] 3. Checkpoint - Ensure Upload Lambda tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement Analysis Lambda - Image Analysis
  - [ ] 4.1 Create Rekognition integration
    - Retrieve image from S3 using inventory ID
    - Call Rekognition DetectLabels API with MinConfidence=70
    - Call Rekognition DetectText API for text on garments
    - Parse Rekognition response to extract garment type, colors, patterns
    - Handle low-confidence results (< 70%) as analysis failure
    - _Requirements: 2.1, 2.2_
  
  - [ ] 4.2 Write property test for attribute extraction
    - **Property 4: Attribute Extraction and Storage**
    - **Validates: Requirements 2.2, 2.4**
  
  - [ ] 4.3 Implement retry logic for Rekognition
    - Exponential backoff: 1s, 2s, 4s delays
    - Maximum 3 retry attempts
    - Update DynamoDB status to "failed" after exhausting retries
    - _Requirements: 9.1_
  
  - [ ] 4.4 Write property test for retry logic
    - **Property 14: Retry Logic with Exponential Backoff**
    - **Validates: Requirements 9.1**
  
  - [ ] 4.5 Store extracted attributes in DynamoDB
    - Update inventory item record with GarmentAttributes
    - Update status and timestamp
    - _Requirements: 2.4_

- [ ] 5. Implement Analysis Lambda - Knowledge Base Integration
  - [ ] 5.1 Create RAG Knowledge Base query logic
    - Construct query from garment attributes, category, region
    - Derive current season from timestamp
    - Query Bedrock Knowledge Base for fashion trends and market data
    - Handle Knowledge Base unavailability with cached fallback
    - _Requirements: 3.1_
  
  - [ ] 5.2 Write property test for service orchestration
    - **Property 5: Service Orchestration**
    - **Validates: Requirements 2.1, 3.1**

- [ ] 6. Implement Analysis Lambda - Market Analysis
  - [ ] 6.1 Create Bedrock prompt for market analysis
    - Construct structured prompt with product attributes and trend data
    - Include garment type, colors, patterns, region, time unsold
    - Request explanation of why product is not selling
    - _Requirements: 3.2, 3.3_
  
  - [ ] 6.2 Call Bedrock API with Claude model
    - Use Claude 3 Sonnet model
    - Configure temperature=0.7, max_tokens=2000
    - Parse response to extract explanation text
    - Implement retry logic with exponential backoff
    - _Requirements: 3.2, 9.1_
  
  - [ ] 6.3 Write property test for market analysis generation
    - **Property 6: Market Analysis Generation**
    - **Validates: Requirements 3.3, 3.4**
  
  - [ ] 6.4 Store market analysis in DynamoDB
    - Update inventory item with MarketAnalysis object
    - Store explanation and trend evaluation
    - _Requirements: 3.4_

- [ ] 7. Implement Analysis Lambda - Strategy Generation
  - [ ] 7.1 Create Bedrock prompt for reuse strategies
    - Construct prompt requesting multiple strategy types
    - Include product details, quantity, region, market analysis
    - Request structured JSON response with all required fields
    - Specify strategy types: redesign, repurpose, resale, redistribution
    - _Requirements: 4.1_
  
  - [ ] 7.2 Call Bedrock API and parse strategies
    - Parse JSON response into ReuseStrategy array
    - Validate each strategy has required fields
    - Calculate profitRecovery = expectedResaleValue - costEstimate
    - _Requirements: 4.1, 4.2_
  
  - [ ] 7.3 Implement strategy ranking logic
    - Sort strategies by profitRecovery in descending order
    - Set bestStrategy index to 0 (highest profit)
    - _Requirements: 4.3_
  
  - [ ] 7.4 Write property test for strategy ranking
    - **Property 8: Strategy Ranking**
    - **Validates: Requirements 4.3**

- [ ] 8. Implement Sustainability Calculation
  - [ ] 8.1 Create sustainability metrics calculator
    - Retrieve conversion factors from Knowledge Base
    - Calculate wasteReduction = quantity × garment_weight × reuse_percentage
    - Calculate carbonSavings = wasteReduction × carbon_per_kg
    - Calculate waterSavings = wasteReduction × water_per_kg
    - Calculate landfillReduction = wasteReduction × landfill_percentage
    - _Requirements: 5.1, 5.2_
  
  - [ ] 8.2 Write property test for sustainability calculation
    - **Property 10: Sustainability Calculation Accuracy**
    - **Validates: Requirements 5.2**
  
  - [ ] 8.3 Write property test for sustainability metrics completeness
    - **Property 9: Sustainability Metrics Completeness**
    - **Validates: Requirements 5.1, 5.3**
  
  - [ ] 8.4 Apply sustainability calculation to each strategy
    - Calculate metrics for each ReuseStrategy
    - Store metrics in strategy.sustainability field
    - _Requirements: 5.1, 5.3_
  
  - [ ] 8.5 Aggregate total sustainability impact
    - Sum all four metric types across all strategies
    - Store in totalSustainabilityImpact field
    - _Requirements: 5.4_
  
  - [ ] 8.6 Write property test for sustainability aggregation
    - **Property 11: Sustainability Aggregation**
    - **Validates: Requirements 5.4**

- [ ] 9. Finalize Analysis Lambda
  - [ ] 9.1 Update DynamoDB with complete results
    - Store all strategies with sustainability metrics
    - Store bestStrategy index
    - Store totalSustainabilityImpact
    - Update status to "completed"
    - _Requirements: 4.4, 5.3_
  
  - [ ] 9.2 Implement comprehensive error handling
    - Catch all errors in try-catch blocks
    - Update status to "failed" with error message
    - Preserve original user input data on failure
    - Log errors to CloudWatch
    - _Requirements: 9.2, 9.3, 9.4_
  
  - [ ] 9.3 Write property test for error notification
    - **Property 15: Error Notification**
    - **Validates: Requirements 9.2, 9.4**
  
  - [ ] 9.4 Write property test for data preservation on failure
    - **Property 16: Data Preservation on Failure**
    - **Validates: Requirements 9.3**

- [ ] 10. Checkpoint - Ensure Analysis Lambda tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 11. Implement Dashboard Lambda
  - [ ] 11.1 Create DynamoDB query logic
    - Query inventory item by ID
    - Check item status: pending/completed/failed
    - Return appropriate response based on status
    - _Requirements: 6.1_
  
  - [ ] 11.2 Implement user data isolation
    - Validate requesting userId matches stored userId
    - Return 403 Forbidden if user IDs don't match
    - _Requirements: 7.4_
  
  - [ ] 11.3 Write property test for user data isolation
    - **Property 13: User Data Isolation**
    - **Validates: Requirements 7.4**
  
  - [ ] 11.4 Generate S3 signed URL for image
    - Create pre-signed URL with 1-hour expiration
    - Include in dashboard response
    - _Requirements: 6.1_
  
  - [ ] 11.5 Format dashboard response
    - Include imageUrl, productInfo, detectedAttributes
    - Include whyNotSelling explanation
    - Include bestStrategy with all fields
    - Include alternativeStrategies array sorted by profit
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_
  
  - [ ] 11.6 Write property test for dashboard response completeness
    - **Property 12: Dashboard Response Completeness**
    - **Validates: Requirements 6.1, 6.2, 6.3, 6.4, 6.5**
  
  - [ ] 11.7 Write unit tests for Dashboard Lambda
    - Test 404 for non-existent inventory ID
    - Test 202 for pending analysis
    - Test 500 for failed analysis
    - _Requirements: 6.1_

- [ ] 12. Set up API Gateway
  - [ ] 12.1 Create API Gateway REST API
    - Define three endpoints: POST /inventory, GET /inventory/{id}, GET /inventory/{id}/dashboard
    - Configure CORS for web frontend
    - Set up request validation for POST endpoint
    - _Requirements: All requirements (API layer)_
  
  - [ ] 12.2 Configure API Gateway integrations
    - Integrate POST /inventory with Upload Lambda
    - Integrate GET /inventory/{id} with Dashboard Lambda
    - Integrate GET /inventory/{id}/dashboard with Dashboard Lambda
    - Configure Lambda proxy integration
    - _Requirements: All requirements (API layer)_
  
  - [ ] 12.3 Set up authentication and rate limiting
    - Configure API key authentication
    - Set rate limit: 100 requests per minute per user
    - Configure throttling and quota limits
    - _Requirements: All requirements (security)_

- [ ] 13. Create Infrastructure as Code
  - [ ] 13.1 Write AWS CDK or CloudFormation templates
    - Define S3 bucket with encryption and versioning
    - Define DynamoDB table with GSI for userId-createdAt
    - Define Lambda functions with IAM roles
    - Define API Gateway with all configurations
    - Configure CloudWatch log groups
    - _Requirements: All requirements (infrastructure)_
  
  - [ ] 13.2 Configure IAM roles and policies
    - Upload Lambda: S3 write, DynamoDB write, Lambda invoke
    - Analysis Lambda: S3 read, DynamoDB read/write, Rekognition, Bedrock
    - Dashboard Lambda: DynamoDB read, S3 read (signed URLs)
    - Apply least-privilege principle
    - _Requirements: 7.2_
  
  - [ ] 13.3 Set up CloudWatch monitoring
    - Create alarms for error rate > 5%
    - Create alarms for Lambda duration > 25s
    - Create alarms for DynamoDB throttling
    - Configure log retention: 30 days
    - _Requirements: 9.2_

- [ ] 14. Create Knowledge Base content
  - [ ] 14.1 Prepare fashion trends data
    - Create JSON documents with seasonal trends by region
    - Include popular styles, colors, patterns
    - Include regional preferences and cultural considerations
    - _Requirements: 3.1, 3.2_
  
  - [ ] 14.2 Prepare sustainability benchmarks
    - Create JSON documents with conversion factors
    - Include carbon_per_kg, water_per_kg, garment_weight
    - Include industry-standard values
    - _Requirements: 5.2_
  
  - [ ] 14.3 Set up Bedrock Knowledge Base
    - Create OpenSearch Serverless collection
    - Configure Bedrock Knowledge Base with data sources
    - Index all JSON documents
    - Test retrieval with sample queries
    - _Requirements: 3.1, 5.2_

- [ ] 15. Integration testing
  - [ ] 15.1 Write end-to-end integration tests
    - Test complete flow: upload → analysis → dashboard
    - Use real AWS services in test environment
    - Verify S3 storage and retrieval
    - Verify DynamoDB record creation and updates
    - Verify Rekognition integration
    - Verify Bedrock integration
    - _Requirements: All requirements_
  
  - [ ] 15.2 Test error scenarios
    - Test Rekognition failure and retry
    - Test Bedrock failure and retry
    - Test S3 upload failure
    - Test DynamoDB write failure
    - Verify error messages and status updates
    - _Requirements: 9.1, 9.2, 9.3, 9.4_

- [ ] 16. Final checkpoint - Ensure all tests pass
  - Run complete test suite
  - Verify all property tests pass with 100+ iterations
  - Verify all unit tests pass
  - Verify integration tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- All tasks are required for comprehensive implementation
- Each task references specific requirements for traceability
- Property tests validate universal correctness properties with 100+ iterations
- Unit tests validate specific examples and edge cases
- Integration tests verify end-to-end flows with real AWS services
- Infrastructure setup (task 13) can be done in parallel with Lambda development
- Knowledge Base content (task 14) should be prepared early to unblock Analysis Lambda testing
