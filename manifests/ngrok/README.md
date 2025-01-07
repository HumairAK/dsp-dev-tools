
### Steps

```bash
cd manifests/ngrok
```

Get NGROK auth token from: https://dashboard.ngrok.com/get-started/your-authtoken
Start ngrok:

```bash
YOUR_AUTHTOKEN=...
var=$YOUR_AUTHTOKEN yq -i '.authtoken = env(var)' ngrok.yml

ngrok start --all --config ngrok.yml
```

Retrieve the values from the ngrok view with those in nginx deployment: 

For example: 

```txt
tcp://4.tcp.ngrok.io:19033 -> localhost:8888   # maps to KFP_NGROK_HTTP_HOSTNAME & KFP_NGROK_HTTP_PORT                                                                                                                                                
tcp://8.tcp.ngrok.io:19559 -> localhost:8887   # maps to KFP_NGROK_GRPC_HOSTNAME & KFP_NGROK_GRPC_PORT      
```

So given the output above the yaml should look like this: 

```yaml
env:
- name: KFP_NGROK_GRPC_HOSTNAME
  value: "8.tcp.ngrok.io"
- name: KFP_NGROK_GRPC_PORT
  value: "19559"
- name: KFP_NGROK_HTTP_HOSTNAME
  value: "4.tcp.ngrok.io"
- name: KFP_NGROK_HTTP_PORT
  value: "19033"
```

Deploy nginx to mapp these to a service: 

```bash
cd manifests/ngrok/nginx
namespace=kubeflow
kustomize build . | oc -n $namespace apply -f - 

# If nginx was already running
oc -n $namespace rollout restart deployment/grpc-nginx
```

### Troubleshooting

```
failure while getting executionCache: 
failed to list tasks: 
rpc error: code = Unavailable desc = connection error: desc = "transport: 
Error while dialing: dial tcp [::1]:8887: connect: connection refused"
```

* Make sure the mappings for port/host are accurate
* If using the updated driver image where we set ML PIPELINE HOST, make sure it's set to the service (and not localhost) when starting api server