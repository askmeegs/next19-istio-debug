# notes - debugging istio mtls + weatherapp 

prerequisites 
- cluster with the istio add-on
- inject default namespace 


https://bani.com.br/2018/08/istio-mtls-debugging-a-503-error/
https://istio.io/help/ops/security/keys-and-certs/
https://istio.io/help/ops/security/mutual-tls/

## 1 - Create secret 

```
kubectl create secret generic openweathermap --from-literal=apikey="<api-key-here>"
```

## 2 - Deploy weatherapp 


## 3 - Hit in browser 

## 4 - Enable mtls for the weather-backend  

```
kubectl apply -f mtls-backend-1.yaml
```

Open the frontend: 

![screenshot](mtls-error.png)

Oh no! Why did enabling mTLS for one service cause a 503 error between the frontend and the backend(s)? 

This is an Envoy error. 

## Envoy logs 

*stern frontend istio-proxy* 

*view the 503 error* 
```
weather-frontend-7ff5d6d698-9nfbv istio-proxy [2019-03-28T12:53:32.264Z] "GET /api/weatherHTTP/1.1" 503 UC 0 57 1 - "-" "python-requests/2.21.0" "b3785f8b-a8c8-4fc9-bde1-12ffa1e92a01" "weather-backend:5000" "10.52.1.12:5000" outbound|5000|multiple|weather-backend.default.svc.cluster.local - 10.55.253.172:5000 10.52.2.22:54984
```

*stern backend both* 

This command lets us see if traffic is getting to *either* of the 2 backends. 

```
stern weather-backend -c weather-backend
```

^ no traffic is getting to the actual backend server. now what about the envoy for either of those two? 

```
stern weather-backend -c istio-proxy 
```

(nothing) 

## Control plane logs 

Which control plane component is responsible for sending mTLS policies to envoys? https://istio.io/docs/concepts/security/#high-level-architecture (It's pilot) 

```
stern -n istio-system pilot -c pilot
```

Remove policy then re-create it to see the logs. 

```
istio-pilot-6bd6f7cdb-hhbfl discovery 2019-03-28T13:22:28.239906Z	info	ads	LDS: PUSH for node:weather-backend-single-84f4977f7-c78wg.default addr:"10.52.2.23:47460" listeners:29 55799
istio-pilot-6bd6f7cdb-hhbfl discovery 2019-03-28T13:22:28.240007Z	info	ads	Push finished: 227.271798ms {
istio-pilot-6bd6f7cdb-hhbfl discovery     "ProxyStatus": {
istio-pilot-6bd6f7cdb-hhbfl discovery         "pilot_conflict_outbound_listener_tcp_over_current_tcp": {
istio-pilot-6bd6f7cdb-hhbfl discovery             "0.0.0.0:443": {
istio-pilot-6bd6f7cdb-hhbfl discovery                 "proxy": "weather-backend-single-84f4977f7-c78wg.default",
istio-pilot-6bd6f7cdb-hhbfl discovery                 "message": "Listener=0.0.0.0:443 AcceptedTCP=accounts.google.com RejectedTCP=*.googleapis.com TCPServices=1"
istio-pilot-6bd6f7cdb-hhbfl discovery             }
istio-pilot-6bd6f7cdb-hhbfl discovery         }
istio-pilot-6bd6f7cdb-hhbfl discovery     },
```

What does this mean? 

Envoy LDS (Listeners) https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/lds 

Pilot is sending the mTLS policy to ONE of the weather backends (`single`) -- but where is the other? (`multiple`)? 
And if this is the case, then why are we seeing 503 errors 100% of the time, not half the time? 

## Back to the YAML 

Look at the policy 

Look at the destinationrule and the service 