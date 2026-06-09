# Roaming Home Directory Validation

This document validates that FreeIPA users can log in from multiple client systems and access the same NFS-backed home directory through autofs.

## User Home Directory Mapping

| User | Home Host | FreeIPA Home Directory |
|---|---|---|
| nwright | client01.example.test | /net/client01.example.test/home/nwright |
| kellis | client03.example.test | /net/client03.example.test/home/kellis |

## Physical Home Directory Validation

Validated that each user’s physical home directory exists on the assigned home host with the expected ownership.

### Validation commands

```bash
ansible client01.example.test -m command -a "ls -ld /home/nwright" -b
ansible client03.example.test -m command -a "ls -ld /home/kellis" -b
```

### Observed result

```text
/home/nwright exists on client01.example.test and is owned by nwright:nwright
/home/kellis exists on client03.example.test and is owned by kellis:kellis
```

## autofs Path Resolution Validation

Validated that each roaming home directory path resolves from all FreeIPA client systems through the `/net` autofs map.

### Validation commands

```bash
ansible ipaclients -m command -a "ls -ld /net/client01.example.test/home/nwright" -b
ansible ipaclients -m command -a "ls -ld /net/client03.example.test/home/kellis" -b
```

### Observed result

```text
/net/client01.example.test/home/nwright resolves from client01, client02, and client03
/net/client03.example.test/home/kellis resolves from client01, client02, and client03
```

## Cross-Client File Persistence Validation

Validated that files created in a user’s roaming home directory remain available when the same user logs in from another client.

### nwright

Created a test file from client01:

```bash
ssh nwright@client01.example.test
echo "Hello from client01 as nwright" > ~/nwright-test.txt
cat ~/nwright-test.txt
exit
```

Verified from client02:

```bash
ssh nwright@client02.example.test
pwd
ls -l ~/nwright-test.txt
cat ~/nwright-test.txt
exit
```

### Observed result

```text
pwd resolved to /net/client01.example.test/home/nwright
nwright-test.txt was available from the second client login
nwright-test.txt contained: Hello from client01 as nwright
```

### kellis

Created a test file from client03:

```bash
ssh kellis@client03.example.test
echo "Hello from client03 as kellis" > ~/kellis-test.txt
cat ~/kellis-test.txt
exit
```

Verified from client02:

```bash
ssh kellis@client02.example.test
pwd
ls -l ~/kellis-test.txt
cat ~/kellis-test.txt
exit
```

### Observed result

```text
pwd resolved to /net/client03.example.test/home/kellis
kellis-test.txt was available from the second client login
kellis-test.txt contained: Hello from client03 as kellis
```

## Result

Validation passed.

FreeIPA authentication, NFS `/home` exports, autofs `/net` path resolution, file ownership, and cross-client roaming home directory access were verified successfully.
