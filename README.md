# Deployment di Ollama + Open WebUI su Kubernetes
## Descrizione del progetto
Il progetto consiste nel deployment di **Llama3.2:1b**, **Ollama** e **Open WebUI** su un cluster Kubernetes.  
1. **Llama3.2:1b** è un Large Language Model rilasciato da Meta con licenza Source-available. Si tratta di un modello leggero, con 1 miliardo di parametri.
2. **Ollama** è un framework per l'esecuzione di large language model. Reso accessibile attraverso un servizio kubernetes, rappresenta il backend dell'applicazione. Ollama esponde delle API REST attraverso cui è possibile scaricare modelli e interrogarli.
3. **Open WebUI** è un'interfaccia AI self-hosted. Accessiile anche essa attraveso un servizio kubernetes, rappresenta il frontend dell'applicazione, interagendo con il backend (Ollama) attraverso chiamate HTTP.

---

## Architettura del cluster

Il cluster Kubernetes è costituito da due macchine virtuali su cui è installato Xubuntu 24. 
Le due macchine virtuali sono connesse ad una rete con NAT gestita da VirtualBox. 

Per quanto riguarda i nodi Kubernetes, una VM ospita il nodo master e l’altra il nodo worker:
* **master** (`192.168.43.10`)
* **worker** (`192.168.43.11`)

![Figura 1 – Architettura del cluster Kubernetes](img/cluster-architettura.jpg)

## 1. Creazione namespace e PVC

Creazione del namespace `ollama_ns`:
```bash
kubectl apply -f ollama_ns.yaml
```

Creazione dei **PV/PVC**:
```bash
kubectl apply -f ollama_pvc.yaml
```

Verifica che il PVC sia **bound**:
```bash
kubectl get pvc -n ollama
```

---

## 1. Deployment di Ollama

Creazione del namespace "ollama" 
`ollama/ollama_ns.yaml`:

```yaml



Esegui il deployment di Ollama:
```bash
kubectl apply -f ollama_deploy.yaml
```
*(potrebbe richiedere qualche minuto)*

Verifica che il pod sia stato creato:
```bash
kubectl get pods -n ollama
```

Crea un servizio di tipo **NodePort** che espone Ollama:
```bash
kubectl apply -f ollama_service.yaml
```

Verifica la presenza del servizio:
```bash
kubectl get svc -n ollama
kubectl describe svc -n ollama ollama
```

---

## 3. Deployment di Llama3.2:1b

Per ora, Ollama non ha nessun modello.  
Scarichiamo il modello **llama3.2:1b** (leggero, ~1B parametri) tramite un job:
```bash
kubectl apply -f ollama_load-model-job.yaml
```

Verifica che il job sia terminato (modello scaricato):
```bash
kubectl get pods -n ollama
```

---

## 4. Interrogare il modello via cURL

Puoi interrogare Ollama tramite `ClusterIP` e `ServicePort`:
```bash
curl http://<ClusterIP>:<ServicePort>/api/generate -d '{
  "model": "llama3.2:1b",
  "prompt": "explain briefly why the sky is blue.",
  "stream": false
}'
```

Oppure tramite `NodeIP` e `NodePort`:
```bash
curl http://192.168.43.11:31434/api/generate -d '{
  "model": "llama3.2:1b",
  "prompt": "explain briefly why the sky is blue.",
  "stream": false
}'

curl http://192.168.43.10:31434/api/generate -d '{
  "model": "llama3.2:1b",
  "prompt": "explain briefly why the sky is blue.",
  "stream": false
}'
```

---

## 5. Deployment di Open WebUI

Creiamo i PV/PVC necessari per la memorizzazione di conversazioni e credenziali:
```bash
kubectl apply -f openwebui_pvc.yaml
```

Verifica che siano bound:
```bash
kubectl get pvc -n ollama
```

Deployment di Open WebUI:
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

## 6. Accesso a Open WebUI

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
