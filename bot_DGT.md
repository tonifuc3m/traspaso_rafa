
# Bot DGT
**INTRODUCCIÓN:** Es muy parecido al bot de agencias, pero los documentos son otros. Las diferencias principales son:
- Documentos diferentes
- Código refactorizado
- Tiene un endpoint con filtro de preguntas no relevantes y otro sin el filtro
 
De hecho, aunque existe código de conector para el Bot DGT, no está desplegado y no se está usando. El mismo conector del Bot Agencias si le das `collection=DGT` manda las peticiones a este bot.

## Código
- [Repo en lhf-labs, branch bot_dgt](https://github.com/lhf-labs/rag_muface_dgt/tree/bot_dgt)
- [Zip en Gitlab](gitlab-ic.scae.redsara.es/060_lhf/bot_dgt)

### Resumen arquitectura 
Igual que bot agencias, pero con otro LLM extra para filtrar preguntas no relevantes.

En el archivo de configuración `src/config.json` se debe indicar la ruta a: la carpeta con los datos, el servicio llama3, el servicio SOLAR, servicio LLM generación de preguntas, servicio embeddings, y modelo reranker. Ejemplo de JSON:
```JSON
{
    "DATA_FILE_PATH": "/usr/src/app/src/data",
    "SERVER_LLAMA3": "http://10.118.187.160:8080",
    "SERVER_SOLAR": "http://10.118.187.161:8082", 
    "SERVER_GEN_QUESTIONS": "http://10.118.187.160:8000",
    "SERVER_EMBEDDINGS": "http://10.118.187.160:8001",
    "RERANKER_MODEL_PATH": "/data/models/rerankers/bge-reranker-large",
    "EMBEDDINGS_MODEL_PATH": "/data/models/embeddings/bge-m3",
    "OPENAI_API_KEY": "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "NUM_RETRIEVES_BM25": "10",
    "NUM_RETRIEVES_EMBEDDING": "10",
    "MAX_TOKENS_MAIN": "300",
    "LOGGING_MODE": "DEBUG"
}
```

### Despliegue
Casi igual que el bot agencias

**Preparación**:
1. Necesitas desplegar los LLMs y Text Embeddings Inference a los que apunta tu archivo de configuración.
```bash
# SOLAR con llamacpp
mkdir -p /data/models
cd /data/models
sudo wget https://huggingface.co/TheBloke/SOLAR-10.7B-Instruct-v1.0-GGUF/resolve/main/solar-10.7b-instruct-v1.0.Q8_0.gguf 
sudo docker run -v /mnt/sdc1/models:/MODELS -p 8082:8082 -e MODEL=solar-10.7b-instruct-v1.0.Q8_0.gguf -e PORT=8082 -e CTX=4096 --privileged --gpus all llamacpp_service # imagen llamacpp de Rafa

# Llama3 con vLLM 
cd /data/models 
git clone https://huggingface.co/meta-llama/Meta-Llama-3-8B-Instruct
cd /home/admin/
git clone https://github.com/lhf-labs/sgad.git # repo sgad del playground 
cd sgad/services/vLLM 
sudo ./deploy_vLLM.sh 0 /data/models/vLLM_models/ /root/.cache/huggingface 8000 vllm/vllm-openai:v0.4.2 NousResearch/Meta-Llama-3-8B-Instruct 

# TEI
mkdir –p /data/models/embeddings 
cd /data/models/embeddings 
git clone https://huggingface.co/BAAI/bge-m3 
docker run -d --privileged --gpus all -p 8001:80 -v /data/models/embeddings/bge-m3/:/model ghcr.io/huggingface/text-embeddings-inference:1.2 --model-id /model 
```

2. Necesitas descargarte el reranker y ponerlo en la ruta que indicas en el archivo de configuración
```bash
mkdir -p /data/models/rerankers 
cd /data/models/rerankers 
git clone https://huggingface.co/BAAI/bge-reranker-large 
```


**Despliegue**

1. Despliegue del RAG (docker): 
```bash

cd rag_muface_dgt

docker build -t bot_dgt -f core/Dockerfile .


docker run -d --restart unless-stopped -t --privileged --gpus all \
-v /home/admin/bot_dgt/dgt/config.json:/usr/src/app/src/config.json \
-v /home/admin/bot_dgt/data/dgt/:/usr/src/app/src/data \
-v /home/admin/bot_dgt/dgt/logs:/usr/src/app/logs \
-v /data/models/rerankers/bge-reranker-large:/data/models/rerankers/bge-reranker-large \
-v /data/models/embeddings/bge-m3:/data/models/embeddings/bge-m3 \
-p 8083:8083 -e PORT=8083 --name bot_dgt_instance bot_dgt
```

Nota: En el archivo de configuración `src/config.json` se debe indicar la ruta a: la carpeta con los datos, el servicio llama3, el servicio SOLAR, servicio del LLM que genera las preguntas, servicio embeddings, y modelo reranker.

Nota: Para los datos, mantén [la estructura actual que está en sharepoint](https://colaboraage-my.sharepoint.com/:f:/r/personal/josecarlos_martinez_correo_gob_es/Documents/Documentaci%C3%B3n%20Servicio%20060/LHF-VF/Doc%20Proyectos%20LHF/Bot%20Agencias/Otros%20documentos/documentos%20maestros%20procesados?csf=1&web=1&e=Zrnbyx). Son dos directorios: uno llamado FAQ_csvs con los excels de preguntas frecuentes formateados; otro llamado text_docs con los Words. 

2. Despliegue del conector (NOTA: no lo hacemos, el conector de agencias ya te redirige a este bot si usas `collection=DGT`)
```bash
screen -S connector
uvicorn connectors.rag_connector_agencias:app --host 0.0.0.0 --port 8085

```


### Diagnóstico
Igual que el Bot Agencias, pero cuando llames al conector pon `collection=DGT`.


## Datos
Los datos son una mezcla de:
- Datos DGT de la Réplica Azure
- Datos DGT del Bot Agencias
- Word extra creado por Consuelo

Están aquí: 
- [Originales](https://colaboraage-my.sharepoint.com/:f:/r/personal/josecarlos_martinez_correo_gob_es/Documents/Documentaci%C3%B3n%20Servicio%20060/LHF-VF/Doc%20Proyectos%20LHF/Bot%20DGT/Otros%20documentos/Documentos%20maestros%20originales?csf=1&web=1&e=CctVRJ)
- [Procesados](https://gitlab-ic.scae.redsara.es/060_lhf/bot_dgt/-/blob/main/dgt-data.zip?ref_type=heads)

## Evaluación
No hay



## Documentación 
- [Carpeta en SharePoint](https://colaboraage-my.sharepoint.com/:f:/r/personal/josecarlos_martinez_correo_gob_es/Documents/Documentaci%C3%B3n%20Servicio%20060/LHF-VF/Doc%20Proyectos%20LHF/Bot%20DGT?csf=1&web=1&e=8suSCP)
- Documentación de la API en sharepoint: [la del Bot agencias](https://colaboraage-my.sharepoint.com/:w:/r/personal/josecarlos_martinez_correo_gob_es/Documents/Documentaci%C3%B3n%20Servicio%20060/LHF-VF/Doc%20Proyectos%20LHF/Bot%20Agencias/Otros%20documentos/Documentaci%C3%B3n%20de%20la%20API%20Bot%20Agencias.docx?d=w6cbb382e92204bc58b0021fa151a2feb&csf=1&web=1&e=yXRGa5)
- Documentación formal de SGAD: [DST](https://colaboraage-my.sharepoint.com/:w:/r/personal/josecarlos_martinez_correo_gob_es/Documents/Documentaci%C3%B3n%20Servicio%20060/LHF-VF/Doc%20Proyectos%20LHF/Bot%20DGT/25102024_DST_Bot%20DGT%20v0%20.docx?d=wcac47fba3df448ccab9035e337d85cd6&csf=1&web=1&e=fQk0Jl) y [ENT](https://colaboraage-my.sharepoint.com/:w:/r/personal/josecarlos_martinez_correo_gob_es/Documents/Documentaci%C3%B3n%20Servicio%20060/LHF-VF/Doc%20Proyectos%20LHF/Bot%20DGT/25102024_ENT_Bot%20DGT%20v0.docx?d=wbd810dca2d404afabd6220d64b31239e&csf=1&web=1&e=rLmU9Y)


## Uso actual

Actualmente:
- el conector es el de agencias: está desplegado en 161:8085 (screen session)
- el docker con el RAG está desplegado en 159:8083 (docker)
- Hay un front-end de Pablo en demochat.pre.060.gob.es. Este front apunt al conector del bot Agencias, y le manda peticiones con el argumento `collections=DGT`, así que llegan al docker del bot DGT en 159:8083


## Otros
NA

