---
title: "Go Notes: Auth0 validation for AWS Lambda Pt 2"
date: 2020-03-02T13:20:28-06:00
draft: true
---

We will use the `auth0` library we forked previously, to create an AWS Lambda(lambda) authorizer. This authorizer will indicate whether a request is authorized or denied. We'll also deploy the authorizer using the [serverless framework](https://serverless.com) and write a basic unit test for the authorizer.

### Requirements
- Basic understanding of what jwt is.
- Some basic Go knowledge.
- Some basic AWS Lambda knowledge.
- Some basic serverless knowledge.


### Step 1: Configure the Lambda Authorizer
The lambda authorizer is defined in the `serverless.yaml` file under the `functions` block. It is no different from any other lambda, except that we won't provide any `events`. The Auth0 library we will use makes sure that the `issuer` and the `audience` match what is in the `jwks` key. We also need to provide the url that the auth0 client will use to download the `jwks` key. We'll provide these values as environment variables:

```yaml
functions:
  authorizer:
    handler: bin/authorizer
    environment:
      AUTH_ISSUER: ${self:custom.authorizer.TOKEN_ISSUER}
      AUTH_AUDIENCE: ${self:custom.authorizer.AUDIENCE}
      JWKS_URI: ${self:custom.authorizer.JWKS_URI}

custom:
  stage: ${opt:stage, "dev"}
  authorizer: ${file(../../config/authorizer.${opt:stage,self:provider.stage}.yml)}
```

`bin/authorizer` is the eventually compiled asset that will be uploaded to AWS.

The authorizer variables are maintained in a yaml file under a `config` folder following the pattern: `authorizer.<stage>.yml`. Example content:

```yaml
TOKEN_ISSUER: https://yourapp.auth0.com/
AUDIENCE: your-audience
JWKS_URI: https://yourapp.auth0.com./.well-known/jwks.json
```

### Step 2: Initialize the Auth0 Client and logger
We'll initialize the auth0 client and the logger in the `init` function of our `main.go`. The reason for doing this is that the AWS Lambda go runtime will keep an instance of our program running after it finishes so that it can re-use them when a request hits the warm instance. This allows us to limit instantiations:

```go
package main

import (
    log "github.com/sirupsen/logrus"
    "github.com/uris77/auth0"
)

var auth0Client auth0.Auth0

func init() {
	// Instantiate an auth0 client with a Cache with the capacity for
	// 60 tokens and a ttl of 24 hours
	auth0Client = auth0.NewAuth0(60, 518400)
	log.SetFormatter(&log.JSONFormatter{})
	log.SetOutput(os.Stdout)
}

```

The logger is configured to format its output as `JSON` and to write to `Stdout`. Lambda will push all `stdout` to `Cloud Watch`. This allows us to view our logs as structured data. 

### Step 3: Gather Preconditions
The preconditions for this `authorizer` are that we need to have values for `AUTH_AUDIENCE`, `JWKS_URI` and `AUTH_ISSUER`:

```go
func Authorizer(ctx context.Context, evt events.APIGatewayCustomAuthorizerRequest) (events.APIGatewayCustomAuthorizerResponse, error) {
	if len(os.Getenv("JWKS_URI")) < 1 {
		panic("The required JWKS_URI is missing")
	}
	jwkUrl := os.Getenv("JWKS_URI")

	if len(os.Getenv("AUTH_AUDIENCE")) < 1 {
		panic("The required AUTH_AUDIENCE is missing")
	}
	aud := os.Getenv("AUTH_AUDIENCE")

	if len(os.Getenv("AUTH_ISSUER")) < 1 {
		panic("The required AUTH_ISSUER is missing")
	}
	iss := os.Getenv("AUTH_ISSUER")

        log.WithFields(log.Fields{
		"event":       evt,
		"jwkUrl":      jwkUrl,
		"aud":         aud,
		"iss":         iss,
		"auth0Client": auth0Client,
		"context":     ctx,
	}).Info("Authorizer triggered")

}
```

If all the preconditions are true, we log the initial state. This can be helpful when troubleshooting because we might want to know what were the inputs and the initial state to the `authorizer`.

