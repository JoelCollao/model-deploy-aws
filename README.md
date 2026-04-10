# Diabetes Prediction API - Model Deploy on AWS Lambda

API REST construida con **FastAPI** que expone un modelo de Machine Learning entrenado con Scikit-learn para predecir si un paciente tiene diabetes. El proyecto esta containerizado con Docker y disenado para desplegarse en **AWS Lambda** usando el adaptador Mangum.

---

## Tabla de contenidos

- [Descripcion](#descripcion)
- [Arquitectura](#arquitectura)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Requisitos previos](#requisitos-previos)
- [Configuracion del entorno local](#configuracion-del-entorno-local)
- [Docker](#docker)
- [Endpoints](#endpoints)
- [Pruebas - Local vs Contenedor](#pruebas---local-vs-contenedor)
- [Despliegue en AWS Lambda](#despliegue-en-aws-lambda)
- [Dependencias principales](#dependencias-principales)

---

## Descripcion

El modelo fue entrenado sobre el dataset **Pima Indians Diabetes** y serializado en `diabetes_model.pkl`. La API recibe 8 caracteristicas clinicas de un paciente y devuelve un resultado binario: **Diabetic** / **Non-diabetic**.

---

## Arquitectura

```
Cliente HTTP
     |
     v
  [Local directo]        [Docker local]          [Produccion AWS]
  localhost:8000         localhost:9001           Lambda Function URL
       |                      |                         |
       v                      v                         v
 FastAPI+Uvicorn        Lambda RIE (8080)          FastAPI+Mangum
       |                 Mangum adapta                  |
       +------------------+-----+---------------------+
                          |
                          v
               Scikit-learn model (diabetes_model.pkl)
```

| Modo | Puerto | Como se llama |
|------|--------|---------------|
| Local directo | 8000 | POST /predict directamente |
| Docker (RIE) | 9001->8080 | Evento Lambda v2 envuelto por Mangum |
| AWS Lambda | Function URL | Evento Lambda automatico via API Gateway |

---

## Estructura del proyecto

```
model-deploy-aws/
├── app.py                  # Aplicacion FastAPI con el endpoint /predict
├── invoke_api.py           # Cliente de prueba que llama a la Lambda URL de AWS
├── diabetes_model.pkl      # Modelo entrenado serializado (Scikit-learn 1.5.2)
├── requirements.txt        # Dependencias Python con versiones fijadas
├── Dockerfile              # Imagen basada en Lambda Python 3.10 de AWS ECR
├── .gitignore              # Archivos y carpetas excluidos de Git
├── .env/                   # Entorno virtual local (excluido de Git)
└── README.md               # Documentacion del proyecto
```

---

## Requisitos previos

| Herramienta | Version minima |
|-------------|----------------|
| Python      | 3.10           |
| Docker      | 20.x           |
| AWS CLI     | 2.x (solo para despliegue en AWS) |

---

## Configuracion del entorno local

### 1. Clonar el repositorio

```bash
git clone <URL_DEL_REPOSITORIO>
cd model-deploy-aws
```

### 2. Crear y activar el entorno virtual

```bash
python -m venv .env

# Windows
.env\Scripts\activate

# Linux / macOS
source .env/bin/activate
```

### 3. Instalar dependencias

```bash
pip install -r requirements.txt
```

> `scikit-learn` esta fijado a `1.5.2` porque el modelo fue entrenado con esa version exacta.
> Usar otra version genera `InconsistentVersionWarning` y puede producir resultados incorrectos.

### 4. Levantar el servidor

```bash
.env\Scripts\activate
python app.py
# Servidor disponible en http://localhost:8000
```

> Si el puerto 8000 esta ocupado por un proceso anterior:
> ```powershell
> Get-NetTCPConnection -LocalPort 8000 | Select-Object -ExpandProperty OwningProcess | ForEach-Object { Stop-Process -Id $_ -Force }
> ```

---

## Docker

### Construir la imagen --datos

```bash
docker build -t diabetes-model-api .
```

### Crear y ejecutar el contenedor

```bash
docker run -d --name diabetes-api-container -p 9001:8080 diabetes-model-api
```

El puerto **8080** es el Lambda Runtime Interface Emulator (RIE) dentro del contenedor, mapeado al puerto **9001** del host.

> El contenedor **no expone FastAPI directamente**: todo pasa por el Lambda RIE y Mangum.

### Verificar estado

```bash
docker ps --filter "name=diabetes-api-container"
```

### Ver logs

```bash
docker logs diabetes-api-container
```

### Detener y eliminar

```bash
docker stop diabetes-api-container
docker rm diabetes-api-container
```

---

## Endpoints

| Metodo | Ruta     | Descripcion                         |
|--------|----------|-------------------------------------|
| POST   | /predict | Recibe datos del paciente y predice |

### Esquema de entrada

```json
{
  "Pregnancies": 3,
  "Glucose": 120,
  "BloodPressure": 70,
  "SkinThickness": 20,
  "Insulin": 80,
  "BMI": 25.6,
  "DiabetesPedigreeFunction": 0.5,
  "Age": 32
}
```

### Esquema de respuesta

```json
{
  "prediction": "Non-diabetic"
}
```

---

## Pruebas - Local vs Contenedor

### PRUEBA LOCAL (python app.py)

FastAPI escucha en el puerto **8000**. El request va directo al endpoint `/predict` sin envoltura Lambda.

**curl:**
```bash
curl -X POST "http://localhost:8000/predict" \
  -H "Content-Type: application/json" \
  -d "{\"Pregnancies\":3,\"Glucose\":120,\"BloodPressure\":70,\"SkinThickness\":20,\"Insulin\":80,\"BMI\":25.6,\"DiabetesPedigreeFunction\":0.5,\"Age\":32}"
```

**PowerShell:**
```powershell
Invoke-RestMethod -Method POST `
  -Uri "http://localhost:8000/predict" `
  -ContentType "application/json" `
  -Body '{"Pregnancies":3,"Glucose":120,"BloodPressure":70,"SkinThickness":20,"Insulin":80,"BMI":25.6,"DiabetesPedigreeFunction":0.5,"Age":32}'
```

**Swagger UI (navegador):**
```
http://localhost:8000/docs
```

**Respuesta esperada:**
```json
{
  "prediction": "Non-diabetic"
}
```

---

### PRUEBA EN CONTENEDOR DOCKER (Lambda RIE)

El contenedor corre el Lambda Runtime Interface Emulator (RIE) en el puerto **8080** (mapeado al **9001** del host).
Mangum requiere que el evento tenga estructura de **API Gateway v2** con los campos
`"version"`, `"requestContext"` y el body como **string JSON escapado**.

> Sin esos campos Mangum devuelve el error:
> `"The adapter was unable to infer a handler to use for the event"`

**PowerShell desde tu maquina (puerto 9001):**
```powershell
Invoke-RestMethod -Method POST `
  -Uri "http://localhost:9001/2015-03-31/functions/function/invocations" `
  -ContentType "application/json" `
  -Body '{"version":"2.0","requestContext":{"http":{"method":"POST","path":"/predict","sourceIp":"127.0.0.1"}},"headers":{"content-type":"application/json"},"body":"{\"Pregnancies\":3,\"Glucose\":120,\"BloodPressure\":70,\"SkinThickness\":20,\"Insulin\":80,\"BMI\":25.6,\"DiabetesPedigreeFunction\":0.5,\"Age\":32}","isBase64Encoded":false}'
```

**bash dentro del contenedor (puerto 8080):**
```bash
# Entrar al contenedor
docker exec -it diabetes-api-container bash

# Ejecutar el request
curl -s -X POST http://localhost:8080/2015-03-31/functions/function/invocations \
  -H "Content-Type: application/json" \
  -d '{"version":"2.0","requestContext":{"http":{"method":"POST","path":"/predict","sourceIp":"127.0.0.1"}},"headers":{"content-type":"application/json"},"body":"{\"Pregnancies\":3,\"Glucose\":120,\"BloodPressure\":70,\"SkinThickness\":20,\"Insulin\":80,\"BMI\":25.6,\"DiabetesPedigreeFunction\":0.5,\"Age\":32}","isBase64Encoded":false}'
```

**Respuesta esperada del contenedor:**
```json
{
  "statusCode": 200,
  "body": "{\"prediction\":\"Non-diabetic\"}",
  "headers": {"content-type": "application/json"},
  "isBase64Encoded": false
}
```

> La respuesta tiene envoltura Lambda (`statusCode`, `body` como string).
> En produccion AWS, API Gateway desempaqueta ese body automaticamente.

---

### Diferencias clave local vs contenedor

|                  | Local (puerto 8000)      | Contenedor (puerto 9001)                              |
|------------------|--------------------------|-------------------------------------------------------|
| **URL**          | `/predict`               | `/2015-03-31/functions/function/invocations`          |
| **Body request** | JSON plano del paciente  | Evento Lambda v2 con body como string JSON escapado   |
| **Campos extra** | Ninguno                  | `version`, `requestContext`, `isBase64Encoded`        |
| **Respuesta**    | `{"prediction":"..."}`   | `{"statusCode":200,"body":"{\"prediction\":\"...\"}"}`|

---

## Despliegue en AWS Lambda

### 1. Autenticarse en Amazon ECR

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

### 2. Crear repositorio ECR

```bash
aws ecr create-repository --repository-name diabetes-model-api --region us-east-1
```

### 3. Etiquetar y subir la imagen

```bash
docker tag diabetes-model-api:latest \
  <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/diabetes-model-api:latest

docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/diabetes-model-api:latest
```

### 4. Crear la funcion Lambda

En AWS Console > Lambda > **Create function** > **Container image** > apuntar al URI de ECR.

- **Handler**: `app.handler`
- **Architecture**: x86_64
- **Memory**: 512 MB minimo
- **Timeout**: 30 segundos

---

## Dependencias principales

| Paquete      | Version | Rol                                                                          |
|--------------|---------|------------------------------------------------------------------------------|
| fastapi      | latest  | Framework web para la API REST                                               |
| uvicorn      | latest  | Servidor ASGI para ejecucion local                                           |
| mangum       | 0.17.0  | Adaptador FastAPI <-> AWS Lambda (version fijada por compatibilidad de formato)|
| scikit-learn | 1.5.2   | Carga y ejecucion del modelo ML (version fijada igual que el entrenamiento)  |
| numpy        | latest  | Manipulacion de arrays para inferencia                                       |
| pydantic     | latest  | Validacion del esquema de entrada                                            |
