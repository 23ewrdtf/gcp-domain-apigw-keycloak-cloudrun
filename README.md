# Domain to Cloud Run via API GW

# WARNING
```
This product or feature is covered by the Pre-GA Offerings Terms of the Google Cloud Terms of Service.
Pre-GA products and features might have limited support, and changes to pre-GA products and features 
might not be compatible with other pre-GA versions. For more information, see the launch stage descriptions.
```

# Official google guide
https://cloud.google.com/api-gateway/docs/gateway-serverless-neg

# Guide how to connect custom domain+LB+APIGW+KeyCloak+CloudRun running httpbin.

This guide is correct as of 27/02/2023.

Traffic Flow
 - Client > Domain > IP > 
    - if 443 > Forwarding Rule 443 > Target Proxy > URL Map > Backend > Network Endpoint Group > API GW (and keycloak) > Cloud Run
    - if port is 80 > Forwarding Rule 80 > Target Proxy > URL Map > Forwarding Rule 443 > Target Proxy > URL Map > Backend > Network Endpoint Group > API GW (and keycloak) > Cloud Run


# Diagram of below setup

![image](https://user-images.githubusercontent.com/8076798/221585631-37e21bb5-5184-497d-b632-5bc918c5060e.png)

# IP
Reserve a global public IP.

# Domain
Get a domain and add it to the Cloud DNS.

 - Use Certificate Manager and create a classic SSL Certificate managed by google.

 - Its provisioning will fail until we attach the domain to the load balancer at a later stage.

# Cloud Run

## Create a service account:

 - name: `example-com-cloud-run-sa`

 - role: none at the moment.

## Create a cloud run

 - Upload `httpbin` container to `Artifact Registry`.

 - name: `example-com-cloud-run-httpbin`.

 - image: `Uploaded image from Artifact Registry`.

 - port: `80`.

 - region: `europe-west2`.

 - Authentication: `Require authentication`.

 - After creating the cloud run service, tick the service name and select `show info panel`. 

 - `Add Principal` and add created earlier SA with role: `Cloud Run Invoker`

Going to URL of the cloud run should give error:

```
Error: Forbidden
Your client does not have permission to get URL / from this server.
``` 

# Keycloak

## Create or have access to Keycloak

 - Get OpenID config from `https://<keycloak>/realms/<realm>/.well-known/openid-configuration`

 - Open `https://<keycloak>/realms/<realm>/.well-known/openid-configuration>` and copy below values:

    - `issuer`

    - `jwks_uri`

    - `token_endpoint`

    - Create a new client with:

      - Client type: `OpenID Connect`

      - Client ID: `<anything>`

      - Client authentication: `Enabled`

      - Standard flow: `Disabled`

      - Direct access grants: `Disabled`

      - Service accounts roles: `Enabled`

      - Go To `Credentials` tab of the client and copy `Client secret`.

  - Open Postman

    - In the authorization tab select `OAuth 2.0`

    - Type Access Token URL from `token_endpoint`

    - Type `client id`

    - Type `Client Secret`

    - Click `Get New Access Token` and copy the `Access Token`

    - Type the token to your local jwt decoder (https://www.jvt.me/posts/2020/09/01/against-online-tooling/) and copy the `aud` value.

# API GW

## Create a new API GW

 - name, id, etc: `example-com-apigw`

 - Select a Service Account: `example-com-cloud-run-sa`

 - region: `europe-west2`

 - Copy Cloud Run URL and add to the below config file

 - Change x-google- values

```
# openapi2-functions.yaml
swagger: '2.0'
info:
  title: Http Bin
  description: Sample API on API Gateway with a Google Cloud Functions backend
  version: 1.0.0
schemes:
  - https
produces:
  - application/json
paths:
  /get:
    get:
      summary: Greet a user
      operationId: hello
      x-google-backend:
        address: https://example-com-cloud-run-httpbin.run.app/get
      responses:
        '200':
          description: A successful response
          schema:
            type: string
      security:
        - hello: []
  /ip:
    get:
      summary: Get IP
      operationId: ip
      x-google-backend:
        address: https://example-com-cloud-run-httpbin.run.app/ip
      responses:
        '200':
          description: A successful response
          schema:
            type: string
securityDefinitions:
  hello:
    authorizationUrl: ""
    flow: "implicit"
    type: "oauth2"
    x-google-issuer: "https://<keycloak>/realms/<realm>/"
    x-google-jwks_uri: "https://<keycloak>/realms/<realm>/protocol/openid-connect/certs"
    x-google-audiences: "account"
```

Once API GW is deployed use Postman to test.

Going to below URLs should give you:

 - `https://<gatewy url>.nw.gateway.dev/` - `{"code":404,"message":"The current request is not defined by this API."}`

 - `https://<gatewy url>.nw.gateway.dev/ip` - `your IP`

 - `https://<gatewy url>.nw.gateway.dev/get` - `{"code":401,"message":"Jwt is missing"}`

# Load Balancer

Load balancer in front of API Gateway is required if you want to use domains other than provided by API GW.

## Create a network endpoint group

This should point to the config ID of the API GW.

```
gcloud beta compute network-endpoint-groups create example-com-neg \
  --region=europe-west2 \
  --network-endpoint-type=serverless \
  --serverless-deployment-platform=apigateway.googleapis.com \
  --serverless-deployment-resource=example-com-apigw
```

## Create a backend

```
gcloud compute backend-services create example-com-backend --global
 ```

## Attach a network endpoint group the back end

```
gcloud compute backend-services add-backend example-com-backend \
  --global \
  --network-endpoint-group=example-com-neg \
  --network-endpoint-group-region=europe-west2
```

## Create an URL Map pointing to the backend

```
gcloud compute url-maps create example-com-urlmap \
  --default-service=example-com-backend
``` 

## Create a Target HTTPS Proxy with URL Map

```
gcloud compute target-https-proxies create example-com-target-https-proxy \
  --ssl-certificates=example-com-ssl \
  --url-map=example-com-urlmap
``` 

## Create a Forwarding Rule with target https proxy

```
gcloud compute forwarding-rules create example-com-forwarding-rule \
  --target-https-proxy=example-com-target-https-proxy \
  --global \
  --ports=443 \
  --address=<IP address reserved earlier>
``` 

## Create a HTTP to HTTPS redirect URL Map from below yaml

```
gcloud compute url-maps import example-com-urlmap-http-to-https-redirect \
  --source example-com-urlmap-http-to-https-redirect.yaml \
  --global
``` 

#### yaml

```
kind: compute#urlMap
name: example-com-urlmap-http-to-https-redirect
defaultUrlRedirect:
  redirectResponseCode: MOVED_PERMANENTLY_DEFAULT
  httpsRedirect: True
``` 

## Create a Target HTTP Proxy with URL Map

```
gcloud compute target-http-proxies create example-com-target-http-proxy \
  --url-map=example-com-urlmap-http-to-https-redirect
``` 

## Create a Forwarding Rule with target HTTP proxy

```
gcloud compute forwarding-rules create example-com-forwarding-rule-http \
  --target-http-proxy=example-com-target-http-proxy \
  --global \
  --ports=80 \
  --address=<IP address reserved earlier>
``` 

# Domain

Assign A record of the domain to the IP address you created earlier.

Please wait 24h for certificate provisioning before contacting support.

Going to below URLs should give you:

 - `https://example.com/` - `{"code":404,"message":"The current request is not defined by this API."}`

 - `https://example.com/ip` - `your IP`

 - `https://example.com/get` - `{"code":401,"message":"Jwt is missing"}`

 - Going to `http` should redirect to `https`.
