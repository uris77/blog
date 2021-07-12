---
title: "Go Notes: Auth0 validation for AWS Lambda"
date: 2020-01-27T13:19:32-06:00
draft: false
---


I found myself migrating a serverless service written in node to `Go`, and needed to write a lambda authorizer. An authorizer is simply an AWS Lambda that the API Gateway will invoke to verify if the request is authorized.


The AWS Go SDK has a type for the API Gateway Request that the authorizer will receive. The authorizer I need is very simple for now. It will only concern itself with validating the JWT token. This token is represented by `AuthorizationToken` in the `APIGatewayCustomAuthorizeRequest`.

```go
type APIGatewayCustomAuthorizerRequest struct {
	Type               string `json:"type"`
	AuthorizationToken string `json:"authorizationToken"`
	MethodArn          string `json:"methodArn"`
}

```


I looked around for a library that I could leverage and found `github.com/apibillme/auth0`. However, it didn't quite suit my use case. The function signature for validating a token is:

```go
Validate(jwkURL, audience, issuer string, req *http.Request) (*jwt.Token, error)
```

The main obstacle to me using it is that the authorizer won't be receiving an `http.Request`. I needed a function where the token was a string:

```go
Validate(jwkURL, audience, issuer, jwtToken string) (*jwt.Token, error)
```

So I cloned the repo and made the changes I needed. Since the most recent activity in that repo was over a year ago, I figured that it was being actively maintained. But I still found some value in the code, so I decided to fork it and publish my own version.

The other interesting feature of this code is that it also includes a cache. The tokens are stored in the cache and for every request, the cache is inspected, and if the token is found in the cache, then we don't need to re-validate that token. So, there was one minor update I decided to make. The original code base requires that a cache be created explicitly before we can validate a token:

```go
New(128, 5)
Validate(jwkEndpoint, audience, issuer, ctx)

```

I was confused initially because I didn't know what `New` was doing. So, I created an `Auth0` type, along with a constructor:

```go
type Auth0 struct {
	cache cache.Cache
}


// Create a new Auth0 client.
// keyCapacity indicates the maximum number of keys the Cache will hold.
// ttl is the time to live in seconds for keys to live in the Cache.
func NewAuth0(keyCapacity int, ttl int64) Auth0 {
	globalTTL := time.Duration(ttl)
	Cached := cache.New(keyCapacity, cache.WithTTL(globalTTL*time.Second))
	return Auth0{cache: Cached}
}


// Validate - validate with JWK & JWT Auth0 & audience & issuer for net/http
func (a Auth0) Validate(jwkURL, audience, issuer, jwtToken string) (*jwt.Token, error) {
	// process token
	tokenParts := strings.Split(jwtToken, " ")
	token, err := verifyBearerToken(tokenParts)
	if err != nil {
		return nil, err
	}
	return a.processToken(token, jwkURL, audience, issuer)
}
```

Now using it is pretty simple:

```go
func handler(ctx Context, req APIGatewayCustomAuthorizerRequest) APIGatewayCustomAuthorizerResponse {
    //Retrieve the issuer, audience and the JWKS_URI from the environment.

    token := req.AuthorizationToken
    auth0 := NewAuth0(128, 60)
    jwtToken, err := auth0.Validate(JWKS_URI, audience, issuer, token)
    // Use the jwtToken to create the response
}

```

I'm still trying to figure out how to unit test this library. My main struggle is creating a token and a key for the tests using Go. I'll write down my notes for the next blog post, before documenting how to eventually create the `APIGatewayCustomAuthorizerResponse`