# Guía de Despliegue - Validador GIS en Producción

## Información del Proyecto

**Cliente:** Insumosgeo  
**URL Producción:** https://insumosgeo.com/validador  
**Administrador Servidor:** David Vasquez (@sparveriusdev)  
**Desarrollador Original:** Pablo Aranzazu (@paaranzazuv)

### Repositorios

- **Repo Original (Desarrollo):** https://github.com/paaranzazuv/PREVALIDADOR
- **Fork Producción:** https://github.com/sparveriusdev/PREVALIDADOR
- **Rama Producción:** `servidor-produccion`
- **Rama Desarrollo:** `version-2`

---

## Arquitectura del Servidor

### Stack Tecnológico

```
Ubuntu Server 24.04 LTS
├── Nginx 1.26.3 (Proxy Reverso + SSL)
├── Tomcat 9 (Contenedor de aplicaciones)
├── Node.js v20.18.1 + PM2 (Gestor de procesos)
├── Python 3.x + uv (Gestor de dependencias)
└── Git (Control de versiones)
```

### Estructura de Directorios

```
/var/lib/tomcat9/webapps/validador/
├── validador-server.js          # Servidor Node.js/Express
├── package.json                  # Dependencias Node.js
├── .env                          # Configuración de entorno
├── pyproject.toml               # Metadatos del proyecto web
├── public/                      # Frontend (HTML, CSS, JS)
│   ├── index.html
│   ├── app.js
│   └── styles.css
├── repo/                        # Repositorio clonado de Python
│   ├── .venv/                   # Entorno virtual Python
│   ├── pyproject.toml          # Configuración del paquete Python
│   ├── src/prevalidador/       # Código fuente Python
│   └── ...
├── tmp/                         # Archivos temporales (uploads)
├── backups/                     # Respaldos automáticos
│   ├── inbox/                   # Archivos de entrada
│   ├── resultados/              # Resultados procesados
│   └── historico/               # Históricos de validación
└── node_modules/                # Dependencias Node.js
```

---

## Configuración de Nginx

### Conceptos de Proxy Reverso

Nginx actúa como **proxy reverso**, interceptando peticiones HTTPS y redireccionándolas a servicios internos:

```
Cliente HTTPS (443) → Nginx → Backend HTTP (puerto interno)
```

**Ventajas:**
- SSL/TLS centralizado en Nginx
- Balanceo de carga
- Cache y compresión
- Seguridad (backends no expuestos directamente)
- Múltiples aplicaciones en un solo dominio

### Configuración para Validador

**Archivo:** `/etc/nginx/sites-available/insumosgeo.com`

```nginx
# Frontend estático (HTML, CSS, JS)
location /validador/ {
    alias /var/lib/tomcat9/webapps/validador/public/;
    try_files $uri $uri/ /validador/index.html;
    
    # Cache para recursos estáticos
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1h;
        add_header Cache-Control "public, immutable" always;
    }
}

# API (proxy a Node.js en puerto 3009)
location /validador/api/ {
    proxy_pass http://localhost:3009/api/;
    proxy_http_version 1.1;
    
    # Headers necesarios para proxy
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    
    # CORS
    add_header Access-Control-Allow-Origin "https://insumosgeo.com" always;
    add_header Access-Control-Allow-Methods "GET, POST, OPTIONS" always;
    
    # Timeouts para procesos largos (Python)
    proxy_read_timeout 600s;
    
    # Upload de archivos grandes
    client_max_body_size 500M;
    client_body_timeout 600s;
}
```

**Flujo de peticiones:**

1. Usuario visita: `https://insumosgeo.com/validador/`
2. Nginx sirve: `/var/lib/tomcat9/webapps/validador/public/index.html`
3. Usuario sube archivos: `POST https://insumosgeo.com/validador/api/validate`
4. Nginx redirecciona a: `http://localhost:3009/api/validate`
5. Node.js procesa con Python y responde
6. Usuario descarga: `GET https://insumosgeo.com/validador/api/download/{jobId}`

---

## Configuración de Entorno (.env)

**Archivo:** `/var/lib/tomcat9/webapps/validador/.env`

