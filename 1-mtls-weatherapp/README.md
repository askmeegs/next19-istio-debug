# notes - debugging istio mtls + weatherapp 

prerequisites 
- permissive cluster with the istio add-on
- inject default namespace 


https://bani.com.br/2018/08/istio-mtls-debugging-a-503-error/
https://istio.io/help/ops/security/keys-and-certs/
https://istio.io/help/ops/security/mutual-tls/

## Create secret 

```
kubectl create secret generic openweathermap --from-literal=apikey="62bc56feae84c10baabb80a1369359c9"
```

## Deploy weatherapp / make sure it's working  


## Hit in browser 

## Enable mtls for the weather-backend  

```
kubectl apply -f mtls-backend-1.yaml
```

Open the frontend: 

![screenshot](mtls-error.png)

503 

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
stern -n istio-system pilot -c discovery
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

## Open the Envoy Admin console on the weather-backend pod 

```
kubectl port-forward weather-backend-multiple-68798f5c49-7pnsd 15000:15000
```

go to localhost:15000 

## Set logging to trace 

exec into backend pod 

curl -X POST "http://localhost:15000/logging?level=trace"


## View the backend proxy logs again 

```
kubectl logs weather-backend-multiple-68798f5c49-7pnsd -c istio-proxy | grep handshake
```

*this tells us that the frontend is not authenticating properly to the backend via mtLS* 


## Is the control plane doing its job?

Check that citadel is actually generating certs 

```
kubectl get secret istio.default
```

Is the cert valid? 

```
kubectl get secret -o json istio.default | jq -r '.data["cert-chain.pem"]' | base64 --decode | openssl x509 -noout -text
```

## Is this cert being correctly mounted into the pods? 

```
kubectl exec -it weather-backend-multiple-68798f5c49-7pnsd -c istio-proxy -- ls /etc/certs

kubectl exec -it weather-frontend-7ff5d6d698-j5nhl -c istio-proxy -- ls /etc/certs
```

## Hypothesis, after all this 

*so it must be our fault.*

*let's check our config* 


## Back to the YAML 

Look at the policy 

Look at the destinationrule and the service 