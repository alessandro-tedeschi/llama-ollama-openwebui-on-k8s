# Deployment di Llama, Ollama e Open WebUI su cluster Kubernetes
## Descrizione del progetto
Il progetto consiste nel deployment di **Llama-3.2-1B**, **Ollama** e **Open WebUI** su un cluster Kubernetes.  
1. **Llama-3.2-1B** è un Large Language Model rilasciato da Meta con licenza Source-available. Si tratta di un modello leggero, con circa 1 miliardo di parametri.
2. **Ollama** è un framework per l'esecuzione di large language model. Reso accessibile attraverso un servizio Kubernetes, rappresenta il backend della nostra applicazione. Espone delle API REST attraverso cui è possibile scaricare modelli e interrogarli.
3. **Open WebUI** è un'interfaccia AI self-hosted. Accessibile anch'essa attraveso un servizio Kubernetes, rappresenta il frontend dell'applicazione, interagendo con il backend (**Ollama**) attraverso chiamate HTTP.

---

## Architettura del cluster

Il cluster Kubernetes è costituito da due macchine virtuali su cui è installato LUbuntu. 
Le due macchine virtuali sono connesse ad una rete con NAT gestita da VirtualBox. 

Per quanto riguarda i nodi Kubernetes, una VM fa da master e l'altra da worker:
* **master** (`192.168.43.10`)
* **worker** (`192.168.43.11`)

![Figura 1 – Architettura del cluster Kubernetes](img/cluster-architettura.jpg)

---

## 1. Deployment di Ollama (backend)

Creazione del **Namespace** in cui definiremo tutte le risorse relative a questo progetto:

`ollama/ollama_ns.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ollama
```
Eseguire il comando
```bash
kubectl apply -f ollama_ns.yaml
```

Creazione di **PersistentVolume** e **PersistentVolumeClaim** per la persistenza dei dati , tra cui i modelli stessi.

`ollama/ollama_pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ollama-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/ollama
    type: DirectoryOrCreate
  claimRef:
    namespace: ollama
    name: ollama-pvc  
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-pvc
  namespace: ollama
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```
Eseguire il comando
```bash
kubectl apply -f ollama_pvc.yaml
```

Creazione del **Deployment** e del **Service** per Ollama.

`ollama/ollama_deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: ollama
  labels:
    app: ollama
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
      - name: ollama
        image: ollama/ollama:latest
        ports:
        - containerPort: 11434
        resources:
          requests:
            cpu: 1000m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 3Gi
        volumeMounts:
        - name: models
          mountPath: /root/.ollama
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: ollama-pvc
```

`ollama/ollama_service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: ollama
spec:
  type: NodePort
  selector:
    app: ollama
  ports:
  - port: 11434            
    nodePort: 31434
```

Eseguire i comandi:
```bash
kubectl apply -f ollama_deploy.yaml
```
```bash
kubectl apply -f ollama_service.yaml
```

Per testare:
```bash
kubectl get pods -n ollama
```
```bash
kubectl get svc -n ollama
```

---

## 3. Deployment di Llama-3.2-1B

Sfruttiamo le API di Ollama per scaricare il modello Llama-3.2-1B. Lo facciamo tramite un Job:

`ollama/ollama_load-model-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: load-model
  namespace: ollama
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: curl
        image: curlimages/curl
        command: ["/bin/sh", "-c"]
        args:
          - |
            until curl -sf http://ollama:11434; do
              echo "Waiting for ollama..."; sleep 2;
            done &&
            curl -X POST http://ollama:11434/api/pull \
              -H 'Content-Type: application/json' \
              -d '{"name": "llama3.2:1b"}'
```
Eseguire il comando:
```bash
kubectl apply -f ollama_load-model-job.yaml
```
Per testare:
```bash
kubectl get pods -n ollama
```
```bash
kubectl get svc -n ollama
```

A questo punto, è già possibile interrogare il modello via cURL dal terminale di uno dei nodi, sfruttando le API REST esposte da Ollama.

La chiamata può essere fatta a IP e porta del nodo:
```bash
curl http://<NodeIP>:31434/api/generate -d '{
  "model": "llama3.2:1b",
  "prompt": "explain briefly why the sky is blue.",
  "stream": false
}'
```
Nel nostro cluster, gli indirizzi IP dei nodi sono `192.168.43.10` e `192.168.43.11`.
In alternativa, si può usare ClusterIP e porta del servizio.
```bash
curl http://<ClusterIP>:11434/api/generate -d '{
  "model": "llama3.2:1b",
  "prompt": "explain briefly why the sky is blue.",
  "stream": false
}'
```
dove il ClusterIP è visualizzabile mediante:
```bash
kubectl get svc -n ollama
```

---

## 4. Open WebUI

Creazione di **PV** e **PVC** necessari per la persistenza di conversazioni e credenziali. Nel nodo worker deve essere presente la directory /mnt/data/openwebui:

`openwebui/openwebui_pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: openwebui-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/openwebui
    type: DirectoryOrCreate
  claimRef:
    namespace: ollama
    name: openwebui-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openwebui-pvc
  namespace: ollama
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi                        
```
Eseguire il comando:
```bash
kubectl apply -f openwebui_pvc.yaml
```
Creazione di **Deployment** e **Service** (di tipo **LoadBalancer**) per Open WebUI. Come LoadBalancer è stato utilizzato MetalLB.

`openwebui/openwebui_deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-webui
  namespace: ollama
  labels:
    app: open-webui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: open-webui
  template:
    metadata:
      labels:
        app: open-webui
    spec:
      containers:
      - name: open-webui
        image: ghcr.io/open-webui/open-webui:main
        ports:
        - containerPort: 8080
        env:
        - name: OLLAMA_BASE_URL
          value: "http://ollama:11434"
        volumeMounts:
          - name: webui-data
            mountPath: /app/backend/data
        resources:
          requests:
            cpu: 1000m
            memory: 128Mi
          limits: 
            cpu: 2000m
            memory: 1Gi
      volumes:
      - name: webui-data
        persistentVolumeClaim:
          claimName: openwebui-pvc
```

`openwebui/openwebui_service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: open-webui
  namespace: ollama
spec:
  type: LoadBalancer
  selector:
    app: open-webui
  ports:
  - port: 8080
    nodePort: 31808
```

Eseguire i comandi:

```bash
kubectl apply -f openwebui_deploy.yaml
```

```bash
kubectl apply -f openwebui_service.yaml
```
Per testare:
```bash
kubectl get pods -n ollama
```
```bash
kubectl get svc -n ollama
```

---

Ora è possibile accedere al servizio Open WebUI tramite browser da uno dei due nodi all’URL:
```
http://<ExternalIP>:<8080>
```
dove l'indirizzo IP esterno può essere visualizzato attraverso:
```bash
kubectl get svc -n ollama
```
oppure
```bash
kubectl describe svc -n ollama open-webui
```

---

## 5. Test dell'interfaccia Open WebUI
