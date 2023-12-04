---
title: "OAuth2 Proxy with Istio"
date: 2020-12-04T17:00:00+05:30
# weight: 1
# aliases: ["/first"]
author: "Farzan"
tags: ["istio", "kubernetes", "oauth2-proxy", "kyverno", "kustomize"]
ShowToc: true
TocOpen: true
---

## Intro

let's say there is an app you want to deploy on your kubernetes cluster, but the app doesn't ship with an auth module. or at least an auth module that supports oauth2. how would you do it? do you write your own auth module? nah, that's too much work. do you just trust the honor system and deploy the app without auth until one day you get an email from openai that your API Key was compromised? nah, already did that. okay, so what do you do? 

this was a problem I faced recently. looking around I came accross a number of oauth2 relays. these are intermediate services that sit between the ingress of our cluster and our app. they handle the oauth2 flow and then forward the request to the app.

the one that caught my eye was [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy). it's a simple oauth2 relay that supports a number of oauth2 providers. it's also written in go, which is a plus for me.

## oauth2-proxy

in an of itself oauth2-proxy is relatively simple to deploy. here's a simple deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: oauth2-proxy
    namespace: default
spec:
    replicas: 1
    selector:
    matchLabels:
        app: oauth2-proxy
    template:
    metadata:
        labels:
        app: oauth2-proxy
    spec:
        containers:
            - name: oauth2-proxy
            image: "quay.io/oauth2-proxy/oauth2-proxy:v7.5.1"
            imagePullPolicy: IfNotPresent
            args:
                ## this are the default args
                - --http-address=0.0.0.0:4180
                - --https-address=0.0.0.0:4443
                - --metrics-address=0.0.0.0:44180
                - --auth-logging=true
                - --cookie-httponly=true
                - --cookie-secure=true
                - --cookie-expire=0
                - --cookie-refresh=1h
                - --request-logging=true
                - --set-authorization-header=true
                - --silence-ping-logging=true
                - --skip-provider-button=true
                - --skip-auth-strip-headers=false
                - --skip-jwt-bearer-tokens=true
                - --standard-logging=true
                - --reverse-proxy=true
                - --upstream="static://200"
                - --pass-access-token=true
                - --pass-authorization-header=true    
                ## this is the part where we configure our oauth2 provider
                - --provider=azure
                - --azure-tenant="[AZURE TENANT ID]"            
                - --oidc-issuer-url=https://login.microsoftonline.com/[AZURE TENANT ID]/v2.0
                ## we can whitelist domains and emails
                - --email-domain="*"
                - --whitelist-domain=farzaaaan.com
                ## this is where we use redis to manage sessions
                - --session-store-type=redis
                - --redis-connection-url=redis://redis-master.oauth2-proxy.svc.cluster.local:6379
                - --redis-password=[REDIS PASSWORD]
                           
            env:
            - name: OAUTH2_PROXY_CLIENT_ID
                value: "[CLIENT ID FROM OUR OAUTH2 PROVIDER]"
            - name: OAUTH2_PROXY_CLIENT_SECRET
                value: "[CLIENT SECRET FROM OUR OAUTH2 PROVIDER]"
            - name: OAUTH2_PROXY_COOKIE_SECRET
                value: "[RANDOM base64 To SPICE UP OUR COOKIES]"
            ports:
                - containerPort: 4180
                name: http
                protocol: TCP
                - containerPort: 44180
                protocol: TCP
                name: metrics
            livenessProbe:
                httpGet:
                path: /ping
                port: http
                scheme: HTTP
                initialDelaySeconds: 0
                timeoutSeconds: 1
            readinessProbe:
                httpGet:
                path: /ready
                port: http
                scheme: HTTP
                initialDelaySeconds: 0
                timeoutSeconds: 5
                successThreshold: 1
                periodSeconds: 10
            resources:
                {}
