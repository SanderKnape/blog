+++
title = "Using Amazon Cognito JWT tokens to authenticate with an Amazon HTTP API"
author = "Sander Knape"
date = 2020-08-02T16:29:33+02:00
draft = false
tags = ["aws", "cognito", "http api", "jwt", "authentication"]
categories = []
+++
Last year AWS released a new iteration of their API Gateway product: [HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api.html). This new version promises lower prices, improved performance and some new features. Some features that are available in the older REST API are not (yet) available for HTTP APIs, though. The official [comparison page](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html) gives a good overview of which features are available in both products.

My favorite new feature available for HTTPs APIs is [JWT Authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html). It is now possible to have the HTTP API validate a JWT token coming from an OIDC or OAuth 2.0 provider. While this was already possible using a Lambda Authorizer, now this can be achieved in a fully managed way with only a minimum amount of work required. It's even easier now to build secure APIs with proper authentication.

In this blog post, I'll create an [Amazon Cognito User Pool](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html) with a test user and authenticate to an HTTP API using a JWT token issued by Cognito. You can find the fully working code in my [GitHub repository](https://github.com/SanderKnape/cognito-apigateway-jwt). Below I'll go through the code and explain it step by step.

## Creating the Cognito User Pool

The following CloudFormation creates a User Pool and a User Pool Client:

```yaml
UserPool:
  Type: AWS::Cognito::UserPool
  Properties:
    AutoVerifiedAttributes:
      - email
    UsernameAttributes:
      - email
    UserPoolName: cognito-apigateway

UserPoolClient:
  Type: AWS::Cognito::UserPoolClient
  Properties:
    ClientName: cognito-apigateway
    ExplicitAuthFlows:
      - ALLOW_USER_PASSWORD_AUTH
      - ALLOW_REFRESH_TOKEN_AUTH
    UserPoolId: !Ref UserPool
```

This code provisions a User Pool that accepts the user's e-mail address as the username. In addition, the [User Pool App Client](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-client-apps.html) is required as we need an entity that is allowed to make API calls to our user pool (such as signing in).

## Creating the HTTP API

The following CloudFormation sets up the HTTP API with the JWT Authorizer:

```yaml
HttpApi:
  Type: AWS::ApiGatewayV2::Api
  Properties:
    Name: cognito-apigateway
    ProtocolType: HTTP

HttpApiAuthorizer:
  Type: AWS::ApiGatewayV2::Authorizer
  Properties:
    ApiId: !Ref HttpApi
    AuthorizerType: JWT
    IdentitySource:
      - "$request.header.Authorization"
    JwtConfiguration:
      Audience:
        - !Ref UserPoolClient
      Issuer: !Sub "https://cognito-idp.${AWS::Region}.amazonaws.com/${UserPool}"
    Name: JwtAuthorizer

HttpApiIntegration:
  Type: AWS::ApiGatewayV2::Integration
  Properties:
    ApiId: !Ref HttpApi
    IntegrationMethod: GET
    IntegrationType: HTTP_PROXY
    IntegrationUri: https://www.wikipedia.org/
    PayloadFormatVersion: 1.0

HttpApiRoute:
  Type: AWS::ApiGatewayV2::Route
  Properties:
    ApiId: !Ref HttpApi
    AuthorizationType: JWT
    AuthorizerId: !Ref HttpApiAuthorizer
    RouteKey: GET /
    Target: !Sub "integrations/${HttpApiIntegration}"

HttpApiStage:
  Type: AWS::ApiGatewayV2::Stage
  Properties:
    ApiId: !Ref HttpApi
    AutoDeploy: true
    StageName: $default
```

This code sets up an HTTP API with a single GET route that forwards all requests to the Wikipedia homepage. The route is configured to use the JWT Authorizer. This authorizer expects the token to be present under the `Authorization` header, optionally prefixed with `Bearer` to conform to the formal specification. You can however use any header you want and omit the `Bearer` prefix altogether.

We also configure the authorizer to require the ID of the User Pool Client to match either the audience (`aud`) or the `client_id` entry in the token. We then set the required issuer to the URL of the User Pool; we can now be sure we only accept tokens issued by our User Pool.

## Authenticating with the HTTP API

To test the authentication, we first need to gather some details from the resources we just created.

* The ID of the User Pool. You can find this at the top of the homepage of your User Pool.
* The ID of the User Pool Client. When in the Cognito User Pool UI, click "App clients" on the left. The ID we're looking for is the `App client id`.
* The URL of the HTTP API. You can find this on the homepage of your API under "Invoke URL".

We'll test the JWT authentication using some bash scripts. Let's first set the above values as variables in addition to fake credentials for our test user:

```bash
EMAIL=fake@example.com
PASSWORD=S3cure!!

CLIENT_ID=<client_id>
POOL_ID=<pool_id>
API_URL=<api_url>
```

Next, we first properly add a user to the user pool. This is just required for this demo to test the functionality.

```bash
aws cognito-idp sign-up \
  --client-id ${CLIENT_ID} \
  --username ${EMAIL} \
  --password ${PASSWORD}

aws cognito-idp admin-confirm-sign-up \
  --user-pool-id ${POOL_ID} \
  --username ${EMAIL}
```

We now authenticate with this user and store the returned JWT token:

```bash
TOKEN=$(aws cognito-idp initiate-auth \
    --client-id ${CLIENT_ID} \
    --auth-flow USER_PASSWORD_AUTH \
    --auth-parameters USERNAME=${EMAIL},PASSWORD=${PASSWORD} \
    --query 'AuthenticationResult.AccessToken' \
    --output text)
```

Finally, we perform a `curl` request on our API using the token we just retrieved:

```bash
curl -s -D - -o /dev/null -H "Authorization: Bearer ${TOKEN}" ${API_URL}
```

This command only displays the returned headers, not the body. The header should show a 200 status code, meaning that we properly authenticated with the API. If you run this script without the token - or open the URL in your browser - you will get a 401 Unauthorized response instead. As expected! The API is only accessible with a valid, non-expired JWT token from an authenticated user.

## Conclusion

In this post I went through the steps required to authenticate to an HTTP API with a JWT token issued by AWS Cognito. Keep in mind that you can use this method with any OIDC identity provider that issues JWT token. Considering the simplicity in setting this up and the fact that no maintenance is required to keep it up and running, this is certainly an apporach to consider when building authenticated APIs.
