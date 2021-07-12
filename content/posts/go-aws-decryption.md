---
title: "Go AWS Notes: KMS - Decryption"
date: 2020-01-06T09:21:44-06:00
draft: false
---

I use KMS for serverless apps when I want to put some secrets as an environment variable (Yes, I know SSM exists). I use the serverless app and the serverless-kms-secrets (https://github.com/nordcloud/serverless-kms-secrets) for encrypting secrets and making it convenient for using them in a lambda. 

After installing the plugin, we can encrypt a secret in the command line:
```bash
sls encrypt -n MY_SECRET -v my-password
```
The plugin readme (https://github.com/nordcloud/serverless-kms-secrets) goes into more detail on how to use it.

Decrypting the secret can be rather simple with Go, but I initially found it a bit tricky when I first tried to use it. My first attempt was something like this:

```go
// Create an AWS Session
sess, _ := session.NewSession(&aws.Config{
		Region: aws.String("us-west-2"),
	})	
svc := kms.New(sess)
mypassword := os.Getenv("MY_SECRET")	
input := &kms.DecryptInput{CiphertextBlob: []byte(mypassword)}	
_, err := svc.Decrypt(input)
```

But this kept throwing an `InvalidCiphertextException` exception. The AWS documentation states that this exception occurs because "the specified ciphertext, or additional authenticated data incorporated into the ciphertext, such as the encryption context, is corrupted, missing, or otherwise invalid." The documentation also mentions that the encrypted secret is bas64 encoded. This means that I have to base64 decode the encrypted secret and use that as the value for the `CiphertextBlob` in the `DecryptInput`:


```go
encpass := os.Getenv("MY_SECRET")
password, _ := base64.StdEncoding.DecodeString(encpass)

```

The resulting output is of type `DecryptOutput` that looks something like:

```go
type DecryptOutput struct {
   _ struct{} `type:"structure"`

   // ARN of the key used to perform the decryption. This value is returned if
   // no errors are encountered during the operation.
   KeyId *string `min:"1" type:"string"`

   // Decrypted plaintext data. When you use the HTTP API or the AWS CLI, the value
   // is Base64-encoded. Otherwise, it is not encoded.
   //
   // Plaintext is automatically base64 encoded/decoded by the SDK.
   Plaintext []byte `min:"1" type:"blob" sensitive:"true"`
}

```

The clear secret is now accessible through the `Plaintext`  field.

P.S.: Remember to do the appropriate error handling.
