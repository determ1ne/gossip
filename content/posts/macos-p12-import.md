+++
date = '2025-07-11T10:07:09+08:00'
draft = false
title = 'macOS p12 file import "MAC verification failed"'
tags = [
  'macOS',
  'pki',
]
+++

When importing p12 file to macOS, the password is always incorrect.
After 3 tries, keychain app fails with "MAC verification failed during PKCS12 import (wrong password?)".

See [stackoverflow#70431528](https://stackoverflow.com/questions/70431528/mac-verification-failed-during-pkcs12-import-wrong-password-azure-devops).

OpenSSL 3.x's default algorithm is not supported on macOS.
Use `-legacy` option to export the p12 file.

```bash
openssl pkcs12 -export -legacy -out client.p12 -in client.crt -inkey client.key
```

OpenSSL 3.x's p12 info:

```bash
$ openssl pkcs12 -info -in client.p12
Enter Import Password:
MAC: sha256, Iteration 2048
MAC length: 32, salt length: 8
PKCS7 Encrypted data: PBES2, PBKDF2, AES-256-CBC, Iteration 2048, PRF hmacWithSHA256
```

OpenSSL 3.x (with legacy option)'s p12 info:

```bash
$ # add -legacy for info
$ openssl pkcs12 -info -in client.p12
Enter Import Password:
MAC: sha1, Iteration 2048
MAC length: 20, salt length: 8
PKCS7 Encrypted data: pbeWithSHA1And40BitRC2-CBC, Iteration 2048
Error outputting keys and certificates
00B37244F87F0000:error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:crypto/evp/evp_fetch.c:375:Global default library context, Algorithm (RC2-40-CBC : 0), Properties ()
```
