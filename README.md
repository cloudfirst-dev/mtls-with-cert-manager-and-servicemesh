# Ingress MTLS ServiceMesh with Cert-Manager Certs
This example will demonstrate using cert-manager (cert-manager.io) to generate MTLS certificates to use for authentication requests inbound to service mesh virtual services.  All certificates including the CA will be generated and managed by cert-manager.

## Assumptions
* Existing servicemesh control plane deployed to test-mesh with ior enabled
* Existing servicemesh member project created as mtls-test
* Cert-Manger operator installed on cluster

## Flow of Application
1. Create self-signed issuer
1. Create self signed CA certificate key pair
1. Create CA issuer from requested cert in previous step
1. Create server cert for ingress gateway
1. Manually copy secret from issuer namespace to istio namesapce (due to limitations of not being able to use secret references only paths are supported currently)
1. Create client cert for calling client
1. Deploy example service
1. Attempt to make call without certs to show failure
1. Attempt to make call with cert to show success

## Create Certificates

### Certificate Authority
This steps we will create a selfsigned issuer to generate a CA root certificate that will be used to sign the certs used for MTLS.

1. Create the selfsigned issuer to generate CA certs
    ```
    oc project mtls-test
    oc create -f ./ca-issuer.yaml
    ```
1. We then need to create a CA cert to sign all the certs for MTLS communication
    ```
    oc create -f ./ca-cert.yaml
    ```
1. The final step of preparing our CA is creating the CA issuer
    ```
    oc create -f ./mtls-issuer.yaml
    ```

### Create MTLS Certs
During this step is where we create the MTLS certs for the server and also the client to be used later in the guide.

1. Create the server certificate used by the ingress gateway
    ```
    sed -i '' 's/testmtls.apps.example.com/<your-dns-hostname-here>/g' ./service-cert.yaml
    oc create -f ./service-cert.yaml
    ```
1. Create the client certificate to be used by curl (example client)
    ```
    oc create -f ./client-cert.yaml
    ```

### Deploy Sample App
This is a simple spring boot web application that returns hello world.

1. Deploy the application
    ```
    oc create -f ./deployment.yaml
    ```

### Create the MTLS gateway
This is where we configure inbound traffic through the ingress gateway to require MTLS

1. Copy secret from mtls-test namespace to test-mesh
    ```
    oc get secret istio-ingressgateway-certs -o yaml > gateway-cert.yaml
    sed -i '' 's/mtls-test/test-mesh/g' ./gateway-cert.yaml 
    oc create -f ./gateway-cert.yaml
    ```
1. Create istio gateway.  You will need to replace <your-dns-hostname-here> with your actual wildcard address (ex. mtls.apps.example.com)
    ```
    sed -i '' 's/testmtls.apps.example.com/<your-dns-hostname-here>/g' ./gateway.yaml
    oc create -f ./gateway.yaml
    ```
1. Create a virtual service to route the traffic to the mesh service
    ```
    oc create -f ./virtual-service.yaml
    ```

### Validate that the service fails
1. Create a GET request with your dns host (ex. testmtls.apps.example.com).  Notice we will get an error of alert handshake failure
    ```
    oc get route -n test-mesh --selector=maistra.io/gateway-name=mtls-gateway -o=jsonpath=https://{.items[0].spec.host} | xargs curl --insecure
    ```

### Execute MTLS request with Curl
1. Get the client certs
    ```
    oc get secret client-apps-example-com-tls -o=jsonpath={'.data.ca\.crt'} | base64 -D > client-ca.crt
    oc get secret client-apps-example-com-tls -o=jsonpath={'.data.tls\.crt'} | base64 -D > client-tls.crt
    oc get secret client-apps-example-com-tls -o=jsonpath={'.data.tls\.key'} | base64 -D > client-tls.key
    ```
1. Curl with the client certs
    ```
    oc get route -n test-mesh --selector=maistra.io/gateway-name=mtls-gateway -o=jsonpath=https://{.items[0].spec.host} | xargs curl --cacert client-ca.crt --key client-tls.key --cert client-tls.crt
    ```