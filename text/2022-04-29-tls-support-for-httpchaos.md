# TLS Support for HTTPChaos

## Summary

A solution to support TLS in HTTPChaos.

## Motivation

Currently, our HTTPChaos can only injects faults into unencrypted TCP connection.
Once the clients and servers connect to others by TLS, we can do nothing in the middle.
With the HTTPS protects more and more connections, we must figure out a solution to attack it.

## Detailed Design

### User-Provided Secrets

The simplest solutions, mainly used in the case of server-side HTTPChaos, is providing private-key(the secret) by the user.
In server-side HTTPChaos, we should inject some inbound stream of a server, so the user should be willing
to provide private-key of the certificate. In this solution, our chaos-tproxy can decrypt the client-generated
symmetric-key by the private-key.

Here is the schematic diagram:

```

STEP

               request for tls connection                     request for tls connection
1.  client  <------------------------------>  chaos-tproxy <------------------------------> server
                    cert(public-key)          (private-key)         cert(public-key)        (private-key)

             encrypted(by public-key) symmetric-key                 encrypted(by public-key) symmetric-key
2.  client -----------------------------------------> chaos-tproxy ---------------------------------------> server
                                                (decrypted symmetric-key)                           (decrypted symmetric-key)

             encrypted(by symmetric-key) request data                           modifited request data
3.  client <------------------------------------------> chaos-tproxy <--------------------------------------------> server
                    modifited response data           (decrypted data)  encrypted(by symmetric-key) response data 
```

The user can also provide different certificates from the server, so long as the certificate is trusted by the client.

For example, we can configure the secrets in HTTPChaos:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: HTTPChaos
spec:
  ...
  tls:
    secrets:
      privateKey: <the secretName of private-key>
      certificate: <the secretName of certificate>

```

### Self-Signed Certificates

Sometimes, especially in the case of client-side HTTPChaos, the user cannot provide client-trusted certificates of target hosts.
For example, there is a client of the github API in our cluster, we decided to inject faults into its request to
`https://api.github.com`, how to provide a valid certificate for our chaos-tproxy to cheat the client?

The best solution is signing a fake certificate by ourself. Generating self-signed certificate is an easy job, and the
[cert-manager](https://cert-manager.io/) can manage it for us. However, if we attck in the middle by this certificate, the client
won't trust us, because we forget to inject the CA into trust list of the client.

The [cainjector](https://cert-manager.io/docs/concepts/ca-injector) is commonly used in k8s custom webhook. To register a custom
webhook for a resouce, we must inject CA into trust list of k8s apiserver:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: webhook1
  annotations:
    cert-manager.io/inject-ca-from: example1/webhook1-certificate
webhooks:
- name: webhook1.example.com
  admissionReviewVersions:
  - v1
  clientConfig:
    service:
      name: webhook1
      namespace: example1
      path: /validate
      port: 443
  sideEffects: None
```

Unfortunately, there is no common solution to update root CA accros different operation system. In debian or ubuntu, we should
move the CA to `/usr/local/share/ca-certificates` at first, then we should run `update-ca-certificates` to update root CA;
but in REHL, we should move the CA to `/etc/pki/ca-trust/source/anchors/` and run `update-ca-trust extract`. In some virtual machine
like JVM, solutions become more complicated, we must update the "trust store" by the toolchain in JDK.

Let's deal with above problems step by step:

1. Resolve the linux distribution automatically;
2. Inject cert into root CA of system;
3. Find the JAVA_HOME and inject cert into truststore;
4. update CA for other VMs...

## More Features

Beside the enhancement of HTTPChaos, the TLS attack can bring us more features. For example, we can sign a trusted but expired
certificate to inject cert-expired fault for both clients and servers; we can also sign an untrusted certificate to simulate attack
in the middle.

## Alternatives

No idea.