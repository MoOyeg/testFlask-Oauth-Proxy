# testFlask-Oauth-Proxy

Add authentication to an application using Openshift's Internal Oauth Server via an Oauth Proxy.  
Application used is:  

[TestFlask Application](https://github.com/MoOyeg/testFlask): it's a flask application that shows how to run a flask applications in Openshift.

## Find More Information about Oauth Proxy and examples below 
- [Github oauth-Proxy](https://github.com/openshift/oauth-proxy.git)  
- [Using OpenShift OAuth Proxy to secure your Applications on OpenShift](https://linuxera.org/oauth-proxy-secure-applications-openshift/)

## Steps to Run

- Source Environment Variables
   ```bash
   eval "$(curl https://raw.githubusercontent.com/MoOyeg/testFlask/master/sample_env)"
   ```

- Optional: Create OAuth-Proxy Image  

   - You can create the oauth-proxy image yourself

      ```bash
      export OAUTH_DOCKERFILE=$(curl https://raw.githubusercontent.com/MoOyeg/testFlask-Oauth-Proxy/main/Dockerfile)
      ```

      ```bash
      oc new-build --strategy=docker -D="$OAUTH_DOCKERFILE" --name=oauth-proxy -n ${NAMESPACE_DEV}
      ```

- Create the Unsecured Version of the Application for this demo.
   - Start by following the steps to create our [TestFlask Application](https://github.com/MoOyeg/testFlask#steps-to-build-and-run-application)

   - After creating the application set an environment variable to let our application know we are using oauth authentication.
      ```bash
      oc set env deploy/${APP_NAME} AUTH_INTEGRATION=true AUTH_TYPE=openshift_oauth_proxy -n $NAMESPACE_DEV
      ```  

- We are using the Openshift Service CA to provide TLS Certificates for our service, if you have your own certs you can provide them. To understand more about the [Openshift Service CA](https://docs.openshift.com/container-platform/4.6/security/certificates/service-serving-certificate.html)

  - Annotate the Service to use the Openshift Serving CA provided certs and secrets  

      ```bash
      oc annotate service ${APP_NAME} service.beta.openshift.io/serving-cert-secret-name=${APP_NAME}-secret-tls -n ${NAMESPACE_DEV}
      ```

- For the OAuth Proxy to work we need to use our Service Account as an Oauth Client and provide a redirect uri when the internal oauth tries to callback. For the Redirect URI we will be using our Application Route. To understand more see [Service Account as Oauth Client](https://docs.openshift.com/container-platform/4.6/authentication/using-service-accounts-as-oauth-client.html)

- Annotate the ServiceAccount with an OauthRedirect Reference pointing to our Route.  
   ```bash
   oc -n ${NAMESPACE_DEV} annotate serviceaccount default serviceaccounts.openshift.io/oauth-redirectreference.first='{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"testflask"}}'
   ```

- Create a Cookie Session Secret to use on the browser  

  ```bash
  oc -n ${NAMESPACE_DEV} create secret generic ${NAMESPACE_DEV}-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
  ``` 


- Patch the Application Deployment with the oauth-proxy sidecar.If you Create the image yourself remember to update the patch.

   ```bash
   oc patch deploy/${APP_NAME} --patch "$(curl https://raw.githubusercontent.com/MoOyeg/testFlask-Oauth-Proxy/main/patch-deploy.yaml)" -n ${NAMESPACE_DEV}
   ```

- Patch the Service with the new Oauth Proxy Port

   ```bash
   oc patch svc/${APP_NAME} --patch "$(curl https://raw.githubusercontent.com/MoOyeg/testFlask-Oauth-Proxy/main/patch-svc.yaml)" -n ${NAMESPACE_DEV}
   ```

- Mount the Service CA Secret on the Oauth Proxy Container

   ```bash
   oc set volume deploy/${APP_NAME} --add --containers=oauth-proxy -t=secret --secret-name=${APP_NAME}-secret-tls --mount-path=/etc/tls/private -n ${NAMESPACE_DEV}
   ```

- Mount the Cookie Secret on the Oauth Proxy Container
   ```bash
   oc set volume deploy/${APP_NAME} --add --containers=oauth-proxy -t=secret --secret-name=${NAMESPACE_DEV}-proxy --mount-path=/etc/proxy/secrets -n ${NAMESPACE_DEV}
   ```


- Patch the Route to enable TLS Passthrough and to route to the Oauth Pod instead of the Application
   ```bash
   oc patch route/${APP_NAME} --patch "$(curl https://raw.githubusercontent.com/MoOyeg/testFlask-Oauth-Proxy/main/patch-route.yaml)" -n ${NAMESPACE_DEV}
   ```


- If working as expected opening the route should redirect to the interal Oauth Server.Note Route will be https if TLS was enabled above.
   - You can get the route from:  
      ```bash
      oc get routes -n ${NAMESPACE_DEV} ${APP_NAME} -o jsonpath='{.spec.host}'
      ```