```

once we have this deployment running, we could use whatever ingress we have to route traffic to it when there are no `Authorization` headers. oauth2-proxy will then redirect the user to whatever oauth2 provider we configured and then redirect them back to the app with an `Authorization` header containing a JWT token.

relatively simple, that is if we just want one default app to authorize all our applications. but more likely than not, this won't be the case. this means we'll need to manage multiple oauth2-proxy deployments, one for each app. this is where things get a bit more complicated, especially add to the mix a service mesh like istio.

## oauth2-proxy with istio

[istio](https://istio.io/) has a pretty solid traffic management setup. it's quickly become my go to service mesh. as a quick recap: istio utilizes a number of its own CRDs to manage traffic. these are: 
- __Gateway__ : this is our ingress gateway, it listens on a host and can terminate TLS.
- __VirtualService__ : this is where we define our routing rules, a virtual service could listen to a Gateway androute traffic to a service based on a number of criteria. this is where we can add additonal policies for Authentication and Authorization.
- __DestinationRule__ : this is where we define our traffic policies, we can define things like TLS settings, load balancing settings, circuit breaking settings, etc. unlike VirtualService that defines routes for a request to a service, DestinationRule defines what happens to that request when it arrives at that service. as you could guess, by the time we get to the DestinationRule it's too late to add any Authentication or Authorization policies.
- __RequestAuthentication__ : this is where we define our Authentication policies, we can define things like JWT tokens, API keys, or in our case, external auth.
- __AuthorizationPolicy__ : this is where we define our Authorization policies, we can define things like allow/deny lists, etc. If an AuthorizationPolicy denies on a given criteria, RequestAuthentication is called.


### istio: the intro

let's start with a simple istio setup. we'll have a simple app that we want to deploy, and we want to add oauth2-proxy to it. we'll also have a simple ingress gateway that will route traffic to our app. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
    name: ingress-gateway
    namespace: default
spec:
    selector:
    istio: ingressgateway
    servers:
    - hosts:
        - foo.example.com
        port:
        name: http
        number: 80
        protocol: HTTP
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
    name: app
    namespace: default
spec:
    hosts:
    - foo.example.com
    gateways:
    - ingress-gateway
    http:
    - match:
        - uri:
            prefix: /
        route:
        - destination:
            host: app
            port:
            number: 80
```

now let's say I want to add an AuthorizationPolicy and AuthenticationPolicy to this setup. but not for the whole site, just where path starts with `/admin`. when a request comes in, I want to check if a JWT token is available via AuthorizationPolicy, if it is then I'll validate it otherwise I'll request one via AuthenticationPolicy. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: AuthorizationPolicy
metadata:
    name: app
    namespace: default
spec:
    selector:
    matchLabels:
        app: app
    action: DENY
    rules:
    - from:
        - source:
            requestPrincipals: ["*"]
    when:
    - key: request.path
        values:
        - /admin*
---
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
    name: app
    namespace: default
spec:

    selector:
    matchLabels:
        app: app
    jwtRules:
    - issuer: "http://foo.example.com"
        jwksUri: "https://our.oauth2.provider/.well-known/jwks.json"
        audiences:
        - "http://foo.example.com"     
```

the behaviour I would now see upon a request is: 

```bash
curl -v http://foo.example.com/

## out: 200

curl -v http://foo.example.com/admin

## out: 401

curl -v http://foo.example.com/admin -H "Authorization=Bearer [JWT TOKEN]"

## out: 200

```

this is all well and good, but what if I want to add oauth2-proxy to this setup? so far what's happening here is that I, as the user, am passing a JWT token with my request. istio then validates that token and allows the request to go through. but I can't possibly expect my users to go through the trouble of getting a JWT token every time they want to send requests to my app. rather: I would like to check for a JWT token, and if not present I want to redirect the user to my oauth2 provider to login, get a JWT token, come back to the original request url and continue with their session. and I want all of this to happen seamlessly.


### istio: external auth