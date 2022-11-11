# Sigstore the Hard Way - AWS version

Welcome the Sigstore the Hard Way tutorial, AWS version.
This tutorial is inspired by the original GCP version that can be found under the [lukehinds/sigstore-the-hard-way](https://github.com/lukehinds/sigstore-the-hard-way) repository and at https://sthw.decodebytes.sh/

The goal of this tutorial is to learn how to deploy Sigstore from scratch on AWS EC2, by setting up:

1. [Fulcio](https://github.com/sigstore/fulcio) WebPKI
2. [Rekor](https://github.com/sigstore/rekor), signature transparency log and timestamping authority
3. [Certificate Transparency Log](https://github.com/google/certificate-transparency-go/tree/master/trillian)
4. [Dex](https://github.com/dexidp/dex), OpenID Connect provider
5. [Cosign](https://github.com/sigstore/cosign), container (and more) signing and verifying tool

## Requirements

In this tutorial, we will provision AWS EC2 instances to install and configure Sigstore components.

### Copyright

This tutorial is an adaptation of the original Sigstore the Hard Way tutorial available under the [lukehinds/sigstore-the-hard-way repository](https://github.com/lukehinds/sigstore-the-hard-way) for AWS.
If you have not guessed by name, this is based off, and comes with credit to [Kelsey Hightower's Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

## Having issues, something not working?

Please raise an issue or feel free to contact me on slack or by [email](mcostant@redhat.com), this tutorial is a work in progress, feedback and contributions are greatly appreciated.
