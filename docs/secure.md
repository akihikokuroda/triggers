
# Securing communication
The communication between the event source and the event listner must be secure to gurantee the privacy and data integrity of events.  Transport Layer Security (TLS)  protocol provides the secure communication. The HTTPS protocol is the HTTP protocol over TLS commnication.  The event source must be configured to send the event using TLS.  The event listener must be configured to receive the the event sent using TLS.
## Confuging Event listener
The event from outside of the kubernetes cluster comes through one of the exposed entry points and is routed to the event listener.  The ingress or route can be defined as the entry point for the event listener and it can be configured to receive the event using TLS. 
### Ingress 
Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.
#### Prerequisite
The certificate used for the ingress must be prepared before creating the ingress for the event listener.  The certificate file and certificate key file in the pem format are necessary.  If there isn't, create a selfsigned certificate.
#### Steps

1. Create a work directory and create the ingress.yaml file with the following contents in the directory.
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ${EVENT_LISTENER_NAME}
  namespace: ${NAMESPACE}
spec:
  tls:
  - hosts:
    - ${URL}
    secretName: ${CERTIFICATE_SECRET_NAME}
  rules:
  - host: ${URL}
    http:
      paths:
      - backend:
          serviceName: ${EVENT_LISTENER_NAME}
          servicePort: 8082
```
2. Create the ingress.sh with the following contents in the work directory.
```
#!/bin/bash

# work directory (path to cert, keys, script & yaml file)
export WORK_DIR=""

# certificate data
# *Make sure names contain only lowercase alphanumeric characters, . or -. Must start & end with alphanumeric characters*
export CERTIFICATE_KEY=""
export CERTIFICATE_NAME=""
# IP address of the Proxy node (or master node)
export IP_ADDRESS=""
export URL="${EVENT_LISTENER_NAME}.${IP_ADDRESS}.nip.io"
export NAMESPACE=""
export EVENT_LISTENER_NAME=""

# create the secret
kubectl create secret tls ${CERTIFICATE_SECRET_NAME} --cert=${WORK_DIR}/${CERTIFICATE_NAME}.pem --key=${WORK_DIR}/${CERTIFICATE_KEY}.pem -n $(NAMESPACE)

# populate variables in https-ingress & apply yaml file:
envsubst < ${WORK_DIR}/https-ingress.yaml | kubectl apply -f -

echo "Done. Now access the host with https://"${URL}
```
3. Edit ingress.sh file with your configuration information
4. Place the certificate files ($(CERTIFICATE_NAME).pem and $(CERTIFICATE_KEY).pem) in the work directory 
5. Execute ingress.sh

These steps expose the event listener at "https://\<event listener name\>.\<proxy node IP\>.nip.io/".  Use this URL in the event source configuration.

Reference: [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

### Generate self signed certificate

If the cluster doesn't have the CA signed certificate or the cluster CA certificate, the selfsigned certificate must be generated to set to the ingress.

#### Steps

1. Create a working directory
2. Create the certificate.sh with the following contests in the work directory.

```
#!/bin/bash

# work directory (path to cert, keys, script & yaml file)
export WORK_DIR=""

# certificate data
# *Make sure names contain only lowercase alphanumeric characters, . or -. Must start & end with alphanumeric characters*
export CERTIFICATE_KEY=""
export CERTIFICATE_KEY_PASSPHRASE=""
export CERTIFICATE_NAME=""
export CERTIFICATE_SECRET_NAME=""
# optional certificate information
export COUNTRY=""
export STATE=""
export LOCATION=""
export ORGANIZATION=""
export ORGANIZATIONAl_UNIT=""
export COMMON_NAME=$URL

# create a private key for the CA & add passphrase
openssl genrsa -des3 -out ${WORK_DIR}/${CERTIFICATE_KEY}.pem -passout pass:${CERTIFICATE_KEY_PASSPHRASE} 2048

# generate the root CA
openssl req -x509 -new -nodes -key ${WORK_DIR}/${CERTIFICATE_KEY}.pem -sha256 -days 1825 -out ${WORK_DIR}/${CERTIFICATE_NAME}.pem -passin pass:${CERTIFICATE_KEY_PASSPHRASE} -subj /C=${COUNTRY}/ST=${STATE}/L=${LOCATION}/O=${ORGANIZATION}/OU=${ORGANIZATIONAl_UNIT}/CN=${COMMON_NAME}

# for some reason the key wasn't being parsed when trying to create it with kubectl so this command fixes it
openssl rsa -in ${INGRESS_DIR}/${CERTIFICATE_KEY}.pem -out ${INGRESS_DIR}/${CERTIFICATE_KEY}.pem -passin pass:${CERTIFICATE_KEY_PASSPHRASE}

echo "Done."
```
3. Edit certificate.sh file with your configuration information
4. Execute certificate.sh
5. Certificate files ($(CERTIFICATE_NAME).pem and $(CERTIFICATE_KEY).pem) are in the work directlry

Reference: [Certificate generation](https://www.openssl.org/docs/man1.1.1/man1/)
### Ingress with cert-manager
If your cluster has the cert-manager running, the signed certificate is automatically created and attached for the ingress.

Refernece: [Cert-manager](https://itnext.io/automated-tls-with-cert-manager-and-letsencrypt-for-kubernetes-7daaa5e0cae4)
### Route
In the OpeShift cluster, the defining the route is easy way to configure the entry point with TLS. 

#### Steps
1. Create a work directory and create the route.yaml file with the following contents in the directory.
```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ${EVENT_LISTENER_NAME}
  namespace: ${NAMESPACE}
spec:
  to:
    kind: Service
    name: ${EVENT_LISTENER_NAME}
  tls:
    termination: Reencrypt
```

Reference: [Route](https://docs.openshift.com/container-platform/4.1/networking/routes/route-configuration.html)
## Configuring Event source
The way to configure the event source is unique for each event source.
### github
In the "Webhooks / Add webhook" page (repository->settings->Hooks->Add webhook) in the github repository, the event listener URL with the "https" must be set in "Payload URL" field.  If the event listner doesn't have the CA signed certificate(self signed certificate), the "Disable" must be selected for "SSL verification". 
