# imagesigning-openshift

## Pre-requisites

- Skopeo -- latest version to support GPG passphrase being passed (https://github.com/containers/skopeo/pull/1540)
- GPG
  
##  Build Process

  ```
    gpg --full-generate-key
    gpg --list-keys 
    gpg --export --armor email > signer-key-new.pub #This public key will be stored on the worker nodes to validate the signature of the images that are being pulled
    gpg --export-secret-key avyaan > my-key.key  #private key for gpg encryption  - store in keyvault?
    gpg --import ~/my-key.key  # import it back when build process starts

  ```
##  Container build

Build the image in the normal process that is used currently but push it to the registry using Skopeo

```
export CONTAINERS_SIGNATURE_PASSPHRASE="passwordhere" # passphrase used for creating the gpg key - only the latest version of skopeo will use this variable, older versions will prompt for a GPG passphrase (https://github.com/containers/skopeo/pull/1540)
```
```
skopeo copy --remove-signatures --sign-by emailusedabove docker://busybox docker://test.azurecr.io/mallam/busybox:2.0 
```
- Signatures are generated to a folder based on the config from default.yaml normally located under
/etc/containers/registries.d/default.yaml 
  - in my case, signatures are stored under /Users/rakeshkumarmallam/sigstore/

- Copy these signatures to azure blob storage, Note the container should be public read for the openshift
cluster to be able to read them.
```
azcopy copy "/Users/rakeshkumarmallam/sigstore/*" https://test.blob.core.windows.net/imagecontainer --recursive=true
```
## openshift config to allow only signed images from RedHat registries and Custom registries hosted internally.
```
cat > policy.json <<EOF
{
  "default": [
    {
      "type": "insecureAcceptAnything"
    }
  ],
  "transports": {
    "docker": {
      "yourregistryhere": [
        {
          "type": "signedBy",
          "keyType": "GPGKeys",
          "keyPath": "/etc/pki/developers/signer-key.pub"
        }
      ],
      "registry.access.redhat.com": [
        {
          "type": "signedBy",
          "keyType": "GPGKeys",
          "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release"
        }
      ],
      "registry.redhat.io": [
        {
          "type": "signedBy",
          "keyType": "GPGKeys",
          "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release"
        }
      ]
    },
    "docker-daemon": {
      "": [
        {
          "type": "insecureAcceptAnything"
        }
      ]
    }
  }
}
EOF

cat <<EOF > y


.yaml
docker:
     yourregistry:
         sigstore: https://test.blobra.core.windows.net/imagecontainer #location where the signatures are stored.
EOF

cat <<EOF > registry.access.redhat.com.yaml
docker:
     registry.access.redhat.com:
         sigstore: https://access.redhat.com/webassets/docker/content/sigstore
EOF

cat <<EOF > registry.redhat.io.yaml
docker:
     registry.redhat.io:
         sigstore: https://registry.redhat.io/containers/sigstore
EOF
```
- Covert the files to base64 for openshift to understand
```
export PUB_KEY=$(cat signer-key-new.pub | base64 -w0 )
export ARC_REG=$( cat registry.access.redhat.com.yaml | base64 -w0 )
export RIO_REG=$( cat registry.redhat.io.yaml | base64 -w0 )
export CU_REG=$(cat yourregistry.yaml | base64 -w0 )
export POLICY_CONFIG=$( cat policy.json | base64 -w0 )
```

- create config file for openshift
```
cat > 51-worker-rh-registry-trust.yaml <<EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 51-worker-rh-registry-trust
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,${ARC_REG}
        mode: 420
        path: /etc/containers/registries.d/registry.access.redhat.com.yaml
      - contents:
          source: data:text/plain;charset=utf-8;base64,${RIO_REG}
        mode: 420
        path: /etc/containers/registries.d/registry.redhat.io.yaml
      - contents:
          source: data:text/plain;charset=utf-8;base64,${CU_REG}
        mode: 420
        path: /etc/containers/registries.d/yourregistry.yaml
      - contents:
          source: data:text/plain;charset=utf-8;base64,${POLICY_CONFIG}
        mode: 420
        path: /etc/containers/policy.json
      - contents:
          source: data:text/plain;charset=utf-8;base64,${PUB_KEY}
        mode: 420
        path: /etc/pki/developers/signer-key.pub
EOF
```

- Apply machine config
```
oc apply -f 51-master-rh-registry-trust.yaml```
