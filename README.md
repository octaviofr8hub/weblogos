## Requisitos Previos
### Puertos Requeridos
- **9000**: API S3 de MinIO
- **9001**: Consola Web de MinIO

### Espacio en Disco
- Mínimo: **2 GB** para pruebas
- Recomendado: **20 GB+** para producción (según el volumen de archivos)

---

### Estructura de Almacenamiento

```
pv-infra/
├── compose.yml               # Docker Compose con MinIO y Qdrant
├── policy-pvapp.json         # Política de acceso para usuario pvapp
├── minio-data/               # Volumen de datos persistentes
│   ├── .minio.sys/           # Metadatos del sistema MinIO
│   ├── pv-material/          # Bucket para material educativo (PDFs)
│   │   ├── cache_text/       # Cache de textos extraídos
│   │   └── pruebas/          # PDFs de pacientes de prueba
│   └── user-profiles/        # Bucket para fotos de perfil
│       ├── estudiantes/      # Fotos de alumnos
│       └── profesores/       # Fotos de profesores
└── qdrant-storage/           # (Vector DB, opcional)
```

### Credenciales por Defecto

**Root Admin** | `pvadmin` | `super-strong-pass-change-me` |
**App User** | `pvapp` | `PacienteVirtual2025` |

---

## Instalación con Docker Compose

### Paso 1: Crear Directorio de Infraestructura

```bash
# Crear directorio principal
mkdir -p ~/pv-infra
cd ~/pv-infra

# Crear subdirectorios para volúmenes
mkdir -p minio-data
mkdir -p qdrant-storage
```

### Paso 2: Crear `compose.yml`

```bash
cat > compose.yml << 'EOF'
services:
  minio:
    image: quay.io/minio/minio:latest
    container_name: pv-minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: pvadmin
      MINIO_ROOT_PASSWORD: super-strong-pass-change-me
    ports:
      - "9000:9000"   # API S3
      - "9001:9001"   # Consola web
    volumes:
      - ./minio-data:/data
    restart: unless-stopped
    networks:
      - pv-network

  qdrant:
    image: qdrant/qdrant:latest
    container_name: pv-qdrant
    ports:
      - "6333:6333"   # HTTP API
    volumes:
      - ./qdrant-storage:/qdrant/storage:Z
    restart: unless-stopped
    networks:
      - pv-network

networks:
  pv-network:
    driver: bridge
EOF
```

### Paso 3: Levantar MinIO

```bash
# Iniciar los servicios
docker compose up -d

# Verificar que esté corriendo
docker compose ps

# Ver logs (opcional)
docker compose logs -f minio
```

**Salida esperada:**
```
✅ Container pv-minio  Started
✅ MinIO API: http://YOUR_IP:9000
✅ MinIO Console: http://YOUR_IP:9001
```

---

## 🪣 Configuración de Buckets

### Opción 1: Desde la Consola Web (Recomendado)

1. **Acceder a la consola:**
   ```
   http://YOUR_IP:9001
   ```

2. **Iniciar sesión:**
   - Usuario: `pvadmin`
   - Contraseña: `super-strong-pass-change-me`

3. **Crear Buckets:**
   - Click en **"Buckets"** → **"Create Bucket"**
   - Crear los siguientes buckets:
     - `pv-material` (para PDFs de pacientes)
     - `user-profiles` (para fotos de perfil)

### Opción 2: Desde CLI con MinIO Client

```bash
# Instalar mc (MinIO Client)
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configurar alias para tu servidor MinIO
mc alias set myminio http://localhost:9000 pvadmin super-strong-pass-change-me

# Crear los buckets necesarios
mc mb myminio/pv-material
mc mb myminio/user-profiles

# Verificar que se crearon
mc ls myminio

# Salida esperada:
# [2025-12-08 12:00:00 UTC]     0B pv-material/
# [2025-12-08 12:00:00 UTC]     0B user-profiles/
```

---

## Configuración de Usuarios y Políticas

### Crear Usuario de Aplicación (pvapp)

#### Desde la Consola Web:

1. **Identity** → **Users** → **Create User**
2. **Access Key:** `pvapp`
3. **Secret Key:** `PacienteVirtual2025`
4. **Assign Policies:** (crear política personalizada)

#### Crear Política para pvapp

Crear archivo `policy-pvapp.json`:

```bash
cat > policy-pvapp.json << 'EOF'
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action":["s3:ListBucket"],
      "Resource":["arn:aws:s3:::pv-material", "arn:aws:s3:::user-profiles"]
    },
    {
      "Effect":"Allow",
      "Action":["s3:GetObject","s3:PutObject","s3:DeleteObject"],
      "Resource":["arn:aws:s3:::pv-material/*", "arn:aws:s3:::user-profiles/*"]
    }
  ]
}
EOF
```

#### Aplicar política desde CLI:

```bash
# Crear la política
mc admin policy create myminio pvapp-policy policy-pvapp.json

# Crear usuario pvapp
mc admin user add myminio pvapp PacienteVirtual2025

# Asignar política al usuario
mc admin policy attach myminio pvapp-policy --user pvapp

# Verificar
mc admin user info myminio pvapp
```

---

## Integración con la Aplicación

### Variables de Entorno

Crear/actualizar los archivos `.env` del proyecto:

**`env/docs.env`:**
```bash
# MinIO / S3 Configuration
S3_ENDPOINT=http://100.64.0.2:9000
S3_REGION=us-east-1
S3_BUCKET_MATERIAL=pv-material
S3_USE_SSL=false
S3_ACCESS_KEY=pvapp
S3_SECRET_KEY=PacienteVirtual2025
```

**`env/ingesta.env`:**
```bash
S3_ENDPOINT=http://100.64.0.2:9000
S3_REGION=us-east-1
S3_BUCKET_MATERIAL=pv-material
S3_USE_SSL=false
S3_ACCESS_KEY=pvapp
S3_SECRET_KEY=PacienteVirtual2025
```

Reemplaza `100.64.0.2` con la IP de tu servidor.

### Código Python de Ejemplo

```python
import boto3
import os
from dotenv import load_dotenv

load_dotenv("env/docs.env")

# Inicializar cliente S3/MinIO
s3_client = boto3.client(
    's3',
    endpoint_url=os.getenv('S3_ENDPOINT'),
    region_name=os.getenv('S3_REGION'),
    aws_access_key_id=os.getenv('S3_ACCESS_KEY'),
    aws_secret_access_key=os.getenv('S3_SECRET_KEY'),
    use_ssl=os.getenv('S3_USE_SSL', 'false').lower() == 'true'
)

# Subir archivo
s3_client.upload_file(
    'local_file.pdf',
    'pv-material',
    'pacientes/demo.pdf'
)

# Descargar archivo
s3_client.download_file(
    'pv-material',
    'pacientes/demo.pdf',
    '/tmp/demo.pdf'
)

# Listar objetos
response = s3_client.list_objects_v2(Bucket='pv-material')
for obj in response.get('Contents', []):
    print(obj['Key'])
```

---

## Verificación de Instalación

### Test de Conectividad

```bash
# 1. Verificar API S3
curl http://localhost:9000/minio/health/live
# Salida esperada: 200 OK

# 2. Verificar consola web
curl -I http://localhost:9001
# Salida esperada: 200 OK

# 3. Listar buckets
mc ls myminio/

# 4. Subir archivo de prueba
echo "test" > test.txt
mc cp test.txt myminio/pv-material/test/test.txt

```

