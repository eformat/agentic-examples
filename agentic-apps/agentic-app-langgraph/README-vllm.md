## VLLM deployment

Here we describe details of deploying vllm for agentic workflow. It is essential to deploy vLLM with tool calling supports (see details [here](https://docs.vllm.ai/en/stable/features/tool_calling.html)).

`Llama` models has following issues with tool calling:
- Parallel tool calls are not supported.

- The model can generate parameters with a wrong format, such as generating an array serialized as string instead of an array.

Similarly, `Mistral` has its own set of [challenges](https://docs.vllm.ai/en/stable/features/tool_calling.html#mistral-models-mistral).

For this demo, we deploy using `IBM granite` LLM model which has effective support for tool calling.

The example below list changes required to deploy `granite-3.2-8b-instruct` with tool calling support using Intel Gaudi. Similar changes should be applied if vLLM deployed on CPU or GPU. Here, the changes is made on the example of llm-on-openshift [repo](https://github.com/rh-aiservices-bu/llm-on-openshift/tree/main/llm-servers/vllm/hpu/gitops).

Here, we need to modify two gitops files and add one required chat template file:

<details>
<summary> edit kustomization.yaml </summary>

    
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

commonLabels:
  component: vllm

resources:
# wave 0
- pvc.yaml
# wave 1
- deployment.yaml
- service.yaml

configMapGenerator:
  - name: vllm-chat-template
    files:
      - tool_chat_template_granite.jinja
```
</details>

<details>
<summary> edit deployment.yaml</summary>

    
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
  labels:
    app: vllm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm
  template:
    metadata:
      labels:
        app: vllm
    spec:
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 120
      containers:
        - name: server
          image: intel/redhat-ai-services:llm-on-openshift_ubi9.4_1.20.0
          imagePullPolicy: Always
          args:
            - "--model=ibm-granite/granite-3.2-8b-instruct"
            - "--download-dir"
            - "/models-cache"           
            - "--device"
            - "hpu"
            - "--tensor-parallel-size"
            - "1"
            - "--pipeline-parallel-size"
            - "1"
            - "--dtype"
            - "float16"
            - "--enable-auto-tool-choice"
            - "--tool-call-parser"
            - "granite"
            - "--chat-template"
            - "/app/tool_chat_template_granite.jinja"
          ports:
            - name: http
              containerPort: 8000
              protocol: TCP
          env:
            - name: HUGGING_FACE_HUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hf-token
                  key: HF_TOKEN
            - name: HABANA_VISIBLE_DEVICES
              value: "all"
            - name: OMPI_MCA_btl_vader_single_copy_mechanism
              value: "none"
            - name: PT_HPU_ENABLE_LAZY_COLLECTIVES
              value: "true"
            - name: PT_HPU_LAZY_ACC_PAR_MODE
              value: "0"
            - name: VLLM_SKIP_WARMUP
              value: "true"
          resources:
            limits:
              cpu: "32"
              memory: 55Gi
              habana.ai/gaudi: 1
              hugepages-2Mi: 8000Mi
            requests:
              cpu: "32"
              memory: 50Gi
              habana.ai/gaudi: 1
              hugepages-2Mi: 8000Mi
          securityContext:
            capabilities:
              drop:
                - ALL
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            seccompProfile:
              type: RuntimeDefault
          readinessProbe:
            httpGet:
              path: /health
              port: http
              scheme: HTTP
            timeoutSeconds: 5
            periodSeconds: 30
            successThreshold: 1
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: http
              scheme: HTTP
            timeoutSeconds: 8
            periodSeconds: 100
            successThreshold: 1
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health
              port: http
              scheme: HTTP
            timeoutSeconds: 1
            periodSeconds: 30
            successThreshold: 1
            failureThreshold: 24
          volumeMounts:
            - name: models-cache
              mountPath: /models-cache
            - name: shm
              mountPath: /dev/shm
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /.cache
            - name: config
              mountPath: /.config
            - name: chat-template-volume
              mountPath: /app/tool_chat_template_granite.jinja
              subPath: tool_chat_template_granite.jinja              
      volumes:
        - name: models-cache
          persistentVolumeClaim:
            claimName: vllm-models-cache
        - name: shm
          emptyDir:
            medium: Memory
            sizeLimit: 12Gi
        - name: tmp
          emptyDir: {}
        - name: cache
          emptyDir: {}
        - name: config
          emptyDir: {}
        - name: chat-template-volume
          configMap:
            name: vllm-chat-template          
      dnsPolicy: ClusterFirst
      tolerations:
        - key: habana.ai/gaudi
          operator: Exists
          effect: NoSchedule
  strategy:
    type: Recreate
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
```
</details>

<details>
<summary>new file: tool_chat_template_granite.jinja </summary>

    
```jinja
{%- if tools %}
    {{- '<|start_of_role|>available_tools<|end_of_role|>\n' }}
    {%- for tool in tools %}
    {{- tool | tojson(indent=4) }}
    {%- if not loop.last %}
        {{- '\n\n' }}
    {%- endif %}
    {%- endfor %}
    {{- '<|end_of_text|>\n' }}
{%- endif %}

{%- for message in messages %}
    {%- if message['role'] == 'system' %}
    {{- '<|start_of_role|>system<|end_of_role|>' + message['content'] + '<|end_of_text|>\n' }}
    {%- elif message['role'] == 'user' %}
    {{- '<|start_of_role|>user<|end_of_role|>' + message['content'] + '<|end_of_text|>\n' }}
    {%- elif message['role'] == 'assistant_tool_call' or (message['role'] == 'assistant' and message.tool_calls is defined) %}
    {{- '<|start_of_role|>assistant<|end_of_role|><|tool_call|>' + message.tool_calls | map(attribute='function') | list | tojson(indent=4) + '<|end_of_text|>\n' }}
    {%- elif message['role'] == 'assistant' %}
    {{- '<|start_of_role|>assistant<|end_of_role|>' + message['content'] + '<|end_of_text|>\n' }}
    {%- elif message['role'] == 'tool_response' or message['role'] == 'tool' %}
    {{- '<|start_of_role|>tool_response<|end_of_role|>' + message['content'] + '<|end_of_text|>\n' }}
    {%- endif %}
    {%- if loop.last and add_generation_prompt %}
    {{- '<|start_of_role|>assistant<|end_of_role|>' }}
    {%- endif %}
{%- endfor %}
```
</details>