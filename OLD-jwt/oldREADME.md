## Debugging JWT Authentication 


## setup 
apply jwt-frontend-1 


## test if JWT works 

 TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/demo.jwt -s)

*when JWT works, this should work, and vice versa* 

curl --header "Authorization: Bearer $TOKEN" http://35.224.242.220/ 




## postman - should see

*without JWT - origin failed* 

*with valid JWT* 


## how far does the request get? 

Show frontend logs / istio-proxy 


## is it an ingress issue? 

Show ingress gateway logs -- traffic *does* make it through the gateway but it 503s before ingressgate routes   

```
[2019-04-02T16:55:02.720Z] "GET /HTTP/1.1" 503 UF 0 57 32 - "10.48.0.1" "PostmanRuntime/7.6.1" "8c5e03a5-67b1-4faa-aaed-b709c03d53df" "35.224.242.220" "10.48.3.6:5000" outbound|5000||weather-frontend.default.svc.cluster.local - 10.48.0.12:80 10.48.0.1:36196
    value: 503
```


## look at the gateway / virtual service resources 


## the problem -- origin authn policy does not denote mTLS 