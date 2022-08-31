<h2>Steps to secure Apicurio Registry backed by Kafka Storage with oauth-proxy</h2>


1. Install AMQ Streams and Apicurio Registry


2. Create a Kafka Cluster for Apicurio Registry storage
   
   a. Configure auth and tls fields to use TLS auth for the Kafka cluster (see configuration here: https://www.apicur.io/registry/docs/apicurio-registry-operator/1.0.0/assembly-registry-storage.html#registry-persistence-kafkasql-tls)

   b. The Kafka topic kafjkasql-journal is created automatically by Apicurio Registry. This can be overridden via environment variables


3. Create a Kafka User resource with the necessary configuration (sample found here: https://www.apicur.io/registry/docs/apicurio-registry-operator/1.0.0/assembly-registry-storage.html#registry-persistence-kafkasql-tls)


4. Verify these 2 secrets were created after Kafka User is created:
   
   a. `<kafka-cluster-name>-cluster-ca-cert` contains the truststore 

   b. `<kafka-user-name>` contains the keystore
   

6. Create an Apicurio Registry instance with the operator with the following configuration:
   ``` 
   apiVersion: registry.apicur.io/v1
   kind: ApicurioRegistry
   metadata:
      name: example-apicurioregistry-kafkasql
   spec:
      configuration:
         persistence: "kafkasql"
         kafkasql:
            bootstrapServers: <bootstrap server with TLS from Kafka cluster>
            security:
               tls:
                  keystoreSecretName: <kafka-user-name>
                  truststoreSecretName: <kafka-cluster-name-cluster-ca-cert>
   ```

7. Verify you have the following Apicurio Registry resources:
   
   a. Deployment 

   b. Service
   
   c. Ingress

   d. Route 
   
   e. CRD of kind ApicurioRegistry


9. Add a Secret named `cookie-secret` in the desired namespace with the following information:
   ```
   kind: Secret
   apiVersion: v1
   metadata:
      name: cookie-secret
      namespace: <your-namespace>
   data:
      cookie-secret: ZGQ5ZTYwMWNjNDE5MDJjZGYxMzM0Y2I1YTc4M2NmZTQxY2QzOGFhNDc2MA==
   type: Opaque
   ```

10. Add a ServiceAccount named `sa-registry` in the desired namespace with the following information:
    ```
    apiVersion: v1
    kind: ServiceAccount
    metadata:
       name: sa-registry
    annotations:
       serviceaccounts.openshift.io/oauth-redirectreference.primary: '{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"<route-name>"}}'
    ```

11. Update the Apicurio Registry Service to include the following annotation which will create a certificate and save it as a Secret:

    `service.alpha.openshift.io/serving-cert-secret-name: oauth-proxy-tls`

    
12. Also in the Service, add an additional port for the oauth-proxy like so:

    ```
    - name: oauth-proxy
      port: 443
      targetPort: 8443
    ```

13. Update the Apicurio Registry Deployment to include an additional container for the oauth-proxy, mount 2 additional volumes for the Service-created certificate and the cookie-secret, and add a reference to the sa-registry ServiceAccount.


15. Update the Apicurio Registry Ingress to create a Reencrypt Route and point to the oauth-proxy port:


        annotations:
           route.openshift.io/termination: reencrypt
        ...
        rules:
            - host: >-
                apicurioregistry-kafkasql-tls.apicurio-oauth.router-default.apps.cluster-mmxzg.mmxzg.sandbox1072.opentlc.com
              http:
                paths:
                    - pathType: ImplementationSpecific
                      backend:
                        service:
                            name: <apicurioregistry-service-name>
                            port:
                                name: oauth-proxy


16. Update the sa-registry ServiceAccount to include the correct Route name that was created by the Ingress


