---
Title: "Integrando o API Gateway com DynamoDB"
description: "..."
tags: ["AWS", "Amazon API Gateway", "Amazon DynamoDB", "Integrations"]
series: ["API Gateway Integrations"]
series_order: 1
weight: 1
draft: false
---

## Introduction

This article explores an integration approach between **AWS API Gateway** and **Amazon DynamoDB** that eliminates the need for Lambda functions as intermediaries. Using VTL (Velocity Template Language) templates and direct integrations, we can create a complete and functional REST API with better performance and lower operational costs.

## Solution Architecture

![API Gateway - DynamoDB](img/apigateway-dynamodb-integration.svg)

The architecture implemented in this project consists of the following main components:

![Amazon API Gateway](img/Arch_Amazon-API-Gateway_48.svg)
**API Gateway REST API**: Receives HTTP requests and transforms them into DynamoDB calls

![Amazon DynamoDB](img/Arch_Amazon-DynamoDB_48.png)
**DynamoDB Table**: Stores music album data (partition key: `Artist`, sort key: `Album`)

![AWS Identity and Access Management](img/Arch_AWS-Identity-and-Access-Management_48.png)
**IAM Role**: Necessary permissions for API Gateway to access DynamoDB

![Amazon CloudWatch](img/Arch_Amazon-CloudWatch_48.svg)
**CloudWatch Logs**: Complete logging of all requests and responses

![AWS X-Ray](img/Arch_AWS-X-Ray_48.png)
**X-Ray Tracing**: Distributed tracing for monitoring and debugging

## Step-by-Step Code Analysis

### 1. CDK Stack Structure

The project uses AWS CDK (Cloud Development Kit) in TypeScript to define infrastructure as code. The main class `ApigatewayDynamodbIntegrationStack` extends `Stack` and organizes resource creation in a modular way:

```typescript
export class ApigatewayDynamodbIntegrationStack extends Stack {
  table: ITable;
  restApi: IRestApi;
  integrationErrorResponses: IntegrationResponse[];
  requestValidator: RequestValidator;
  apiGatewayRole: IRole;
  albumResource: IResource;
  ...
}
```

### 2. DynamoDB Table Creation

The first step is to create the DynamoDB table that will store the data:

```typescript
private createTable() {
  this.table = new Table(this, "dynamodb-table", {
    tableName: "apigateway-dynamodb-integration",
    partitionKey: {
      name: "Artist",
      type: AttributeType.STRING,
    },
    sortKey: {
      name: "Album",
      type: AttributeType.STRING,
    },
    removalPolicy: RemovalPolicy.DESTROY,
  });
}
```

**Important characteristics:**
- Partition key: `Artist` (String)
- Sort key: `Album` (String)
- Removal policy: `DESTROY` (for tutorial purposes - not recommended for production)

### 3. API Gateway Configuration

The `createApiGateway()` method configures all essential aspects of API Gateway:

#### 3.1. Log Group and Log Format

```typescript
const gatewayLogGroup = new LogGroup(this, "apigateway-loggroup", {
  logGroupName: "/aws/api-gateway/apigateway-dynamodb-integration",
  removalPolicy: RemovalPolicy.DESTROY,
});
```

Creates a CloudWatch Log Group to store detailed access logs.

#### 3.2. REST API with Logging and Tracing

```typescript
this.restApi = new RestApi(this, "rest-apigateway", {
  restApiName: "apigateway-dynamodb-integration",
  deployOptions: {
    tracingEnabled: true,
    loggingLevel: MethodLoggingLevel.INFO,
    accessLogDestination: new LogGroupLogDestination(gatewayLogGroup),
    accessLogFormat: AccessLogFormat.jsonWithStandardFields({
      caller: true,
      httpMethod: true,
      ip: true,
      protocol: true,
      requestTime: true,
      resourcePath: true,
      responseLength: true,
      status: true,
      user: true,
    }),
  },
});
```

