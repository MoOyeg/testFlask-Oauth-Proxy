# testFlask-Oauth-Proxy
Add authentication to an application using Openshift's Internal Oauth Server via Proxy.<br/> 
Application used is:<br/>
testFlask Application - https://link](https://github.com/MoOyeg/testFlask.<br/>
it's a flask application that shows how to run a flask application in Openshift.

**Find More Information about Oauth Proxy and examples below:** <br/>
[Github oauth-Proxy](https://github.com/openshift/oauth-proxy.git)<br/>
[Using OpenShift OAuth Proxy to secure your Applications on OpenShift](https://linuxera.org/oauth-proxy-secure-applications-openshift/)

----------

## Steps to Run
### Source Environment Variables
`eval "$(curl https://raw.githubusercontent.com/MoOyeg/testFlask/master/sample_env)"`

### 1 Optional - Create OAuth-Proxy Image:<br/>
- You can create the oauth-proxy image yourself<br/>   
`export OAUTH_DOCKERFILE=$(curl https://raw.githubusercontent.com/MoOyeg/testFlask-Oauth-Proxy/main/Dockerfile)`

  `oc new-build --strategy=docker -D="$OAUTH_DOCKERFILE" --name=oauth-proxy -n ${NAMESPACE_PROD}`

### 2 Create the Unencrypted Version of the Application for this demo.Please copy the steps from:<br/>
testFlask Application - https://link](https://github.com/MoOyeg/testFlask.<br/>
- Steps Below might not be updated, See above link for updated steps
```
oc adm new-project ${NAMESPACE_PROD}

oc create secret generic my-secret --from-literal=MYSQL_USER=$MYSQL_USER --from-literal=MYSQL_PASSWORD=$MYSQL_PASSWORD -n $NAMESPACE_PROD

oc new-app $MYSQL_HOST --env=MYSQL_DATABASE=$MYSQL_DATABASE -l db=mysql -l app=testflask -n $NAMESPACE_PROD --as-deployment-config=true

oc set env dc/$MYSQL_HOST --from=secret/my-secret -n $NAMESPACE_PROD

oc new-app https://github.com/MoOyeg/testFlask.git --name=$APP_NAME -l app=testflask --strategy=source --env=APP_CONFIG=gunicorn.conf.py --env=APP_MODULE=testapp:app --env=MYSQL_HOST=$MYSQL_HOST --env=MYSQL_DATABASE=$MYSQL_DATABASE --as-deployment-config=true -n $NAMESPACE_PROD

oc expose svc/$APP_NAME -n $NAMESPACE_PROD

oc set env dc/$APP_NAME --from=secret/my-secret -n $NAMESPACE_PROD

oc label dc/$APP_NAME app.kubernetes.io/part-of=$APP_NAME -n $NAMESPACE_PROD
oc label dc/$MYSQL_HOST app.kubernetes.io/part-of=$APP_NAME -n $NAMESPACE_PROD
oc annotate dc/$APP_NAME app.openshift.io/connects-to=$MYSQL_HOST -n $NAMESPACE_PROD


```

### 3 We are using the Openshift Service CA to provide TLS Certificates for our service, if you have your own certs you can provide them. To understand more about the 
[Openshift Service CA](https://docs.openshift.com/container-platform/4.6/security/certificates/service-serving-certificate.html):

- Annotate the Service to use the Openshift Serving CA provided certs and secrets<br/>
`oc annotate service ${APP_NAME} service.beta.openshift.io/serving-cert-secret-name=${APP_NAME}-secret-tls -n ${NAMESPACE_PROD}`

### 4 For the OAuth Proxy to work we need to use our Service Account as an Oauth Client and provide a redirect uri when the internal oauth tries to callback. For the Redirect URI we will be using our Application Route. To understand more see 
[Service Account as Oauth Client](https://docs.openshift.com/container-platform/4.6/authentication/using-service-accounts-as-oauth-client.html)

- Annotate the ServiceAccount with an OauthRedirect Reference pointing to our Route.<br/>
`oc -n ${NAMESPACE_PROD} annotate serviceaccount default serviceaccounts.openshift.io/oauth-redirectreference.first='{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"testflask"}}'`

### 5 Create a Cookie Session Secret to use on the browser<br/>

  - `oc -n ${NAMESPACE_PROD} create secret generic ${NAMESPACE_PROD}-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)` 


### 6 Patch the Application Deployment Config with the oauth-proxy sidecar.If you Create the image yourself remember to update the patch.

   - `oc patch dc/${APP_NAME} --patch "$(curl https://raw.githubusercontent.com/MoOyeg/testFlask-Oauth-Proxy/main/patch-dc.yaml)" -n ${NAMESPACE_PROD}`

### 7 Patch the Service with the new Oauth Proxy Port

   - `oc patch svc/${APP_NAME} --patch "$(curl https://raw.githubusercontent.com/MoOyeg/testFlask-Oauth-Proxy/main/patch-svc.yaml)" -n ${NAMESPACE_PROD}`

### 8 Mount the Service CA Secret on the Oauth Proxy Container

   - `oc set volume dc/${APP_NAME} --add --containers=oauth-proxy -t=secret --secret-name=${APP_NAME}-secret-tls --mount-path=/etc/tls/private -n ${NAMESPACE_PROD}`

### 9 Mount the Cookie Secret on the Oauth Proxy Container
   - `oc set volume dc/${APP_NAME} --add --containers=oauth-proxy -t=secret --secret-name=${NAMESPACE_PROD}-proxy --mount-path=/etc/proxy/secrets -n ${NAMESPACE_PROD}`


### Patch the Route to enable TLS Passthrough and to route to the Oauth Pod instead of the Application
   - `oc patch route/${APP_NAME} --patch "$(curl https://raw.githubusercontent.com/MoOyeg/testFlask-Oauth-Proxy/main/patch-route.yaml)" -n ${NAMESPACE_PROD}`


### If working as expected opening the route should redirect to the interal Oauth Server.Note Route will be https if TLS was enabled above.
 - You can get the route from:<br/>
  `oc get routes -n ${NAMESPACE_PROD} ${APP_NAME} -o jsonpath='{.spec.host}'`
