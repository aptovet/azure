# Securing your AKS Deployments - Microservice User Authentication using Azure AD and Oauth 2 Proxy

In today's fast-paced software development environment, safeguarding your development and QA environments from public access is essential. At Aptovet, we've optimized our security process by integrating OAuth2 Proxy with Azure AD, combined with Kubernetes NGINX Ingress. This setup ensures that only authenticated users—developers, stakeholders, managers, and clients—can access these environments, effectively protecting them from unauthorized public access.

Additionally, this solution offers a significant advantage: it requires zero code changes within the application itself. Instead, all necessary adjustments are made at the gateway level through configuration changes, making it a more efficient and streamlined approach.


# Implementation

## Prerequisites

- Assuming we already have configured `aks` cluster and it has deployed `nginx-ingress` setup with a public IP load balancer.

- For SSL certificate management we have installed `cert-manager` deployed in kubernetes cluster
- `azure` subscription with a `resource group` created
- Fully qualified domain

## Azure Active Directory

- **Register an application**
```bash
az ad app create --display-name $APP_NAME --sign-in-audience AzureADMYOrg --web-home-page-url $DOMAIN/oauth2/callback --query appId
```
output:

```json
"00000000-0000-0000-0000-000000000000"
````

Will need this appId for generate secrets.

- **Create secret to application**
```bash
az ad app credential reset --id "00000000-0000-0000-0000-000000000000" --display-name $DISPLAY_NAME --end-date "2017-06-31"
```
Output :
```json
{
  "appId": "00000000-0000-0000-0000-000000000000",
  "password": "9vr8Q~SdsfsdfdsFC21kKM.sdkfjiewfhiow",
  "tenant": "00000000-0000-0000-0000-000000000000"
}
```

- Generate cookie secret
```bash
openssl rand -hex 16
```
output:
```text
2bd1adbe11e6e8e32f559869d3000000
```
## Add kubernetes secrets
```kubectl
kubectl create secret generic oauth2-proxy-client \
 --from-literal "oauth2_proxy_client_id=$APP_ID" \
 --from-literal "oauth2_proxy_client_secret=$PASSWORD" \
 --from-literal "oauth2_proxy_cookie_secret=$COOKIE_SECRET" \
 --namespace $NAMESPACE
```
output:
```text
secret/oauth2-proxy-client created
```


## Write kubernetes deployment file

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    application: <service_name>-service-oauth2-proxy
  name: <service_name>-service-oauth2-proxy-deployment
  namespace: <namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      application: <service_name>-service-oauth2-proxy
  template:
    metadata:
      labels:
        application: <service_name>-service-oauth2-proxy
    spec:
      containers:
      - args:
        - --provider=oidc
        - --azure-tenant=<azure_tenant_id> # Azure AD OAuth2 Proxy application Tenant ID
        - --pass-access-token=true
        - --cookie-name=_proxycookie 
        - --upstream=<service_public_domain>
        - --cookie-csrf-per-request=true
        - --cookie-csrf-expire=5m           # Avoid unauthorized csrf cookie errors.
        - --email-domain=*                  # Email domains allowed to use the proxy
        - --http-address=0.0.0.0:4180
        - --oidc-issuer-url=https://login.microsoftonline.com/<azure_tenant_id>/v2.0
        - --user-id-claim=oid

        name: <service_name>-service-oauth2-proxy
        image: quay.io/oauth2-proxy/oauth2-proxy:v7.4.0
        imagePullPolicy: Always      
        env:
        - name: OAUTH2_PROXY_CLIENT_ID # keep this name - it\'s required to be defined like this by OAuth2 Proxy
          valueFrom:
            secretKeyRef:
              name: oauth2-proxy-client
              key: oauth2_proxy_client_id
        - name: OAUTH2_PROXY_CLIENT_SECRET # keep this name - it\'s required to be defined like this by OAuth2 Proxy
          valueFrom:
            secretKeyRef:
              name: oauth2-proxy-client
              key: oauth2_proxy_client_secret
        - name: OAUTH2_PROXY_COOKIE_SECRET # keep this name - it\'s required to be defined like this by OAuth2 Proxy
          valueFrom:
            secretKeyRef:
              name: oauth2-proxy-client
              key: oauth2_proxy_cookie_secret
        ports:
        - containerPort: 4180
          protocol: TCP
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi

---
apiVersion: v1
kind: Service
metadata:
  labels:
    application: <service_name>-service-oauth2-proxy
  name: <service_name>-service-oauth2-proxy-svc
  namespace: <namespace>
spec:
  ports:
  - name: http
    port: 4180
    protocol: TCP
    targetPort: 4180
  selector:
    application: <service_name>-service-oauth2-proxy
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "2000m"
    nginx.ingress.kubernetes.io/proxy-buffer-size: "32k"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  name: <service_name>-service-oauth2-proxy-ingress
  namespace: <namespace>
spec:
  ingressClassName: nginx
  tls:
    - hosts:
      - <domain_name>
      secretName: <secret_name>
  rules:
     - http:
        paths:
          - path: /oauth2
            pathType: Prefix
            backend:
              service:
                name: <service_name>-service-oauth2-proxy-svc
                port:
                  number: 4180
       host: <domain_name>

```
> [!IMPORTANT]
> Change the `domain_name`, `secret_name`, `service_name`, `namespace`, `azure_tenant_id` From the manifest before deployment
>

- Deploy with kubectl 
  ```bash
  kubectl apply -f oauth-proxy-deployment.yaml
  ```
## Adjust targeted service ingress


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <namespace>-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"   
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/use-regex: 'true'
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-hash: "sha1"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "360"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "360"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "360"
    nginx.ingress.kubernetes.io/proxy-body-size: "2000m"
    nginx.ingress.kubernetes.io/proxy-buffer-size: "32k"
    nginx.ingress.kubernetes.io/auth-url: https://<domain_name>/oauth2/auth
    nginx.ingress.kubernetes.io/auth-signin: "https://<domain_name>/oauth2/start?rd=https://<domain_name>/oauth2/callback"
    nginx.ingress.kubernetes.io/auth-response-headers: "Authorization, X-Auth-Request-Email, X-Auth-Request-User, X-Forwarded-Access-Token"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - <domain_name>
    secretName: <secret_name> 
  rules:
  - host: <domain_name>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <svc_name>
            port:
              number: 5000
```
> [!IMPORTANT]
> Change the `domain_name`, `secret_name`, `svc_name`, `namespace` From the manifest before deployment
>

- Deploy with kubectl 
  ```bash
  kubectl apply -f ingress.yaml
  ```


Now try to access the application and it will redirect you to a microsoft login page. 