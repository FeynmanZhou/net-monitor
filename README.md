# Sign and verify an image with Notaion and Ratify

Log in to ghcr.io.

```
echo $CR_PAT | docker login ghcr.io -u pengfeizhou --password-stdin
```

Build an image from the source code.

```
docker build -t localhost:5001/net-monitor:v1 https://github.com/wabbit-networks/net-monitor.git#main
```

Tag the image.

```
docker tag localhost:5001/net-monitor:v1 ghcr.io/feynmanzhou/net-monitor:v1
```

Push the image to ghcr.io.

```
docker push ghcr.io/feynmanzhou/net-monitor:v1
```

Generate x.509 key and cert for signing and verification.

```
notation cert generate-test --default "gopher-china.io"
```

Sign the sample image in ghcr.io.

```
notation sign ghcr.io/feynmanzhou/net-monitor@sha256:27c0290c485140c3c998e92c6ef23fba2bd9f09c8a1c7adb24a1d2d274ce3e8e
```


List the signature.
```
notation ls ghcr.io/feynmanzhou/net-monitor@sha256:27c0290c485140c3c998e92c6ef23fba2bd9f09c8a1c7adb24a1d2d274ce3e8e
```

Create trust policy.

```
cat <<EOF > ./trustpolicy.json
{
    "version": "1.0",
    "trustPolicies": [
        {
            "name": "gopher-china-images",
            "registryScopes": [ "*" ],
            "signatureVerification": {
                "level" : "strict"
            },
            "trustStores": [ "ca:gopher-china.io" ],
            "trustedIdentities": [
                "*"
            ]
        }
    ]
}
EOF
```

Import Trust policy to trust store.

```
notation policy import ./trustpolicy.json
```

Verify the signed image.

```
notation verify ghcr.io/feynmanzhou/net-monitor@sha256:27c0290c485140c3c998e92c6ef23fba2bd9f09c8a1c7adb24a1d2d274ce3e8e
```

Install Gatekeeper and Ratify.

```
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts

helm install gatekeeper/gatekeeper  \
    --name-template=gatekeeper \
    --namespace gatekeeper-system --create-namespace \
    --set enableExternalData=true \
    --set validatingWebhookTimeoutSeconds=5 \
    --set mutatingWebhookTimeoutSeconds=2
```

```
helm repo add ratify https://deislabs.github.io/ratify

helm install ratify \
    ratify/ratify --atomic \
    --namespace gatekeeper-system \
    --set-file notaryCert=ghcr-networks.io.crt
```

Apply the constrait:

```
kubectl apply -f https://deislabs.github.io/ratify/library/default/template.yaml
kubectl apply -f https://deislabs.github.io/ratify/library/default/samples/constraint.yaml
```

Run an image signed by Notation.

```
$ kubectl run gopher-demo --image=ghcr.io/feynmanzhou/net-monitor@sha256:27c0290c485140c3c998e92c6ef23fba2bd9f09c8a1c7adb24a1d2d274ce3e8e
/net-monitor@sha256:27c0290c485140c3c998e92c6ef23fba2bd9f09c8a1c7adb24a1d2d274ce3e8e
pod/demo created
```

Run an unsigned image.

```
notation ls ghcr.io/feynmanzhou/notation/alpine@sha256:1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870
```

```
$ kubectl run demo --image=ghcr.io/feynmanzhou/notation/alpine@sha256:1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: [ratify-constraint] Subject failed verification: ghcr.io/feynmanzhou/notation/alpine@sha256:1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870
```

```
 {
      "subject": "ghcr.io/feynmanzhou/notation/alpine@sha256:1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870",
      "isSuccess": false,
      "message": "verification failed: no referrers found for this artifact"
    }
···
```

Run a signed image with an expired certificate

```
kubectl run sample --image=ghcr.io/feynmanzhou/alpine:latest@sha256:1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: [ratify-constraint] Subject failed verification: ghcr.io/feynmanzhou/alpine:latest@sha256:1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870
```

View Ratify logs

```
{
  "isSuccess": false,
  "verifierReports": [
    {
      "isSuccess": false,
      "name": "notaryv2",
      "message": "an error thrown by the verifier: failed to verify signature, err: signature is not produced by a trusted signer",
···
```