```env
# Servidor Node.js
PORT=3009
NODE_ENV=production

# Repositorio Git
REPO_URL=https://github.com/sparveriusdev/PREVALIDADOR.git
REPO_BRANCH=servidor-produccion

# Python (se detecta automáticamente según SO)
PYTHON_BIN=python3
VENV_DIR=.venv

# Configuración del validador Python
PY_ENTRY_MODE=module
PY_ENTRY_TARGET=prevalidador.main
```

### Detección Automática de Sistema Operativo

El código detecta el SO y usa el Python apropiado:

```javascript
// En validador-server.js
const PYTHON_BIN = process.env.PYTHON_BIN || 
  (process.platform === "win32" ? "py" : "python3");
```

**Windows:** Usa `py` launcher  
**Linux/Mac:** Usa `python3`

---

## Gestión de Dependencias Python

### Entorno de Desarrollo (Windows)

```bash
# Clonar repositorio
git clone https://github.com/paaranzazuv/PREVALIDADOR.git
cd PREVALIDADOR

# Crear entorno virtual
py -m venv .venv
.venv\Scripts\activate

# Instalar dependencias
pip install -r requirements.txt
pip install -e .

# Ejecutar
py -m prevalidador.main --help
```

### Entorno de Producción (Ubuntu con uv)

```bash
# Clonar fork
cd /var/lib/tomcat9/webapps/validador
git clone https://github.com/sparveriusdev/PREVALIDADOR.git repo
cd repo
git checkout servidor-produccion

# Instalar uv (si no está instalado)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Crear entorno y instalar dependencias
uv venv
uv pip install -r requirements.txt
uv pip install -e .

# Verificar instalación
uv run python -c "import prevalidador; print('OK')"
```

### Compatibilidad Multiplataforma

El código detecta automáticamente el gestor de paquetes disponible:

```javascript
// Intenta con uv primero (más rápido en Linux)
try {
  await execFileAsync("uv", ["pip", "install", "-r", req]);
} catch (uvErr) {
  // Fallback a pip tradicional (Windows)
  await execFileAsync(pip, ["install", "-r", req]);
}
```

---

## Gestión con PM2

### Comandos Básicos

```bash
# Iniciar aplicación
pm2 start validador-server.js --name validador-server

# Ver estado
pm2 status

# Ver logs en tiempo real
pm2 logs validador-server

# Reiniciar
pm2 restart validador-server

# Detener
pm2 stop validador-server

# Eliminar
pm2 delete validador-server

# Guardar configuración (auto-inicio)
pm2 save
pm2 startup
```

### Logs

```bash
# Logs de PM2
pm2 logs validador-server --lines 100

# Logs de Nginx
sudo tail -f /var/log/nginx/validador_api_error.log
sudo tail -f /var/log/nginx/validador_api_access.log

# Logs del sistema
sudo journalctl -u pm2-root -f
```

---

## Flujo de Trabajo Git

### Para el Desarrollador (Pablo)

```bash
# Trabajar en rama de desarrollo
git checkout version-2

# Hacer cambios y commit
git add .
git commit -m "Nueva funcionalidad X"
git push origin version-2

# Notificar al administrador del servidor
```

### Para el Administrador (David)

```bash
cd /var/lib/tomcat9/webapps/validador/repo

# Ver rama actual
git branch

# Actualizar desde el repo original
git remote add upstream https://github.com/paaranzazuv/PREVALIDADOR.git
git fetch upstream
git merge upstream/version-2

# Resolver conflictos si los hay
git status
# Editar archivos en conflicto
git add .
git commit -m "Merge cambios del upstream"

# Push al fork
git push origin servidor-produccion

# Reiniciar aplicación
cd ..
pm2 restart validador-server
```

### Sincronización de Cambios

**Cuando el desarrollador sube cambios:**

```bash
# 1. En el servidor
cd /var/lib/tomcat9/webapps/validador/repo

# 2. Guardar cambios locales si los hay
git stash

# 3. Obtener cambios del upstream
git fetch upstream
git merge upstream/version-2

# 4. Recuperar cambios locales
git stash pop

# 5. Resolver conflictos si existen
# Editar archivos marcados en conflicto
git add .
git commit -m "Merge y resolución de conflictos"

# 6. Push al fork
git push origin servidor-produccion

# 7. Reinstalar dependencias si cambiaron
uv pip install -e .

# 8. Reiniciar
cd ..
pm2 restart validador-server
```

---

## Cambios para Compatibilidad Windows/Linux

### 1. Detección de Python

