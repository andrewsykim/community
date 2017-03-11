# Encrypt Secrets in etcd

## Abstract

The scope of this proposal is to ensure that secrets are encrypted at the datastore layer. Encrypting data at REST and via third party vendors is a desired feature but outside the scope of this proposal. Encryption will be optional for any resource but we suspect it will be used for the Secret resource in most cases.

Allowing sensitive data to be encrypted adheres to best practices as well as other requirements such as HIPAA and many others.

How encryption keys are delivered to the machine running the Kubernetes apiserver is of relevance - we assume that the encryption at rest pattern is loosely coupled to how those keys are delivered and secured on disk.

In general, full disk encryption of the volumes storing etcd data is generally preferred - this proposal focuses on scenarios where additional protection is desired against malicious parties gaining read access to the etcd API or a running etcd instance without access to memory of the etcd process.

## High level design
Before a secret is written to etcd and before the secret is read, an encryption provider will take the plaintext data and encrypt it.  These providers will be able to be created and turned on depending on the users needs or requirements and will adhere to an encryption interface. This interface will provide the abstraction to allow various encryption mechanisms to be implemented, as well as for the method of encryption to be rotated over time.  It should be possible for the distribution of keys to the apiserver to be separated out (injected in by a higher level security process) or to be directly requested by the provider implementation. For the first iteration, a default provider will be developed and will run as part of the kube-apiserver.

## Kubernetes Storage Changes
Kubernetes requires that an update that does not change the serialized form of object not be persisted to etcd to prevent other components from seeing no-op updates.  This must be done within the Kubernetes storage interfaces - we will introduce a new API to the Kube storage layer that transforms the serialized object into the desired at-rest form and provides hints as to whether no-op updates should still persist (when key rotation is in effect).
```go
// ValueTransformer allows a string value to be transformed before being read from or written to the underlying store. The methods
// must be able to undo the transformation caused by the other.
type ValueTransformer interface {
	// TransformFromStorage may transform the provided data from its underlying storage representation or return an error.
	// Stale is true if the object on disk is stale and a write to etcd should be issued, even if the contents of the object
	// have not changed.
	TransformFromStorage([]byte) (data []byte, stale bool, err error)
	// TransformToStorage may transform the provided data into the appropriate form in storage or return an error.
	TransformToStorage([]byte) (data []byte, err error)
}
``` 

When the storage layer of Kubernetes is initialized for secrets, an implementation of this interface that manages encryption will be passed down.  Other resources can use a no-op provider by default.

## Encryption Provider
An encryption provider implements the ValueTransformer interface. Out of the box this proposal will implement encryption using a standard AES-GCM performing AEAD, using the standard Go library for AES-GCM, with the key configuration provided at process startup.

Other encryption or key distribution implementations are possible such as AWS KMS, Google Cloud KMS and Hashicorp Vault just for example depending on the environment the cluster is deployed into as well as its capabilities. 

The encryption provider implementation should work the same where a set of flags will be added to the API server specifying which (if any) provider is requested, which keys should be used for decryption, and which key is the primary encryption key. Each encryption provider will have a unique string identifier to ensure versioning of contents on disk and to allow future schemes to be replaced.

It should be possible to separate the mechanism by which we encrypt the data in etcd from the mechanism whereby the keys to perform that mechanism are loaded onto the appropriate master.  The provider MAY perform dynamic retrieval of keys from a hardware security module, retrieve a set of keys from a cloud-native security provider, or prompt the user for a password to decrypt the encryption keys from rest on disk, or some combination of the above.

During encryption, only a single provider is required.  During decryption, multiple providers or keys may be in use (when migrating from an older version of a provider, or when rotating keys), and thus the ValueTransformer implementation must be able to delegate to the appropriate provider.

For the near term, we identify a single encryption provider out of the box:

### AES-GCM provider
The simplest possible provider is an AES-GCM encrypter/decrypter using AEAD, where we create a unique nonce on each new write to etcd, use that as the IV for AES-GCM of the value (the JSON or protobuf data) along with a set of authenticated data to create the ciphertext, and then on decryption use the nonce and the authenticated data to decode.  The keys come from configuration on the local disk (potentially decrypted at startup time using a stronger password protected key, or in the future from an alternative source).

The provider will be assigned a versioned identifier to uniquely pair the implementation with the data at rest, such as “k8s-aes-gcm-v1”.  Any implementation that attempts to decode data associated with this provider id must follow a known structure and apply a specific algorithm.

The provider would take a set of keys and unique key identifiers from the command line, with the key values stored on disk.  One key is identified as the write key, all others are used to decrypt data from previous keys.  Keys must be rotated more often than every 2^32 writes

The provider should use the recommended Go defaults for all crypto settings unless otherwise noted.  We should use AES-256 keys (32 bytes).

Process for transforming a value (object encoded as JSON or protobuf) to and from stable storage will look like the following:

