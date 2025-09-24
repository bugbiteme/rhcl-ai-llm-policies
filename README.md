# rhcl-ai-llm-policies
Red Hat Connectivity Link Token-based Rate Limiting for Large Language Model APIs

Assumptions: You have all the RHCL operators installed and are running a Kuadrant System

Kuadrant `TokenRateLimitPolicy` tutorial blog (`dev`):
- https://docs.kuadrant.io/dev/kuadrant-operator/doc/user-guides/tokenratelimitpolicy/authenticated-token-ratelimiting-tutorial/

llm simulator image used in this tutorial:
- https://github.com/llm-d/llm-d-inference-sim/pkgs/container/llm-d-inference-sim

```sh
export KUADRANT_GATEWAY_NS=gateway-system
export KUADRANT_GATEWAY_NAME=trlp-tutorial-gateway
export KUADRANT_SYSTEM_NS=$(kubectl get kuadrant -A -o jsonpath='{.items[0].metadata.namespace}')
```
## Create Gateway (Gateway API):

- Note you will need to update the `hostname` in `gateway.yaml` first
```sh
oc apply -k gateway  
```

Deploy llm-sim:
- Note you will need to update the `hostname(s)` in `httproute.yaml` to match `gateway.yaml` first
```sh
oc apply -k llm-sim     
```

## Export the gateway URL for use in requests:

```sh
export KUADRANT_INGRESS_HOST=$(kubectl get gtw ${KUADRANT_GATEWAY_NAME} -n ${KUADRANT_GATEWAY_NS} -o jsonpath='{.status.addresses[0].value}')
export KUADRANT_INGRESS_PORT=$(kubectl get gtw ${KUADRANT_GATEWAY_NAME} -n ${KUADRANT_GATEWAY_NS} -o jsonpath='{.spec.listeners[?(@.name=="http")].port}')
export KUADRANT_GATEWAY_URL=${KUADRANT_INGRESS_HOST}:${KUADRANT_INGRESS_PORT}
export LLM_HOSTNAME=$(oc -n llm-sim get httproute trlp-tutorial-llm-sim -o jsonpath='{.spec.hostnames[0]}{"\n"}')
```

## Test connection:

```sh
curl -H "Host: ${LLM_HOSTNAME}" http://$KUADRANT_GATEWAY_URL/v1/models -i
```

Expected output:
```json
HTTP/1.1 200 OK
server: istio-envoy
date: Tue, 23 Sep 2025 22:35:56 GMT
content-type: application/json
content-length: 180
x-envoy-upstream-service-time: 0

{"object":"list","data":[{"id":"meta-llama/Llama-3.1-8B-Instruct","object":"model","created":1758666957,"owned_by":"vllm","root":"meta-llama/Llama-3.1-8B-Instruct","parent":null}]}
```

Response without the `-i` flag:
```sh
curl -H "Host: ${LLM_HOSTNAME}" http://$KUADRANT_GATEWAY_URL/v1/models | jq
```

```json
{
  "object": "list",
  "data": [
    {
      "id": "meta-llama/Llama-3.1-8B-Instruct",
      "object": "model",
      "created": 1758667109,
      "owned_by": "vllm",
      "root": "meta-llama/Llama-3.1-8B-Instruct",
      "parent": null
    }
  ]
}
```

## Lets test the llm with a prompt!
```sh
curl -H "Host: ${LLM_HOSTNAME}" \
     -H "Authorization: APIKEY iamafreeuser" \
     -H "Content-Type: application/json" \
     -X POST http://$KUADRANT_GATEWAY_URL/v1/chat/completions \
     -d '{
           "model": "meta-llama/Llama-3.1-8B-Instruct",
           "messages": [
             { "role": "user", "content": "What is OpenShift?" }
           ],
           "max_tokens": 100,
           "stream": false,
           "usage": true
         }' | jq
```

Example response (keep in mind that these are probably hard-coded per `llm-sim`)
```json
{
  "id": "chatcmpl-c3be7a21-abc9-43a9-a226-dde07fad525d",
  "created": 1758667641,
  "model": "meta-llama/Llama-3.1-8B-Instruct",
  "usage": {
    "prompt_tokens": 4,
    "completion_tokens": 100,
    "total_tokens": 104
  },
  "object": "chat.completion",
  "do_remote_decode": false,
  "do_remote_prefill": false,
  "remote_block_ids": null,
  "remote_engine_id": "",
  "remote_host": "",
  "remote_port": 0,
  "choices": [
    {
      "index": 0,
      "finish_reason": "length",
      "message": {
        "role": "assistant",
        "content": "The temperature here is twenty-five degrees centigrade. The temperature here is twenty-five degrees centigrade. Testing@, #testing 1$ ,2%,3^, [4&*5], 6~, 7-_ + (8 : 9) / \\ < > . The rest is silence.  Give a man a fish and you feed him for a day; teach a man to fish and you feed him for a lifetime Alas, poor Yorick! I knew him, Horatio: A "
      }
    }
  ]
}
```

## Applying RHCL Policies

For the purpose of this demo, rather than applying all the policies at once, let's apply resources one at a time

### Create Secrets
These are our moc users and API keys that give them different levels of service to our LLM

```yaml
api_key: iamafreeuser
api_key: iamagolduser
```

```sh
oc apply -f rhcl/secrets.yaml -n kuadrant-system  
```

### Apply AuthPolicy 

You can follow Authorino Operator logs while testing

```sh
oc -n kuadrant-system logs deploy/authorino -f
```

The below willl be applied on the Gateway and will only allow users with the `APIKEY` containing one of the two set 
API Keys set in the secrets for `gold` and `free` user groups.

```sh
oc apply -f rhcl/authpolicy.yaml -n gateway-system 
```

Test Auth Policy

- Without APIKEY Headers

```sh
curl -v -H "Host: ${LLM_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -X POST "http://${KUADRANT_GATEWAY_URL}/v1/chat/completions" \
  -d '{"model":"meta-llama/Llama-3.1-8B-Instruct","messages":[{"role":"user","content":"What is OpenShift?"}],"max_tokens":100,"stream":false,"usage":true}'
```

Output:
```
...
< HTTP/1.1 401 Unauthorized
< www-authenticate: APIKEY realm="api-key-users"
< x-ext-auth-reason: credential not found
```

- Using APIKEY headers

```
-H "Authorization: APIKEY iamafreeuser"
```

```sh
curl -v -H "Host: ${LLM_HOSTNAME}" \
  -H "Authorization: APIKEY iamafreeuser" \
  -H "Content-Type: application/json" \
  -X POST "http://${KUADRANT_GATEWAY_URL}/v1/chat/completions" \
  -d '{"model":"meta-llama/Llama-3.1-8B-Instruct","messages":[{"role":"user","content":"What is OpenShift?"}],"max_tokens":100,"stream":false,"usage":true}'
```

Output:

```
...
< HTTP/1.1 200 OK
...
```

### Apply RateLimitPolicy