### Step 4: Validate the token

The authorization token is in the `events.APIGatewayCustomAuthorizerRequest` parameter of the `authorizer`:

```go
type APIGatewayCustomAuthorizerRequest struct {
	Type               string `json:"type"`
	AuthorizationToken string `json:"authorizationToken"`
	MethodArn          string `json:"methodArn"`
}
```

The token is a string and also includes the `Bearer` string.

```go
	jwt, err := auth0Client.Validate(jwkUrl, aud, iss, evt.AuthorizationToken)

```

The auth0 client will return a `jwt` token or an error when we validate a token against the `jwkUrl` and the `audience` and `issuer`.

The authorizer needs to return an `ApiGatewayCustomAuthorizerResponse`, which will indicate to the Api Gateway if the request is authorized. The type is defined as:

```go
type APIGatewayCustomAuthorizerResponse struct {
	PrincipalID        string                           `json:"principalId"`
	PolicyDocument     APIGatewayCustomAuthorizerPolicy `json:"policyDocument"`
	Context            map[string]interface{}           `json:"context,omitempty"`
	UsageIdentifierKey string                           `json:"usageIdentifierKey,omitempty"`
}
```

The `PolicyDocument` is what we use to authorize the request. It will include an Iam Policy Statement:

```go
// APIGatewayCustomAuthorizerPolicy represents an IAM policy
type APIGatewayCustomAuthorizerPolicy struct {
	Version   string
	Statement []IAMPolicyStatement
}

// IAMPolicyStatement represents one statement from IAM policy with action, effect and resource
type IAMPolicyStatement struct {
	Action   []string
	Effect   string
	Resource []string
}
```

### Step 5: Error Handling

If validation failed, we can create a response with a policy that will deny the request:

```go
if err != nil {
		r := events.APIGatewayCustomAuthorizerResponse{
			PolicyDocument: events.APIGatewayCustomAuthorizerPolicy{
				Version: "2012-10-17",
				Statement: []events.IAMPolicyStatement{
					{
						Action:   []string{"execute-api:Invoke"},
						Effect:   "Deny",
						Resource: []string{"arn:aws:execute-api:*"},
					},
				},
			},
		}

		log.WithFields(log.Fields{
			"jwt":      jwt,
			"err":      err,
			"response": r,
		}).Error("Token Validation Failed")

		return r, nil
	}
```

It is a good idea to also log the error in case we need to do some troubleshooting.

We can also make the policy more specific and use the `arn` for the request:

```go

r := events.APIGatewayCustomAuthorizerResponse{
			PolicyDocument: events.APIGatewayCustomAuthorizerPolicy{
				Version: "2012-10-17",
				Statement: []events.IAMPolicyStatement{
					{
						Action:   []string{"execute-api:Invoke"},
						Effect:   "Deny",
						Resource: []string{evt.methodArn},
					},
				},
			},
		}

```

### Step 6: Happy Path

When validation succeeds, we only need to make two minor changes: provide a `PrincipalId` and change the `Policy Document` so that it accepts the request:

```go
resp := events.APIGatewayCustomAuthorizerResponse{
		PrincipalID: jwt.Subject(),
		PolicyDocument: events.APIGatewayCustomAuthorizerPolicy{
			Version: "2012-10-17",
			Statement: []events.IAMPolicyStatement{
				{
					Action:   []string{"execute-api:Invoke"},
					Effect:   "Allow",
					Resource: []string{evt.methodArn},
				},
			},
		},
	}

	log.WithFields(log.Fields{
		"jwt":      jwt,
		"response": resp,
	}).Info("Successfully Validated Token")

	return resp, nil


```

### Conclusion
An AWS Lambda Authorizer is the mechanism used by Api Gateway to authorize requests. It will provide an `APIGatewayCustomAuthorizerRequest` to the authorizer. This request contains both the token and the method's arn. 

We can validate the request's token with the auth0 client, and return the corresponding `IAM Policy Document`, which will indicate to Api Gateway if the request is allowed to be executed.