**Antes (solo Windows):**
```javascript
const PYTHON_BIN = process.env.PYTHON_BIN || "py";
```

**Después (multiplataforma):**
```javascript
const PYTHON_BIN = process.env.PYTHON_BIN || 
  (process.platform === "win32" ? "py" : "python3");
```

### 2. Rutas de Venv

```javascript
function pyBinInVenv(exe = "python") {
  const bin = process.platform === "win32"
    ? path.join(REPO_DIR, VENV_DIR, "Scripts", `${exe}.exe`)
    : path.join(REPO_DIR, VENV_DIR, "bin", exe);
  return bin;
}
```

### 3. Soporte para uv y pip

```javascript
// Intenta instalar con uv (Linux, más rápido)
try {
  await execFileAsync("uv", ["pip", "install", "-r", req]);
} catch (uvErr) {
  // Fallback a pip tradicional (Windows)
  await execFileAsync(pip, ["install", "-r", req]);
}
```

### 4. Rutas de API con prefijo Nginx

**Frontend (app.js):**
```javascript
// Usar ruta completa con prefijo
fetch("/validador/api/validate", { method: "POST", body: fd });
```

**Backend (validador-server.js):**
```javascript
// URL de descarga con prefijo
downloadUrl: `/validador/api/download/${jobId}`
```

### 5. Escuchar en todas las interfaces

**Antes:**
```javascript
app.listen(PORT, () => { ... });
```

**Después:**
```javascript
app.listen(PORT, '0.0.0.0', () => { ... });
```

Esto permite que Nginx se conecte desde localhost.

---

## Troubleshooting

### Error: "Module 'prevalidador' not found"

```bash
cd /var/lib/tomcat9/webapps/validador/repo
uv pip install -e .
pm2 restart validador-server
```

### Error: "Port 3009 already in use"

```bash
# Matar procesos en el puerto
sudo lsof -ti:3009 | xargs kill -9

# O reiniciar PM2
pm2 delete all
pm2 start validador-server.js --name validador-server
```

### Error: "Git dubious ownership"

```bash
git config --global --add safe.directory /var/lib/tomcat9/webapps/validador/repo
```

### Error 502 Bad Gateway

```bash
# Verificar que Node.js esté corriendo
pm2 status

# Verificar que escucha en 3009
sudo netstat -tlnp | grep 3009

# Ver logs de error
pm2 logs validador-server --err
```

### Actualización de Nginx

```bash
# Editar configuración
sudo nano /etc/nginx/sites-available/insumosgeo.com

# Verificar sintaxis
sudo nginx -t

# Recargar (sin downtime)
sudo systemctl reload nginx
```

---

## Testing en Desarrollo

### Backend (Node.js)

```bash
# Iniciar manualmente
cd /var/lib/tomcat9/webapps/validador
node validador-server.js

# Probar health check
curl http://localhost:3009/api/health
```

### Python

```bash
cd /var/lib/tomcat9/webapps/validador/repo

# Con uv
uv run python -m prevalidador.main --help

# Con venv tradicional
source .venv/bin/activate
python -m prevalidador.main --help
deactivate
```

### Frontend

Abrir en navegador: `http://localhost:3009` (desarrollo) o `https://insumosgeo.com/validador/` (producción)

---

## Checklist de Despliegue

- [ ] Código actualizado en fork de sparveriusdev
- [ ] Rama `servidor-produccion` creada y pusheada
- [ ] `.env` configurado correctamente
- [ ] Dependencias Python instaladas: `uv pip install -e .`
- [ ] Dependencias Node.js instaladas: `npm install`
- [ ] Configuración Nginx actualizada y recargada
- [ ] Git configurado para safe.directory
- [ ] PM2 iniciado: `pm2 start validador-server.js --name validador-server`
- [ ] PM2 configurado para auto-inicio: `pm2 save && pm2 startup`
- [ ] Health check funciona: `curl https://insumosgeo.com/validador/api/health`
- [ ] Test de subida de archivos exitoso
- [ ] Logs monitoreados: `pm2 logs validador-server`

---

## Contacto

**Servidor/DevOps:** David Vasquez (@sparveriusdev) - Insumosgeo  
**Desarrollo Python:** Pablo Aranzazu (@paaranzazuv)  

**Soporte:** Grupo T.I Insumosgeo - https://insumosgeo.com