**Enabled features:**
- **Tracing**: X-Ray enabled for distributed tracing
- **Logging**: INFO level to capture request details
- **Access Logs**: Structured JSON format with standard fields

#### 3.3. IAM Role for API Gateway

```typescript
this.apiGatewayRole = new Role(this, "apigateway-role", {
  assumedBy: new ServicePrincipal("apigateway.amazonaws.com"),
});
this.table.grantFullAccess(this.apiGatewayRole);
(this.apiGatewayRole as Role).addToPolicy(
  new PolicyStatement({
    actions: [
      "xray:PutTraceSegments",
      "xray:PutTelemetryRecords"
    ],
    resources: ["*"]
  }),
);
```

The role allows API Gateway to:
- Execute full operations on the DynamoDB table
- Send telemetry data to X-Ray

#### 3.4. Request Validator

```typescript
this.requestValidator = new RequestValidator(this, "request-validator", {
  requestValidatorName: "apigatewa-dynamodb-validator",
  restApi: this.restApi,
  validateRequestBody: true,
  validateRequestParameters: true,
});
```

Automatically validates request bodies and path parameters before processing them.

#### 3.5. Common Error Responses

```typescript
this.integrationErrorResponses = [
  {
    statusCode: "400",
    selectionPattern: "400",
    responseTemplates: {
      'application/json': `{
        "error": "Bad input!"
        }`
    }
  },
  {
    statusCode: "500",
    selectionPattern: "5\\d{2}",
    responseTemplates: {
      'application/json': `{
          "error": "Internal Service Error!"
          }`
    }
  }
];
```

Defines reusable response templates for 400 and 500 errors.

### 4. POST /album Endpoint - Create Album

This endpoint allows adding a new album to the table.

#### 4.1. Validation Schema

```typescript
const postRequestSchema: JsonSchema = {
  title: "PostAlbumSchema",
  type: JsonSchemaType.OBJECT,
  schema: JsonSchemaVersion.DRAFT4,
  properties: {
    artist: { type: JsonSchemaType.STRING },
    album: { type: JsonSchemaType.STRING },
    tracks: {
      type: JsonSchemaType.ARRAY,
      items: {
        type: JsonSchemaType.OBJECT,
        properties: {
          title: { type: JsonSchemaType.STRING },
          lenght: { type: JsonSchemaType.STRING }
        }
      }
    }
  },
  required: ["artist", "album"],
};
```

Defines the expected format of the input JSON, validating:
- `artist` and `album` are required
- `tracks` is an optional array of objects with `title` and `length`

#### 4.2. VTL Request Template

The VTL template transforms the HTTP request JSON into a DynamoDB `PutItem` command:

```vtl
{
  "TableName": "${this.table.tableName}",
  "Item": {
    "Artist": { "S": "$input.path('$.artist')" },
    "Album": { "S": "$input.path('$.album')" }
    #if($input.path('$.tracks') && $input.path('$.tracks').size() > 0)
    ,"Tracks": {
      "L": [
        #foreach($track in $input.path('$.tracks'))
        {
          "M": {
            "Title": { "S": "$track.title" },
            "Length": { "S": "$track.length" }
          }
        }#if($foreach.hasNext),#end
        #end
      ]
    }
    #end
  }
}
```

**Template explanation:**
- `$input.path('$.artist')` extracts the `artist` field from JSON
- `{ "S": "..." }` defines the value as String in DynamoDB format
- The `#if` directive checks if `tracks` exists and is not empty
- The `#foreach` loop iterates over each track and creates the list (`L`) and map (`M`) structure for DynamoDB

#### 4.3. Response Template

```typescript
{
  statusCode: '204',
  responseTemplates: {
    'application/json': '$context.requestId',
  }
}
```

Returns status 204 (No Content) with the request ID as response.

### 5. GET / Endpoint - List All Albums

This endpoint performs a `Scan` on the table and returns all albums.

#### 5.1. Request Template

