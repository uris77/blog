---
title: "Go Notes: Omitting empty structs"
date: 2020-01-10T12:00:16-06:00
draft: false
---

Go provides a convenient way of marshaling and unmarshaling structs by using struct field tags. This gives us the benefit of not letting the Go naming convention to leak into the JSON structure, and it also doesn't force us to use a non-idiomatic naming convention for our APIs that will be consumed outside our team.

Let's draft an example. We are modeling a census application that will capture basic information about people.

```go
type Person struct {
    Name string `json:"name"`
    Address string `json:"address"`
    DateOfBirth string `json:"dob"`
    Occupation string `json:"occupation"`
}
```

With this struct, we can encode and decode the following JSON:

```json
{
 "name": "Rob Pike",
 "address": "On a Street somewhere, in some city",
 "dob": "1970-01-01",
 "occupation": "engineer"
}
```

If we didn't know the address, then that field would be unmarshaled to the default value. In this case, it would be an empty string. Hence, this struct:

```go
Person{
    Name:        "Rob Pike",
    Address:     "",
    DateOfBirth: "197-01-01",
    Occupation:  "engineer",
}
```


would be marshaled to:

```json
{
 "name": "Rob Pike",
 "address": "",
 "dob": "1970-01-01",
 "occupation": "engineer"
}
```

There are times we don't want non-existent fields to be set to their zero values when unmarshalling them. This can be configured by using `omitempty`:

```go
type Person struct {
    Name string `json:"name"`
    Address string `json:"address,omitempty"`
    DateOfBirth string `json:"dob"`
    Occupation string `json:"occupation"`
}
```

Now this person:

```go
Person{
    Name:        "Rob Pike",
    Address:     "",
    DateOfBirth: "197-01-01",
    Occupation:  "engineer",
}
```

would be marshaled to:

```json
{
 "name": "Rob Pike",
 "dob": "1970-01-01",
 "occupation": "engineer"
}
```

We also want to record how many children a person has. We can modify `Person` like this:

```go

type Person struct {
    Name        string   `json:"name"`
    Address     string   `json:"address"`
    DateOfBirth string   `json:"dob"`
    Occupation  string   `json:"occupation"`,
    Dependent   Children `json:"dependent,omitempty"`,
}

type Children struct { Name string `json:"name, omitempty"`}
```

I know that someone can have more than one child, but I'm struggling coming up with didactic examples.

If we want to unmarshal a JSON struct with no dependent:

```json

{
 "name": "Rob Pike",
 "dob": "1970-01-01",
 "occupation": "engineer"
}
```

Our struct will try to create a zero value for `Children`, in this case, it will be a struct with empty fields:

```go
Person{
    Name:        "Rob Pike",
    Address:     "",
    DateOfBirth: "197-01-01",
    Occupation:  "engineer",
    Dependent:   Person{Name: ""},

}
```

There are instances where we would actually want this to be `nil`. For example, if we are pushing this to an elasticsearch index that has mappings that do not allow empty strings for `name`. For this use case, we have to use pointers.

```go
type Person struct {
    Name        string   `json:"name"`
    Address     string   `json:"address"`
    DateOfBirth string   `json:"dob"`
    Occupation  string   `json:"occupation"`,
    Dependent   *Children `json:"dependent"`,
}

type Children struct { Name string `json:"name, omitempty"`}
```

Our unmarshaled person with no dependent now looks like this:
```go
Person{
    Name:        "Rob Pike",
    Address:     "",
    DateOfBirth: "197-01-01",
    Occupation:  "engineer",
    Dependent:   nil,

}
```


## Summary

Use `omitempty` to avoid marshaling and unmarshaling non-existent fields. However, if a field is a struct, we should use pointers.


