# OAuth vs Lambda Authorizer

## OAuth 2.0

**What it is:** OAuth 2.0 is an industry-standard protocol for delegated authorization. It allows a user to grant a third-party application limited access to their resources on another service (like Google, Facebook, etc.) without sharing their credentials.

**Concept:** It concists on the exchange of an access token, which is a credential that represents the permission granted by the user. The token acts as a temporary, revocable key that allows applications to act on behalf of the user.

### Pros / Cons

| Aspect | Pros | Cons |
|--------|------|------|
| **Adoption** | Widely adopted, secure, and standardized method for access management | Not a ready-to-use product—requires implementation or integration with an Identity Provider (IdP) |
| **Security** | Eliminates password sharing between services | Depends on proper implementation; misconfiguration can lead to vulnerabilities |
| **Flexibility** | Works with multiple providers (Google, Facebook, Auth0, Cognito) | Requires understanding of flows (authorization code, implicit, client credentials, etc.) |



## Lambda Authorizer (formerly Custom Authorizer)

**What it is:** A Lambda authorizer is a specific AWS Lambda function that you write to control access to your Amazon API Gateway API methods. API Gateway runs this function before passing the request to your backend service.

**Concept:** The custom code within the Lambda function inspects an incoming request's token (or other parameters), performs a validation check (e.g., calling an OAuth provider's introspection endpoint or verifying a JWT signature), and returns an IAM policy document that either allows or denies access.

### Pros / Cons

| Aspect | Pros | Cons |
|--------|------|------|
| **Flexibility** | Maximum flexibility—implement any custom authorization logic or integrate with any third-party identity provider | You are responsible for writing and maintaining the authorizer code |
| **Performance** | IAM policy result can be cached by API Gateway, reducing Lambda invocations | Adds a second Lambda invocation (cost/latency) if caching is not used or cache miss occurs |
| **Architecture** | Centralizes authorization logic—main business logic Lambda functions don't worry about auth | Additional complexity in deployment and testing |
| **Integration** | Can integrate with legacy or non-standard authorization systems | Requires deeper AWS knowledge (IAM policies, API Gateway integration) |



## Differences

| Feature | OAuth | Lambda Authorizer |
|---------|-------|-------------------|
| **Type** | An open authorization framework/protocol | An AWS service component (custom code) |
| **Purpose** | Standardized way to grant delegated access | Custom logic for controlling access to API Gateway |
| **Implementation** | Requires an Identity Provider (IdP) (e.g., Cognito, Auth0) | Implemented as a custom AWS Lambda function |
| **Relationship** | Defines the authorization protocol | Can be the mechanism to validate an OAuth token |
| **Flexibility** | Standardized flow | Highly flexible/customizable logic |
| **Use Case** | Delegated authorization across services | Fine-grained access control within AWS API Gateway |



## When to Use Which?

### Use OAuth (Built-in JWT Authorizer in API Gateway HTTP APIs) when:
- Your identity provider is standard-compliant (like Amazon Cognito or Auth0)
- You only need standard JWT validation and scope checks
- You want minimal custom code and fast setup
- Your use case fits the OAuth 2.0 flows (authorization code, client credentials, etc.)

### Use Lambda Authorizer when:
- You need to integrate with a **non-standard or legacy authorization system**
- You require **complex, fine-grained access control logic** beyond simple token validation (e.g., checking database permissions, implementing custom business rules)
- You want to use **request parameters** (headers, query strings, etc.) other than just the standard `Authorization` header for authorization
- You need to **combine multiple authorization sources** (e.g., API key + JWT + database lookup)
- You need to implement **custom rate limiting or fraud detection** before allowing access



## Recommended Approach for Voting System

For a high-scale voting platform, here's how you might use both:

**Use OAuth 2.0 for:**
- User authentication (login via social providers or custom IdP)
- Standard authorization flows (authorization code flow for web apps)
- Token issuance and refresh

**Use Lambda Authorizer for:**
- Custom vote eligibility checks (e.g., "Has this user already voted in this show?")
- Rate limiting enforcement (checking Redis counters before allowing access)
- Fraud detection logic (IP reputation, device fingerprinting)
- Time-based access control (voting window validation)
- Integration with multiple auth sources (OAuth token + session cookie + device ID)

**Why this hybrid works:**
OAuth handles the standard, secure authentication flow, while the Lambda Authorizer adds the voting-specific business logic that OAuth alone can't handle. The Lambda authorizer can validate the OAuth token AND check voting eligibility in a single request, keeping your main vote ingestion service clean and focused on processing votes.

**Cost consideration:**
Lambda authorizers add ~50-100ms latency and cost per invocation. However, with caching enabled (cache TTL: 300-3600 seconds), you can reduce invocations by 90%+ for repeat requests with the same token. For a voting system with bursts of traffic, this caching is critical to keep costs manageable.


## Acronyms

| Acronym | Meaning |
|---------|---------|
| **OAuth** | Open Authorization |
| **JWT** | JSON Web Token |
| **IdP** | Identity Provider |
| **IAM** | Identity and Access Management |
| **API** | Application Programming Interface |
| **AWS** | Amazon Web Services |
| **TTL** | Time To Live (cache expiration time) |
| **ms** | Milliseconds | 

## Srouces
https://www.youtube.com/watch?v=al5I9v5Y-kA
https://stackoverflow.com/questions/78367811/aws-cognito-hosted-ui-vs-lambda-authorizer-vs-aws-cognito-custom-flow
https://youtu.be/EUkQcmvRQ6c
https://aws.amazon.com/blogs/compute/evaluating-access-control-methods-to-secure-amazon-api-gateway-apis/#:~:text=Conclusions,applications%20go%20to%20serverlessland.com.
https://aws.amazon.com/blogs/security/building-fine-grained-authorization-using-amazon-cognito-api-gateway-and-iam/
https://stackoverflow.com/questions/33702826/oauth-authorization-vs-authentication#:~:text=Comments,-Add%20a%20comment&text=To%20me%2C%20this%20is%20a,providers%22%2Dstep%20for%20authentication.
https://docs.aws.amazon.com/pt_br/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html
https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html


