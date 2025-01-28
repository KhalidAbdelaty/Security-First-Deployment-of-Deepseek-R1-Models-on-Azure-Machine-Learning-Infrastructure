### Security-First Deployment of Deepseek R1 Models on Azure Machine Learning Infrastructure 
*Security-first deployment blueprint for Deepseek R1 on Azure ML, ensuring regional data sovereignty, infrastructure autonomy, and scalable AI inference.* 

#### Overview  
This guide walks you through securely deploying the Deepseek R1 model on Azure Machine Learning using vLLMs and Managed Online Endpoints. By running the model within your Azure subscription, you gain full control over your data while maintaining flexibility in choosing the deployment region, such as `eastus2` or `swedencentral`.

The deployment focuses on **DeepSeek-R1-Distill-Llama-8B**, an optimized model derived from **Llama-3.1-8B**, ensuring high performance and scalability for real-time inference.

---

### Tools and Technologies  
To implement the deployment, we will utilize the following components:  

- **vLLM**: A high-performance inference engine tailored for large language models.  
- **Azure Machine Learning’s Managed Online Endpoints**: A scalable solution for deploying and managing machine learning models.  

#### What is vLLM?  
vLLM enhances the efficiency of large language model deployments through cutting-edge memory management techniques, such as **PagedAttention**. These innovations enable:  
- Continuous batching of requests for smoother execution.  
- Distributed inference with support for tensor and pipeline parallelism.  
- Compatibility with Hugging Face models and advanced decoding algorithms like beam search.  

#### Managed Online Endpoints on AzureML  
Managed Online Endpoints simplify real-time model serving by handling infrastructure tasks, including scaling, monitoring, and security. This allows developers to focus on improving their models rather than worrying about backend management.

---

### Deploying Deepseek R1  

#### Step 1: Setting Up the Environment  
To prepare the environment, we’ll define a custom Dockerfile and build it as an AzureML environment. Here’s the Dockerfile configuration:  

```dockerfile
FROM vllm/vllm-openai:latest  
ENV MODEL_NAME=deepseek-ai/DeepSeek-R1-Distill-Llama-8B  
ENTRYPOINT python3 -m vllm.entrypoints.openai.api_server --model $MODEL_NAME $VLLM_ARGS  
```  

This setup allows you to specify the model during deployment by simply adjusting the `MODEL_NAME` variable.  

Next, log in to your Azure workspace:  
```bash
az account set --subscription <subscription_id>  
az configure --defaults workspace=<workspace_name> group=<resource_group>  
```  

Now, create the `environment.yml` file to define the environment’s build configuration:  

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/environment.schema.json  
name: r1  
build:  
  path: .  
  dockerfile_path: Dockerfile  
```  

Finally, build the environment:  
```bash
az ml environment create -f environment.yml  
```  

---

#### Step 2: Deploying the Model  
To deploy the model, follow these steps:  

1. **Define the Endpoint**:  
   Create `endpoint.yml` to specify the Managed Online Endpoint:  
   ```yaml
   $schema: https://azuremlsdk2.blob.core.windows.net/latest/managedOnlineEndpoint.schema.json  
   name: r1-prod  
   auth_mode: key  
   ```  
   Deploy the endpoint:  
   ```bash
   az ml online-endpoint create -f endpoint.yml  
   ```  

![image](https://github.com/user-attachments/assets/a54d7666-be47-4399-b1a5-c4be34dde1b8)


2. **Configure the Deployment**:  
   Use a `deployment.yml` file to define the deployment settings, including the model, Docker image, and hardware requirements:  

   ```yaml
   $schema: https://azuremlschemas.azureedge.net/latest/managedOnlineDeployment.schema.json  
   name: current  
   endpoint_name: r1-prod  
   environment_variables:  
     MODEL_NAME: deepseek-ai/DeepSeek-R1-Distill-Llama-8B  
     VLLM_ARGS: "--max-num-seqs 16 --enforce-eager"  
   environment:  
     image: <docker_image_address>  
     inference_config:  
       liveness_route:  
         port: 8000  
         path: /health  
       readiness_route:  
         port: 8000  
         path: /health  
       scoring_route:  
         port: 8000  
         path: /  
   instance_type: Standard_NC24ads_A100_v4  
   instance_count: 1  
   request_settings:  
       max_concurrent_requests_per_instance: 1  
       request_timeout_ms: 10000  
   liveness_probe:  
     initial_delay: 10  
     period: 10  
     timeout: 2  
     success_threshold: 1  
     failure_threshold: 30  
   readiness_probe:  
     initial_delay: 120  
     period: 10  
     timeout: 2  
     success_threshold: 1  
     failure_threshold: 30  
   ```  

   Deploy the model and route all traffic to it:  
   ```bash
   az ml online-deployment create -f deployment.yml --all-traffic  
   ```  

---

#### Testing the Deployment  
To confirm the deployment, retrieve the endpoint’s URI and API key:  
```bash
az ml online-endpoint show -n r1-prod  
az ml online-endpoint get-credentials -n r1-prod  
```  

Use the following Python snippet to send test requests:  

```python
import requests  

url = "<endpoint_uri>"  
headers = {  
    "Content-Type": "application/json",  
    "Authorization": "Bearer <api_key>"  
}  
data = {  
    "model": "deepseek-ai/DeepSeek-R1-Distill-Llama-8B",  
    "messages": [{"role": "user", "content": "Which is better, summer or winter?"}],  
    "max_tokens": 750  
}  

response = requests.post(url, headers=headers, json=data)  
print(response.json())  
```  

---

### Autoscaling the Deployment  
Azure Machine Learning offers autoscaling capabilities to dynamically adjust resources based on workload. By integrating with Azure Monitor, you can set up autoscaling rules, such as increasing instances when GPU utilization exceeds a threshold or scaling down during low traffic periods.  

---

### Conclusion  
This guide demonstrated how to deploy the Deepseek R1 model on AzureML using Managed Online Endpoints. By leveraging this approach, you gain control over your data and the flexibility to choose deployment regions, ensuring a secure and efficient real-time inference solution.  
