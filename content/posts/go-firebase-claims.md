---
title: "Go Notes: Parsing Claims in Firebase JWT Token"
date: 2021-03-04T13:21:37-06:00
draft: false
---

I struggled to parse the Firebase JWT Claims. Because claims may contain strings, maps, and arrays, the claims field type is an []interface{}. In this example, one of the claims was a list of permissions. Permissions themselves were just an array of strings. Go does not know that it is an array string, and it is impossible to convert it directly. Instead, we need an intermediary conversion.

The Firebase Token struct is defined as:

```golang
// Token represents a decoded Firebase ID token.
//
// Token provides typed accessors to the common JWT fields such as Audience (aud) and Expiry (exp).
// Additionally it provides a UID field, which indicates the user ID of the account to which this token
// belongs. Any additional JWT claims can be accessed via the Claims map of Token.
type Token struct {
	AuthTime int64				   `json:"auth_time"`
	Issuer   string				   `json:"iss"`
	Audience string				   `json:"aud"`
	Expires  int64				   `json:"exp"`
	IssuedAt int64				   `json:"iat"`
	Subject  string				   `json:"sub,omitempty"`
	UID	  string				   `json:"uid,omitempty"`
	Firebase FirebaseInfo		    `json:"firebase"`
	Claims   map[string]interface{} `json:"-"`
}
```

The field we are concerned with is `Claims`. It is a map of interfaces. It will hold additional information that is added upstream when a user is created. We can assume that one of these pieces of data added is a `roles` field that consists of an array of strings. E.g.: `["role1", "role2", "role3"]`


After extracting the `claims`, and accessing the `roles` field, we need to convert the `interface` type to `[]interface{}`, because we can't convert directly from `interface` to `[]string`:

```golang
roles := claims["roles"]
var roles []string
for _, p := range claims["roles"].([]interface{}) {
   roles = append(roles, p.(string))
}
```

The takeaway for me was that `interface{}` can't be converted to `[]someType{}`. We can only convert from interface to another type if the shapes are similar.

Reference:
https://golang.org/ref/spec#Type_assertions
