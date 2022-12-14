# Sign Container

We are now ready to sign our container using our own sigstore infrastructure

But before we do that, we need to use our own TUF public key file, you might remember created this when deploying the certificate transparency server

Have this file locally and set it as an environment variable

```bash
export SIGSTORE_CT_LOG_PUBLIC_KEY_FILE="/path/to/ctfe_public.pem"
```

```bash
COSIGN_EXPERIMENTAL=1 cosign sign --oidc-issuer "https://oauth2.sigstore-aws-example.com/auth" --fulcio-url "https://fulcio.sigstore-aws-example.com" --rekor-url "https://rekor.sigstore-aws-example.com" <quay_user>/sigstore-thw:latest
```

> :notebook: `COSIGN_EXPERIMENTAL` does as it says, you're trying out an experimental feature here.

An example run:

```bash
COSIGN_EXPERIMENTAL=1 cosign sign --oidc-issuer "https://oauth2.sigstore-aws-example.com/auth" --fulcio-url "https://fulcio.sigstore-aws-example.com" --rekor-url "https://rekor.sigstore-aws-example.com" quay.io/mcostant/sigstore-thw:latest
Generating ephemeral keys...
Retrieving signed certificate...

        Note that there may be personally identifiable information associated with this signed artifact.
        This may include the email address associated with the account with which you authenticate.
        This information will be used for signing this artifact and will be stored in public transparency logs and cannot be removed later.
        By typing 'y', you attest that you grant (or have permission to grant) and agree to have this information stored permanently in transparency logs.

Are you sure you want to continue? (y/[N]): y
Your browser will now be opened to:
https://oauth2.sigstore-test-infra.click/auth/auth?access_type=online&client_id=sigstore&code_challenge=-YrhBfsskEMHE7rlOz4jeCZkqff847vWyswuaaIApXs&code_challenge_method=S256&nonce=2HH9keJQ2fYgBeTFUCxVumwno8y&redirect_uri=http%3A%2F%2Flocalhost%3A36705%2Fauth%2Fcallback&response_type=code&scope=openid+email&state=2HH9kXxXseJTyHsPUiDSnX3F3AX
**Warning** Using a non-standard public key for verifying SCT: ctfe_public.pem
Successfully verified SCT...
```

## Verifying the signing

We will now verify the signing, but before we do we need to tell cosign about our fulcio root.

Grab your `fulcio-root.pem` cerficate you generated on the fulcio server (and also copied to the certificate transparency server)

Set the following environment variable:

```bash
export SIGSTORE_ROOT_FILE="$HOME/fulcio-root.pem"
```

Download the Rekor public key:

```bash
wget -O publicKey.pem https://rekor.sigstore-aws-example.com/api/v1/log/publicKey
```

Set it in the appropriate environment variable:

```bash
export SIGSTORE_REKOR_PUBLIC_KEY="$PWD/publicKey.pem"
```

We can now verify

```bash
COSIGN_EXPERIMENTAL=1 cosign verify --rekor-url https://rekor.sigstore-aws-example.com quay.io/<quay_user>/sigstore-thw:latest
```

An example:

```bash
COSIGN_EXPERIMENTAL=1 cosign verify --rekor-url https://rekor.sigstore-aws-example.com quay.io/mcostant/sigstore-thw:latest

Verification for quay.io/mcostant/sigstore-thw:latest --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The claims were present in the transparency log
  - The signatures were integrated into the transparency log when the certificate was valid
  - Any certificates were verified against the Fulcio roots.
Certificate subject:  [mcostant@redhat.com]
{"critical":{"identity":{"docker-reference":"quay.io/mcostant/sigstore-thw"},"image":{"docker-manifest-digest":"sha256:568999d4aedd444465c442617666359ddcd4dc117b22375983d2576c3847c9ba"},"type":"cosign container image signature"},"optional":null}
```

## Congrats

Congrats on completing the tutorial!

## Having issues, not working?

Please raise an issue or feel free to contact me on slack or by [email](mcostant@redhat.com), this tutorial is a work in progress, feedback and contributions are greatly appreciated.
