# Deployment di Llama, Ollama e Open WebUI su Kubernetes
## Descrizione del progetto
Il progetto consiste nel deployment di **Llama3.2:1b**, **Ollama** e **Open WebUI** su un cluster Kubernetes.  
1. **Llama3.2:1b** è un Large Language Model rilasciato da Meta con licenza Source-available. Si tratta di un modello leggero, con 1 miliardo di parametri.
2. **Ollama** è un framework per l'esecuzione di large language model. Reso accessibile attraverso un servizio Kubernetes, rappresenta il backend dell'applicazione. Ollama esponde delle API REST attraverso cui è possibile scaricare modelli e interrogarli.
3. **Open WebUI** è un'interfaccia AI self-hosted. Accessibile anch'essa attraveso un servizio Kubernetes, rappresenta il frontend dell'applicazione, interagendo con il backend (Ollama) attraverso chiamate HTTP.

---

## Architettura del cluster

Il cluster Kubernetes è costituito da due macchine virtuali su cui è installato Xubuntu 24. 
Le due macchine virtuali sono connesse ad una rete con NAT gestita da VirtualBox. 

Per quanto riguarda i nodi Kubernetes, una VM ospita il nodo master e l’altra il nodo worker:
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
#persistent volume
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
  claimRef:
    namespace: ollama
    name: ollama-pvc  
---
#persistent volume claim
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

## 3. Deployment di Llama3.2:1b

Sfruttiamo le API di Ollama per scaricare il modello Llama3.2:1b. Lo facciamo tramite un Job:

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

---

A questo punto, è già possibile interrogare il modello via cURL, sfruttando le API REST esposte da Ollama.

La chiamata ad Ollama può essere fatta a IP ClusterIP e porta ServicePort:
```bash
curl http://<ClusterIP>:<ServicePort>/api/generate -d '{
  "model": "llama3.2:1b",
  "prompt": "explain briefly why the sky is blue.",
  "stream": false
}'
```
dove ServicePort ha valore 11434 se il file `ollama/ollama_service.yaml` non viene modificato, e ClusterIP può essere visulizzato mediante 
```bash
kubectl get kubectl get svc -n ollama
```
oppure
```bash
kubectl describe svc -n ollama ollama
```

In alternativa, contattiamo Ollama a IP NodeIP e porta NodePort, sfruttando il fatto il servizio è di tipo NodePort:

```bash
curl http://<NodeIP>:<NodePort>/api/generate -d '{
  "model": "llama3.2:1b",
  "prompt": "explain briefly why the sky is blue.",
  "stream": false
}'
```

dove NodeIP nel nostro cluster può assumere valori 192.168.43.10 o 192.168.43.11, e NodePort assume valore 31434 se il file `ollama/ollama_service.yaml` non viene modificato

---

## 4. Open WebUI

Creazione di **PV** e **PVC** necessari per la persistenza di conversazioni e credenziali:

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
Creazione di **Deployment** e **Service** (di tipo **LoadBalancer**) per Open WebUI:

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
        image: ghcr.io/open-webui/open-webui:latest
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
Esponi Open WebUI con un servizio di tipo **LoadBalancer**:

```bash
kubectl apply -f openwebui_service.yaml
```

Verifica le informazioni del servizio:
```bash
kubectl get svc -n ollama
kubectl describe svc -n ollama open-webui
```

---


Una volta creato il servizio, accedi a Open WebUI tramite browser all’indirizzo:

```
http://<ExternalIP>:<ServicePort>
```

Esempio:
```
http://192.168.1.240:8080
```

La variabile di ambiente definita in `openwebui_deploy.yaml` permette a Open WebUI di comunicare con Ollama.  
In questo modo, l’interfaccia web può vedere i modelli scaricati e inviare richieste di inferenza.  

---