```json
{ "TableName": "${this.table.tableName}" }
```

Simply specifies the table name for the `Scan` command.

#### 5.2. VTL Response Template

The most complex template transforms the DynamoDB response (native format) into readable JSON:

```vtl
[
#foreach($item in $input.path('$.Items'))
{
  "artist": "$item.Artist.S",
  "album": "$item.Album.S",
  #if($item.Tracks && $item.Tracks.L)
  "tracks": [
    #foreach($track in $item.Tracks.L)
    {
      "title": "$track.M.Title.S",
      "length": "$track.M.Length.S"
    }#if($foreach.hasNext),#end
    #end
  ]
  #else
  "tracks": []
  #end
}#if($foreach.hasNext),#end
#end
]
```

**Explanation:**
- `$input.path('$.Items')` accesses the `Items` array returned by DynamoDB
- `$item.Artist.S` extracts the String value of the `Artist` attribute
- The nested loop processes each track within `Tracks.L`
- `$track.M.Title.S` accesses the `Title` attribute within the track map
- The `#if($foreach.hasNext)` logic adds commas between elements

### 6. DELETE /\{artist\}/\{album\} Endpoint - Delete Album

Removes a specific album using its composite key.

#### 6.1. Request Template

```vtl
{
  "TableName": "${this.table.tableName}",
  "Key": {
    "Artist": { "S": "$util.urlDecode($method.request.path.artist)" },
    "Album": { "S": "$util.urlDecode($method.request.path.album)" }
  },
  "ReturnValues": "ALL_OLD"
}
```

**Characteristics:**
- `$method.request.path.artist` and `$method.request.path.album` extract path parameters
- `$util.urlDecode()` decodes special characters (such as spaces)
- `ReturnValues: "ALL_OLD"` returns the deleted item for verification

#### 6.2. Response Template with 404 Error Handling

```vtl
#set($artist = $input.path('$.Attributes.Artist.S'))
#if($artist && "$artist" != "")
{
  "artist": "$input.path('$.Attributes.Artist.S')",
  "album": "$input.path('$.Attributes.Album.S')"
}
#else
#set($context.responseOverride.status = 404)
{
  "error": "Record does not exist in database",
  "message": "The artist '$util.urlDecode($method.request.path.artist)' and album '$util.urlDecode($method.request.path.album)' were not found in the table"
}
#end
```

**Business logic:**
- If the item existed (`Attributes` present), returns status 200 with data
- If it didn't exist (`Attributes` empty), overrides status to 404 and returns error message

## API Endpoints

### POST /album
**Purpose:** Add a new album

**Request Body:**
```json
{
  "artist": "Dream Theater",
  "album": "Images and Words",
  "tracks": [
    {
      "title": "Pull Me Under",
      "length": "8:14"
    }
  ]
}
```

**Responses:**
- `204 No Content` - Success
- `400 Bad Request` - Validation failed
- `500 Internal Server Error` - Internal error

### GET /
**Purpose:** List all albums

**Response:**
```json
[
  {
    "artist": "Dream Theater",
    "album": "Images and Words",
    "tracks": [
      {
        "title": "Pull Me Under",
        "length": "8:14"
      }
    ]
  }
]
```

### DELETE /\{artist\}/\{album\}
**Purpose:** Delete a specific album

**Path Parameters:**
- `artist` - Artist name (URL encoded)
- `album` - Album name (URL encoded)

**Responses:**
- `200 OK` - Album deleted successfully
- `404 Not Found` - Album not found
- `400 Bad Request` - Invalid parameters
- `500 Internal Server Error` - Internal error

## Advantages of Direct API Gateway ↔ DynamoDB Integration

### 1. **Latency Reduction**
- **No intermediary layer**: Eliminates the overhead of invoking Lambda functions
- **Reduced latency**: Requests are processed directly by API Gateway
- **Response time**: Typically 20-50ms faster than Lambda-based architectures

