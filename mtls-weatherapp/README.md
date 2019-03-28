# notes - debugging istio mtls + weatherapp 

prerequisites 
- cluster with the istio add-on
- inject default namespace 


## 1 - Create secret 

```
kubectl create secret generic openweathermap --from-literal=apikey="<api-key-here>"
```

## 2 - Deploy weatherapp 

Single-view only (Austin, TX) 

```
kubectl apply -f weatherapp-single-only.yaml 
```


## 3 - Hit in browser 

http://35.239.119.133/


## 4 - Enable mtls for the backend 

```
kubectl apply -f mtls-backend-1.yaml
```

Open the frontend -- see the error. 

## Figure out why we're getting a 500 err 


