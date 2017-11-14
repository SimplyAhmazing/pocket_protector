PocketProtector provides a cryptographicaly strong
serverless secret management infrastructure.  An applications
secrets are stored securely right alongside its code.

The theory of operation is that the protected.yaml file
consists of key-domains at the root level.  Each key-domain
stores data encrypted by a keypair.  The public key of the
keypair is stored in plaintext, so that anyone may encrypt
and add a new secret.  The private key is stored encrypted
by an owners public key.  The owners are known as
"key custodians", their private keys are protected by passphrases.

Secrets are broken up into domains for the purposes of
granting security differently.  For example, prod, dev, and
stage may all be different domains.  The domains may
further be split up based on application specific.

Alternatively, for a simple use case everything may
be thrown into one big domain.

To allow secrets to be accessed in a certain environment,
the passphrase for an environment-specific key custodian
should be provided.

e.g. for dev domains, just hardcode the passphrase
in settings.py or similar

for prod domains, use AWS / heroku key management to store
the passphrase

An application / script wants to get its secrets:
```python
# at initialization
secrets = KeyFile.decrypt_domain(domain_name, Creds(name, passphrase))
# ... later to access a secret
secrets[secret_name]
```

An application / script that wants to add / overwrite a secret:
```python
KeyFile.from_file(path).with_secret(
    domain_name, secret_name, value).write()
```

Note -- the secure environment key is needed to read secrets, but not write them.
Change management on secrets is intended to follow normal source-code
management.

File structure:
```yaml
[key-domain]:
  meta:
    owners:
      [name]: [encrypted-private-key]
    public_key: [b64-bytes]
    private_key: [b64-bytes]
  secret-[name]: [b64-bytes]
key-custodians:
  [name]:
    public-key: [b64-bytes]
    encrypted-private-key: [b64-bytes]
```

Threat model
------------
An attacker is presumed to be able to read but not write the contents
of protected.yaml.  This could happen because a developrs laptop
is compromised, github credentials are compromised, or (most likely)
git history is accidentally pushed to a publically acessible repo.

With read access, an attacker gets environment and secret names,
and which secrets are used in which environments.

Neither the file as a whole nore individual entries are signed,
since the security model assumes an attacker does not have
write access.

Example File
------------
Here's an example of the protected.yaml file generated by the unit
tests:

```yaml
new_domain:
  secret-hello: zBE9iug3yBYMFCBNBoFOQ/M3LQH5PnJuevmFzJO4BWvLG0JRGwqlTBcjZh2QaTjLJef+2Go=
  meta:
    public-key: DaX/DYMRb8pm6rVzA/Kf3DFmtDBUX5JJGXq57T4k20g=
    owners:
      alice@example.com: JljOSu5SHbFY0UGsMkTqVach/Y6VE9MmVMujauUK8UTwgkjSIq7rISGuOydQwSblHkq8nUToS+cRiCc2LJ19PJaTwgWcuqL+Y4tWvLL9ovo=
      bob@example.com: gUBCI9NJinch1dnnGr93RHVqmv8bJ7DdIDfFEqt5Cm68blyaWM17XhSUjtQRCkouZoiN6gPKrduFTZ6HlRgDTWqP0HSJdZPjNHkOj6wtUxQ=
key-custodians:
  alice@example.com:
    encrypted-private-key: O0XrgEuATOVvlNBRe5nsD+sSlXICDTq3pbzaj0zc338cZMTcrmlAVfzpLca1D6D4JmTyCwarx7qSrHNGP1b9bRNaDW3K7Ey4
    public-key: jPrgsotnMklLdZZv+jK0O1Jb5C2D4ywM0XlvBrPQi3Q=
  bob@example.com:
    encrypted-private-key: e8C/eUqvs8e/RJWLUMc1I3Vm4klUCZ8XVdktj0sGEE117i175oOtkAl3pYt6d4449rcqCM+jew9fq2PT8yZKJpVyKp+822Tl
    public-key: fHKKyzjDwYOQoxIv+c+t5aj9AyXuW4LJW7fq1R7XGRg=
audit-log:
- created key custodian bob@example.com
- created domain new_domain with owner bob@example.com
- created secret hello in new_domain
- created key custodian alice@example.com
- bob@example.com added owner alice@example.com to new_domain
```

Securing Write Access
---------------------
PocketProtector does not provide any security against unauthorized writes
to the protected.yaml file, by design.  Firstly, without any Public Key Infrastructure,
PocketProtector is not a good basis for cryptographic signatures.  (An attacker
that modifies the file could also replace the signing keypair with their own;
the only way to detect this would be to have a data-store outside of the file.)

Secondly -- and more importantly -- the git or mercurial repository already has
good controls around write access.  All changes are auditable, authenticated with
ssh keypairs or user passphrases.  For futher security, consider using signed commits:

https://git-scm.com/book/id/v2/Git-Tools-Signing-Your-Work
https://help.github.com/articles/signing-commits-using-gpg/
https://docs.gitlab.com/ee/user/project/repository/gpg_signed_commits/index.html
