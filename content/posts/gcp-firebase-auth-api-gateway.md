---
title: Firebase Authentication with API Gateway
date: 2023-03-04T12:00:00-06:00
draft: false
---

Google Cloud's API Gateway can be configured to handle authentication if our application is using Firebase Auth. Our Application receives a header `X-Apigateway-Api-Userinfo` that contains the user's token. This is a convenient way to protect the endpoints.

## Add Security Definition

Define a `firebase` security definition. The `x-google-audiences` value should be the project id:

```yaml
securityDefnitions:
	firebase:
		authorization: https://accounts.google.com/o/auth2/auth
		flow: implicit
		type: oauth2
		x-google-issuer: https://securetoken.google.com/moh-htps-staging
		x-google-jwks_uri: https://www.googleapis.com/service_accounts/v1/metadata/x509/securetoken@system.gserviceaccount.com
		x-google-audiences: PROJECT_ID

```

## Secure Endpoints

The endpoints can be configured to require Firebase Authentication by specifying the security scheme:

```yaml
paths:
  /a-path:
	  get:
		  summary: An Endpoint
		  security:
		  - firebase: []
		  responses:
			  200:
				  description: Success
```

## CORS

Setting up CORS when we are using Firebase Authentication is not so straightforward, and the GCP documentation doesn't mention how to do this. Setting up CORS for non-Firebase Auth can be as simple as specifying it in the `x-google-endpoints`:

```yaml
swagger: 2.0
schems:
  - https
x-google-endpoints:
  - name: project-id-gw-randomvalue.apigateway.project-id.cloud.goog
    allowCors: True
```

The name of the endpoint should be the name gateway URL for the API Gateway we are using.

However, this alone won't work with Firebase. We have to also specify the `options` request for every path. This is a bit inconvenient, but it is the only way I got Firebase Auth to work:

```yaml
paths:
  /a-path:
	  options:
		  responses:
			  200:
				  description: Success
	  get:
		  summary: An Endpoint
		  security:
		  - firebase: []
		  responses:
			  200:
				  description: Success
```

## Securing the Firebase API Key

When you create a Firebase Application, an API Key will be automatically generated. This is used by the application to communicate with Firebase. You can see this key by going to `APIs & Services` > `Credentials`. The key will be named `Browser key (auto created by Firebase)` This key has no restrictions by default. We can secure it by adding a `Websites` restriction. We'll need to add `https://project-id.firebaseapp.com` . If we have other domains we will use, we can add them there.

The next step is to add a `Referer` header to every Firebase API call we make. The `Referer` should be the value of one of the domains in this list of `Websites`.

### References

[API Gateway & Terraform](https://blog.openstep.net/posts/gcp-api-terraform/)
