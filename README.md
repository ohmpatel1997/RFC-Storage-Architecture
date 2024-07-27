# Storage Architecture

## Background

We have built a data storing service called as files. It is an end to end encrypted data storage service built on top of IPFS. The data in encrypted right into your browser before uploading to IPFS and only user can access their data.

This RFC focuses on designing the overall architectrure of the service which include:

- Login Flow & Recovery Flow
- Document Encryption/Decryption Model
- Document Sharing Model

## Login Flow

- User Signs up using google signin.
- User Creates a master password (more than n digits, upper/lower case letters, numbers, and special characters).
- We use Argon2id to generate a symettric key from the given user's master password. Lets call this key as `sym_key`. This `sym_key` never leaves the browser. 
- We generate the public/private key pairs and encrypt the private key using `sym_key`.


    We use user's `sym_key` to encrypt sensitive data associated with user's account; this encrypted data is called user's `encrypted_user_data`. This includes user's private keys but not his password. This encrypted information is stored by our server but can only be decrypted with user’s `sym_key`. The next time user logs in, user downloads his `encrypted_user_data` from the server, then decrypts the `encrypted_user_data` in-browser by using the `sym_key`. The `sym_key` never leaves the browser.


In this system, user’s public keys are publicly visible, while his private keys are encrypted end-to-end. His password and `sym_key` are never stored, not even as encrypted data. user’s `sym_key` and password are also never sent over any network, even as encrypted data.

### Argon2id Parameters

When using Argon2id for key derivation, it's important to choose appropriate parameters to ensure security while maintaining reasonable performance. Below are some suggestions:

Salt:
- Use a cryptographically secure random number generator to create a unique salt for each user.
- Salt length: At least 16 bytes (128 bits) is recommended. You could use 32 bytes for extra security.
- we would store the salt information along side the user information on server, as it's needed for key derivation during login.

Iteration Count:

- This parameter determines how computationally intensive the operation is.
- A typical range is 1-4 seconds on the target hardware.

Memory Size:

- Argon2id is designed to be memory-hard, so this is an important parameter.
- Recommended minimum: 64 MiB (65536 KiB), we would use that.

Parallelism Factor:

- This should typically be set to the number of available CPU cores.
- We would use 4, but it can be adjusted based on your target hardware.

Output Key Length:

- we gonna generated AES-256, so we would need a 32-byte (256-bit) key.

### Account Recovery

Currently the entire system relies on master password reliability. What if user lost/forgot his master password. In that case we need to have a recovery mechanism.


This is achieved using a recovery key similar to `sym_key`. A user can enable account recovery which generates a recovery key. The recovery key is used to encrypt the user’s private data (i.e. private keys); this encrypted user data is stored on the server, along with a hash of the recovery key.

- When a user requests account recovery, their identity is first verified through email. The server sends an email to the user containing a random six-digit passcode which they can use to prove access to their email account.
- the user then enters their email, recovery key, and a new master password. The client hashes the recovery key, which is sent to the server.
- The server checks if the hash matches the stored recovery key hash, and that the client retains the correct email code. If both match, the server sends the encrypted user data to the client, which then decrypts it with their recovery key.
- The process is same again, user enters a new password, we generate a new key using argon2id, `sym_key_2`, we encrypt the user private key using `sym_key_2` and store it on server side.

## Document Encryption/Decryption Model

Each document that user uploads is being stored in a bucket. Bucket is a logical entry into server database and each bucket has its own encryption key associated with it. All documents that are being uploaded in this bucket is encrypted using this bucket's encryption key.

### Bucket Types

There are multiple types of bucket user can create:

1. Private bucket

This is a private bucket user can create which is private to the owner only. Only owner can read and write to this kind of bucket.
   
2. Share bucket

This is a share bucket user can create through which owner can share the data inside of this bucket to other users. The owner can control the readers and writers of this bucket.

The model of the bucekt is looks something like below:


### Bucket encryption key 

Each bucket has its owner encryption key using which all documents of that bucket is encrypted and upload to IPFS. Our server does not store the bucket's encryption key directly but rather stores the encrypted version of the key.

Owner generate a random cryptographically safe string while creating the bucket and encrypt it with owner's public key. This encrypted version of the key is being stored on the server.

the basic model of the bucket is being shown below.

```
{
    "id": <uuid>,
    "type": private/share,
    "owner": {
        "id": <uuid of user>,
        "encryption_key": <bucket's encryption key encrypted with owner's public key>
    },
}
```

### Document Upload Flow

1. Client get the bucket details from the server to which user wants to upload data.
2. After getting the bucket details, client decrypts the encrypted bucket's encryption key using the user's private key.
3. Client encrypts the data using decrypted bucket's encryption key and upload the encrypted data to server.
4. Server will upload the encrypted data to IPFS network and stores the CID and filename in database.

### Document Download Flow

1. Client ask server for the file to download using the bucket id and filename
2. Server queries the database to get the cid for that file from the given bucket.
3. Server fetches the encrypted data from IPFS and send back to client.
4. Client decrypts the data using decrypted bucket's encryption key.

![Upload Storage Architecture drawio (1)](https://github.com/user-attachments/assets/f2fd3daf-d4a3-461b-998a-96f1f16a86ca)



## Document Sharing Model

We need to design our system in a way that user can securely share the files to other users and manage other users's access easily. The bucket system we have designed is quite modular and can accomodate our sharing requirement. 

We will update the bucket model as follows to accomodate our sharing requirement as well:

```
{
    "id": <uuid>,
    "type": "share",
    "owner": {
        "id": <uuid of user>,
        "encryption_key": <bucket's encryption key encrypted with owner's pulic key>
    },
    "readers": [
        {
            "id": <uuid of reader>,
            "encryption_key": <bucket's encryption key encrypted with reader's public key>
        }
    ],
    "writers": [
        {
             "id": <uuid of writer>,
            "encryption_key": <bucket's encryption key encrypted with writer's public key>
        }
    ]
}
```

As you can see we added two new attributes called `readers` and `writers`, which maintains list of additional list of users. readers would only be able to download the data from bucket and writers will be able to upload the data to the bucket as well.


#### Flow

This section describes how the sharing flow works. Lets say Alice wants to share a file to Bob.

1. Alice creates a bucket with type `share`
2. Alice upload a file that he wants to share with bob to the share bucket.
3. Client ask server for the bob's public key.
4. Client encrypt the shared bucket's encryption key with bob's public key, lets call this `bob_share_bucket_key`
5. client ask server to add bob to the reader/writers of the bucket along with the `bob_share_bucket_key`.
6. now when bob logs in, he will see the shared bucket.
7. bob download the file inside of the shared bucket, decrypt the `bob_share_bucket_key` using its private key and can view the data.

The below diagram shows the work flow of sharing. 

![Sharing flow drawio](https://github.com/user-attachments/assets/d41d54e5-cb7c-42b8-8ac9-9f10a84af27f)

