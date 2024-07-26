# Storage Architecture

## Background

We have built a data storing service called as files. It is an end to end encrypted data storage service built on top of IPFS. The data in encrypted right into your browser before uploading to IPFS and only user can access their data.

This RFC focuses on designing the overall architectrure of the service which include:

- Login Flow and recovery flow.
- Document Encryption/Decryption Model
- Document Sharing Model

## Login Flow

- User Signs up using google signin.
- User Creates a master password (more than n digits, upper/lower case letters, numbers, and special characters).
- We use Argon2id to generate a symettric key from the given user's master password. Lets call this key as `sym_key`. This `sym_key` never leaves the browser. 
- We generate the public/private key pairs and encrypt the private key using `sym_key`.

    We use user's `sym_key` to encrypt sensitive data associated with user's account; this encrypted data is called user's `encrypted_user_data`. This includes user's private keys but not his password. This encrypted information is stored by our server but can only be decrypted with user’s `sym_key`. The next time user logs in, user downloads his `encrypted_user_data` from the server, then de- crypts the `encrypted_user_data` in-browser by using the `sym_key`. The `sym_key` never leaves the browser.


In this system, user’s public keys are publicly visible, while his private keys are encrypted end-to-end. His password and `sym_key` are never stored, not even as encrypted data. user’s `sym_key` and password are also never sent over any network, even as encrypted data.