Layout as written to etcd2 (json safe string only):
``` 
NONCE := read(/dev/urandom)
PLAIN_TEXT := <VALUE>
AUTHENTICATED_DATA := ETCD_KEY
CIPHER_TEXT := aes_gcm_encrypt(KEY, IV:NONCE, PLAIN_TEXT, A:AUTHENTICATED_DATA)
BASE64_DATA := base64(<NONCE><CIPHER_TEXT>)
STORED_DATA := <PROVIDER>:<KEY_ID>:<BASE64_DATA>
```

Layout as written to etcd3 (bytes):
```
NONCE := read(/dev/urandom)
PLAIN_TEXT := <VALUE>
AUTHENTICATED_DATA := ETCD_KEY
CIPHER_TEXT := aes_gcm_encrypt(KEY, IV:NONCE, PLAIN_TEXT, A:AUTHENTICATED_DATA)
STORED_DATA := <PROVIDER_ID>:<KEY_ID>:<NONCE><AUTHENTICATED_DATA><CIPHER_TEXT>
```

Pseudo-code for encrypt (golang):
```go
block := aes.NewCipher(primaryKeyString)
aead := cipher.NewGCM(c.block)
keyId := primaryKeyId

// for each write
nonce := make([]byte, block.BlockSize())
io.ReadFull(crypto_rand.Reader, nonce)
authenticatedData := ETCD_KEY
cipherText := aead.Seal(nil, nonce, value, authenticatedData)
storedData := providerId + keyId + base64.Encode(nonce + authenticatedData + cipherText)
```

Pseudo-code for decrypt (golang):
```go
// for each read
providerId, keyId, base64Encoded := // slice provider and key from value
// ensure this provider is the one handling providerId

aead := // lookup an aead instance for keyId or error
bytes := base64.Decode(base64Encoded)
nonce, authenticatedData, cipherText := // slice from bytes
out, err := aead.Open(nil, nonce, cipherText, authenticatedData)
```

### Other encryption providers

It should be easy to introduce new variations on the basic provider - such as those that require a password via STDIN on startup to decrypt the keys on disk, or wish to exchange a temporary token on disk for the actual keys from a third party security server.

We expect that the provider implementation may therefore be layered or composed and that as much as possible providers should be separated behind interfaces.

### Potential providers

* A local HSM implementation that retrieves the keys from the secure enclave prior to reusing the AES-GCM implementation (initialization of keys only)
* Exchanging a local temporary token for the actual decryption tokens from a networked secret vault
* Decrypting the AES-256 keys from disk using asymmetric encryption combined with a user input password


### Initializing encryption providers

Like other plugin points to Kubernetes, initialization of the providers should happen close to “cmd/kube-apiserver/app”, where it is possible to translate configuration from command line parameters or config files into mostly orthogonal implementations.  We will follow standard Kubernetes conventions for this initialization to ensure that as much as possible the providers have clean interfaces for composition and testing.

Suggested initial config keys for the AES-GCM provider:
```
--encryption-primary-key=PROVIDER_ID:KEY_ID:PATH_TO_KEY (single)
--decryption-key=PROVIDER_ID:KEY_ID:PATH_TO_KEY (optional, can have many)
--encryption-resources=GROUP_RESOURCE (e.g. v1/Secrets, at least 1 required)
```

## Additional Considerations

### Introducing encryption for existing clusters

Encrypting existing secrets should be as simple as dispatching a PUT once the kube-apiserver is running all the required flags. Since PUT operations are idempotent, it should be a no-op for secrets that are already encrypted unless the encryption key/initialization vectors are modified.

### Changing the Encryption Key 

Dispatching a PUT on a resource that is already encrypted should only change the data in etcd if the encryption algorithm yields a different result (if the key has been changed, the provider has been changed, or the object has been changed). This should only be the case when the encryption key/initialization vectors are modified or when a new encryption strategy is introduced. Changing the encryption key should be as simple as:

1. Introduce a new decryption key (the new key)
2. Restart all apiservers - at this point all servers can decrypt the new key from etcd
3. Make the new key the primary key
4. Restart all apiservers - at this point all servers start writing with the new key
5. Dispatch PUTs on all secrets (decrypts with secondary key, encrypts with primary)
6. Remove secondary key
7. Restart apiservers

NOTE: Steps 1-4 can be collapsed if there is a single API server, or the operator is willing to have the API servers down for some period of time.  

### Performance

Introducing even a relatively well tuned AES-GCM implementation is likely to have performance implications for Kubernetes.  Fortunately, existing optimizations occur above the storage layer and so the highest penalty will be incurred on writes when secrets are created or updated.  In multi-tenant Kube clusters secrets tend to have the highest load factor (there are 20-40 resources types per namespace, but most resources only have 1 instance where secrets might have 3-9 instances across 10k namespaces).  Writes are uncommon, creates usually happen only when a namespace is created, and reads are somewhat common. 

### Actionable Items / Milestones

* [p0] Add ValueTransformer to storage [p0]
* [p0] Create a default implementation of AES-GCM interface [p0]
* [p0] Add encryption flags on kube-apiserver [p0]
* [p1] Audit of default implementation for safety and security [p1]
* [p2] E2E and performance testing [p2]
* [p2] Documentation and users guide [p2]
* [p2] Read cache layer if encrypting/decrypting Secrets adds too much load on kube-apiserver [p2]

