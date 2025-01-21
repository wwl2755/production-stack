# LMStack


This project provides a reference implementation on how to build an inference stack on top of vLLM, including the following components:

- **Helm Chart**: Deploys vLLM services in Kubernetes environments.
- **Grafana Dashboard**: Visualizes key LLM serving metrics for monitoring.

Inside the stack, there are the following key parts:
- **Backend**: The vLLM engines that runs different LLMs
- **Router**: Directs requests to appropriate backends based on routing keys or session IDs to maximize KV cache reuse.

 <img src="https://github.com/user-attachments/assets/ffbdb2de-0dce-46cf-bc07-c4057b35ad7f" alt="Architecture of the stack" width="800"/>

## Helm Chart

### Prerequisites

- A running Kubernetes (K8s) environment with GPUs
  - Run `cd utils && bash install-minikube-cluster.sh`
  - Or: [tutorial](https://minikube.sigs.k8s.io/docs/tutorials/nvidia/)


### Deployment

1. Store your custom configurations in `values-customized.yaml`. Example:
```yaml
servingEngineSpec:
  modelSpec:
  - name: "opt125m"
    repository: "lmcache/vllm-openai"
    tag: "latest"
    modelURL: "facebook/opt-125m"

    replicaCount: 1

    requestCPU: 6
    requestMemory: "16Gi"
    requestGPU: 1

    pvcStorage: "10Gi"
```

3. Deploy the Helm chart:

```bash
sudo helm repo add lmstack-repo https://lmcache.github.io/helm/
sudo helm install lmstack lmstack-repo/lmstack -f values-customized.yaml
```

### Validate the installation

**Monitor the status of the deployment**: 
The following command can be used to monitor the deployment
```bash
sudo kubectl get all
```

You should be able to see outputs like this:
```
NAME                                                  READY   STATUS    RESTARTS   AGE
pod/lmstack-deployment-router-859d8fb668-2x2b7        1/1     Running   0          2m38s
pod/lmstack-opt125m-deployment-vllm-84dfc9bd7-vb9bs   0/1     Running   0          2m38s

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kubernetes               ClusterIP   10.96.0.1       <none>        443/TCP   35d
...
```

Now please wait until the second role to turn into 
```
NAME                                                  READY   STATUS    RESTARTS   AGE
pod/lmstack-opt125m-deployment-vllm-84dfc9bd7-vb9bs   **1**/1     Running   0          2m38s
```

_Note_: it takes some time to download the docker images and the LLM weights. You might need to wait for it to be ready.

**Send a query to the stack**:

First, forward the service port to the host machine (port 30080):
```bash
sudo kubectl port-forward svc/lmstack-router-service 30080:80
```

Then, open a new terminal, and curl the endpoint on the host machine
```bash
# curl the endpoint
curl -o- http://localhost:30080/completions \
-H "content-type: application/json" \
-d '{"model": "facebook/opt-125m", "prompt": "Write a simple poem: ", "stream":false, "max_tokens": 30}'
```

You should see outputs like
```text
{"id":"cmpl-f6527a57403849518dfe793126d59b3e",
"object":"text_completion",
"created":1737411036,
"model":"facebook/opt-125m",
"choices":[
  {"index":0,"text":"  \"If she's not here, you're dead.\"\nlol. If you do troll the world it gets annoying pretty fast.\n...next",
   "logprobs":null,
   "finish_reason":"length",
   "stop_reason":null,
   "prompt_logprobs":null}],
   "usage":{"prompt_tokens":7,"total_tokens":37,"completion_tokens":30,"prompt_tokens_details":null}}
```

#### Uninstall

```bash
sudo helm uninstall lmstack
```


## Grafana Dashboard

### Features

The Grafana dashboard provides the following insights:


1. **Available vLLM Instances**: Displays the number of healthy instances.
2. **Request Latency Distribution**: Visualizes end-to-end request latency.
3. **Time-to-First-Token (TTFT) Distribution**: Monitors response times for token generation.
4. **Number of Running Requests**: Tracks the number of active requests per instance.
5. **Number of Pending Requests**: Tracks requests waiting to be processed.
6. **GPU KV Usage Percent**: Monitors GPU KV cache usage.
7. **GPU KV Cache Hit Rate**: Displays the hit rate for the GPU KV cache.

 <img src="https://github.com/user-attachments/assets/225feb01-ac0f-4bf9-9da3-7bf955b2aa56" alt="Grafana dashboard to monitor the deployment" width="500"/>

### Configuration

See the details in `observability/README.md`

## Router

### Overview

The router ensures efficient request distribution among backends. It supports:

- Routing to endpoints that run different models
- Exporting observability metrics for each serving engine instance, including QPS, time-to-first-token (TTFT), number of pending/running/finished requests, and uptime
- Automatic service discovery and fault tolerance by Kubernetes API
- Multiple different routing algorithms
  - Round-robin routing
  - Session-ID based routing
  - (WIP) prefix-aware routing


## Contributing

Contributions are welcome! Please follow the standard GitHub flow:

1. Fork the repository.
2. Create a feature branch.
3. Submit a pull request with detailed descriptions.

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.

---

For any issues or questions, feel free to open an issue or contact the maintainers.