### 2. **Cost Reduction**
- **No Lambda charges**: No function invocations, significantly reducing costs
- **API Gateway cost only**: You only pay for API Gateway requests
- **Fewer provisioned resources**: Less infrastructure to manage and pay for

### 3. **Architectural Simplicity**
- **Fewer components**: Eliminates the need to create, manage, and version Lambda functions
- **Centralized configuration**: All integration logic stays in API Gateway
- **Simplified maintenance**: Less code to maintain and update

### 4. **Better Scalability**
- **Native auto-scaling**: API Gateway and DynamoDB scale automatically
- **No concurrency bottlenecks**: No Lambda concurrency limits
- **High load support**: Can handle thousands of requests per second without additional configuration

### 5. **Powerful VTL Templates**
- **Complex transformations**: VTL allows sophisticated data transformations
- **Conditional logic**: Support for loops, conditions, and data manipulation
- **Custom formatting**: Easily converts between JSON and DynamoDB formats

### 6. **Integrated Security**
- **IAM Roles**: Granular access control through IAM
- **Input validation**: Schema validation integrated in API Gateway
- **No exposed custom code**: Transformation logic stays on the infrastructure side

### 7. **Complete Observability**
- **CloudWatch Logs**: Automatic logging of all requests
- **X-Ray Tracing**: End-to-end distributed tracing
- **Native metrics**: Integrated monitoring in API Gateway

### 8. **Request Validation**
- **Pre-processing validation**: Invalid requests are rejected before reaching DynamoDB
- **JSON Schemas**: Typed and declarative validation
- **Clear error messages**: Immediate feedback to the client

### 9. **Versioning and Change Control**
- **Infrastructure as code**: CDK allows versioning and change review
- **Controlled deployment**: Changes can be tested and reviewed before deployment
- **Facilitated rollback**: Easy to revert to previous versions

### 10. **Optimized Performance**
- **No cold starts**: No runtime initialization as in Lambda
- **Persistent connections**: API Gateway maintains optimized connections with DynamoDB
- **Template caching**: VTL templates are compiled and cached

## When to Use This Approach

### Ideal Use Cases:
- ✅ Simple CRUD APIs (Create, Read, Update, Delete)
- ✅ Direct data transformations without complex business logic
- ✅ Applications that need low latency
- ✅ High volume request scenarios
- ✅ When cost is a primary concern

### Limitations to Consider:
- ⚠️ Complex business logic can be difficult to implement in VTL
- ⚠️ Debugging VTL templates can be challenging
- ⚠️ No support for complex transactional operations (DynamoDB Transactions requires Lambda)
- ⚠️ Asynchronous processing is not directly supported

## Conclusion

Direct integration between API Gateway and DynamoDB offers an elegant and efficient solution for simple REST APIs. By eliminating the Lambda layer, we reduce latency, costs, and complexity while maintaining all essential validation, logging, and security functionalities.

This approach is especially valuable for:
- Simple microservices
- Backend APIs for frontend
- Rapid prototyping
- Serverless applications optimized for performance and cost

Using VTL templates may seem complex initially, but it offers sufficient flexibility for most common use cases, making this architecture an excellent choice for many modern AWS projects.

## GitHub Repository

{{< github repo="marciocadev/apigateway-dynamodb-integration" showThumbnail=true >}}
<!-- 
**GitHub Repository:** [https://github.com/marciocadev/apigateway-dynamodb-integration](https://github.com/marciocadev/apigateway-dynamodb-integration) -->

{{< keywordList >}}
{{< keyword icon="aws" >}} Amazon API Gateway {{< /keyword >}}
{{< keyword icon="aws" >}} Amazon DynamoDB {{< /keyword >}}
{{< keyword icon="aws" >}} Amazon CloudWatch {{< /keyword >}}
{{< keyword icon="aws" >}} AWS Identity and Access Management {{< /keyword >}}
{{< keyword icon="aws" >}} AWS X-Ray {{< /keyword >}}
{{< /keywordList >}}