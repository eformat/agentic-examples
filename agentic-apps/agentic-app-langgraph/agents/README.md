## Agentic App Backend

Example of App with Agentic AI included.

### Deploy in K8s / OpenShift

```bash
kubectl apply -k templates/
```

> IMPORTANT: you need to update the secrets in `templates/secret-agentic-app-be.yaml`

### Usage

```bash
curl -X POST "http://localhost:8080/ask"      -H "Content-Type: application/json"      -d '{"query": "What is KServe"}'

{"response":"What is KServe\n<tool_call>\nPage: Kubeflow\nSummary: Kubeflow is ...."}
```

### Develop locally

* Install the dependencies using UV:

```bash
uv venv
source .venv/bin/activate
uv pip install -r requirements.txt
```

* Deploy the Agent/Backend:

```bash
source .venv/bin/activate
export MODEL_NAME="bartowski/granite-3.1-8b-instruct-GGUF"
export API_KEY="None"
export API_URL="http://localhost:57364"
python agents/app.py
```