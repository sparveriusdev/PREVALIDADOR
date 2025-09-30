# Changelog - Adaptaciones para Servidor de Producción

## Información de Versiones

- **Versión Desarrollo:** 1.2 (Windows, @paaranzazuv)
- **Versión Producción:** 1.2-server (Ubuntu Server 24, @sparveriusdev)
- **Fecha:** 30 de septiembre de 2025
- **Repositorio Dev:** https://github.com/paaranzazuv/PREVALIDADOR
- **Repositorio Prod:** https://github.com/sparveriusdev/PREVALIDADOR (rama: `servidor-produccion`)

---

## app.js

### Cambio en la URL de la API

**Archivo:** `public/app.js`

#### Versión Desarrollo (Original)
```javascript
const res = await fetch("/api/validate", { 
  method: "POST", 
  body: fd 
});
```

#### Versión Producción (Adaptada)
```javascript
const res = await fetch("/validador/api/validate", { 
  method: "POST", 
  body: fd 
});
```

**Razón del cambio:**

En desarrollo, la aplicación se ejecuta directamente en el puerto del servidor Node.js (ej: `localhost:3009`), por lo que las rutas son relativas al servidor.

En producción, Nginx actúa como proxy reverso y todas las aplicaciones comparten el mismo dominio bajo diferentes paths:
- `/pqrs/api/` → Node.js en puerto 3007
- `/consulta-api/` → Node.js en puerto 3005
- `/consolidador-api/` → Node.js en puerto 3000
- `/validador/api/` → Node.js en puerto 3009 (nuestra app)

**Flujo en producción:**
1. Cliente → `https://insumosgeo.com/validador/api/validate`
2. Nginx intercepta → `location /validador/api/`
3. Nginx redirige → `http://localhost:3009/api/validate`
4. Node.js procesa la petición

### Mejora en el manejo de errores

#### Versión Desarrollo (Original)
```javascript
if (!res.ok) throw new Error("Error de validación");
```

#### Versión Producción (Adaptada)
```javascript
if (!res.ok) {
  const errorData = await res.json().catch(() => ({ error: "Error de validación" }));
  throw new Error(errorData.error || "Error de validación");
}
```

**Razón del cambio:**

Captura y muestra mensajes de error más descriptivos del servidor, facilitando el debugging en producción. Si la respuesta no es JSON válido, tiene un fallback al mensaje genérico.

### Mejora en logging de errores

#### Versión Desarrollo (Original)
```javascript
console.error(e);
```

#### Versión Producción (Adaptada)
```javascript
console.error("Error completo:", e);
```

**Razón del cambio:**

Agrega un prefijo descriptivo para identificar mejor los errores en las herramientas de desarrollo del navegador cuando hay múltiples aplicaciones corriendo.

---

## validador-server.js

### 1. Detección automática del binario de Python

#### Versión Desarrollo (Original)
```javascript
const PYTHON_BIN = process.env.PYTHON_BIN || "py";
```

#### Versión Producción (Adaptada)
```javascript
const PYTHON_BIN = process.env.PYTHON_BIN || 
  (process.platform === "win32" ? "py" : "python3");
```

**Razón del cambio:**

- **Windows:** Usa el launcher `py` que gestiona múltiples versiones de Python
- **Linux/Ubuntu:** Usa `python3` que es el estándar en sistemas Unix

Esto permite que el mismo código funcione sin configuración adicional en ambos sistemas operativos.

---

### 2. Configuración de CORS

#### Versión Desarrollo (Original)
```javascript
app.use(cors());
```

#### Versión Producción (Adaptada)
```javascript
app.use(cors({
  origin: ['https://insumosgeo.com', 'http://localhost:3009', 'http://localhost'],
  credentials: true
}));
```

**Razón del cambio:**

Restringe CORS a dominios específicos por seguridad. En desarrollo acepta cualquier origen, pero en producción debe ser explícito para prevenir ataques de cross-origin.

---

### 3. Creación del directorio temporal

#### Versión Desarrollo (Original)
```javascript
// No existe esta verificación
```

#### Versión Producción (Adaptada)
```javascript
const TMP_DIR = path.join(__dirname, "tmp");
if (!exists(TMP_DIR)) {
  fs.mkdirSync(TMP_DIR, { recursive: true });
}
```

**Razón del cambio:**

En el servidor, el directorio `tmp` puede no existir inicialmente o ser eliminado durante mantenimiento. Esta verificación previene errores al intentar crear subdirectorios.

---

### 4. Límites de upload en multer

#### Versión Desarrollo (Original)
```javascript
const upload = multer({
  storage: multer.diskStorage({ ... })
});
```

#### Versión Producción (Adaptada)
```javascript
const upload = multer({
  storage: multer.diskStorage({ ... }),
  limits: {
    fileSize: 500 * 1024 * 1024, // 500MB
    files: 100
  }
});
```

**Razón del cambio:**

Establece límites explícitos para prevenir ataques de denegación de servicio (DoS) mediante uploads masivos. Coordinado con la configuración de Nginx (`client_max_body_size 500M`).

---

### 5. Función ensureRepoUpdated() - Manejo de conflictos Git

#### Versión Desarrollo (Original)
```javascript
const git = simpleGit(REPO_DIR);
await git.fetch();
await git.checkout(REPO_BRANCH);
await git.pull("origin", REPO_BRANCH, {"--rebase": "true"});
```

#### Versión Producción (Adaptada)
```javascript
const git = simpleGit(REPO_DIR);

try {
  await git.fetch();
  // Resetear cualquier cambio local antes de actualizar
  await git.reset(['--hard', `origin/${REPO_BRANCH}`]);
  await git.checkout(REPO_BRANCH);
  console.log("✅ Repositorio actualizado");
} catch (gitErr) {
  console.log("⚠️ No se pudo actualizar el repositorio, usando versión local");
  console.log(`   Error: ${gitErr.message}`);
}
```

**Razón del cambio:**

El `pull --rebase` falla cuando hay cambios locales sin confirmar. En producción:
- Los archivos pueden ser modificados por el proceso de validación
- Se usa `reset --hard` para descartar cambios y forzar sincronización con origin
- Agrega try-catch para no interrumpir el servicio si la actualización falla

**Advertencia de seguridad:** Esto descarta cambios locales. Si hay modificaciones importantes, deben estar en una rama separada (`servidor-produccion`).

---

### 6. Función ensureVenv() - Soporte para uv

#### Versión Desarrollo (Original)
```javascript
if (!exists(venvPath)) {
  await execFileAsync(PYTHON_BIN, ["-m", "venv", VENV_DIR], { cwd: REPO_DIR });
}

const pip = pyBinInVenv("pip");
if (!exists(pip)) {
  throw new Error(`No se encontró pip en el entorno virtual: ${pip}`);
}

await execFileAsync(pip, ["install", "-r", req], { ... });
```

#### Versión Producción (Adaptada)
```javascript
if (!exists(venvPath)) {
  try {
    // Intentar con uv primero (más rápido)
    await execFileAsync("uv", ["venv", VENV_DIR], { cwd: REPO_DIR });
  } catch (uvErr) {
    // Fallback a venv tradicional
    await execFileAsync(PYTHON_BIN, ["-m", "venv", VENV_DIR], { cwd: REPO_DIR });
  }
}

const pip = pyBinInVenv("pip");
if (!exists(pip)) {
  // No fallar, asumir que uv ya instaló todo
  console.log("ℹ️ Asumiendo que las dependencias ya están instaladas con uv");
  return;
}

try {
  // Intentar con uv pip (más rápido)
  await execFileAsync("uv", ["pip", "install", "-r", req], { ... });
} catch (uvErr) {
  // Fallback a pip tradicional
  await execFileAsync(pip, ["install", "-r", req], { ... });
}
```

**Razón del cambio:**

**uv** es un gestor de paquetes Python moderno y extremadamente rápido (escrito en Rust):
- Instalación de dependencias 10-100x más rápida que pip
- Gestión de entornos virtuales más eficiente
- Compatible con pip (puede usarse como drop-in replacement)

En Ubuntu Server con uv instalado, aprovecha la velocidad. En Windows con pip tradicional, funciona igual. El código detecta automáticamente qué herramienta está disponible.

---

### 7. Función runPrevalidator() - Soporte para uv run y PYTHONPATH

#### Versión Desarrollo (Original)
```javascript
let python = pyBinInVenv("python");
if (!exists(python)) python = PYTHON_BIN;

const args = [];
if (PY_ENTRY_MODE === "module") {
  args.push("-m", PY_ENTRY_TARGET);
}
args.push(inputDir, RULES_DIR, CATALOGS_DIR, HISTORICO_DIR, RESULTADOS_DIR);

await execFileAsync(python, args, {
  cwd: REPO_DIR,
  maxBuffer: 1024 * 1024 * 200,
  env: { ...process.env }
});
```

#### Versión Producción (Adaptada)
```javascript
let python = pyBinInVenv("python");

if (!exists(python)) {
  // Intentar con uv run
  try {
    await execFileAsync("uv", ["--version"], {});
    const args = [inputDir, RULES_DIR, CATALOGS_DIR, HISTORICO_DIR, RESULTADOS_DIR];
    const uvArgs = ["run", "--", "python", "-m", PY_ENTRY_TARGET, ...args];
    
    await execFileAsync("uv", uvArgs, {
      cwd: REPO_DIR,
      maxBuffer: 1024 * 1024 * 200,
      env: { ...process.env }
    });
    return;
  } catch (uvErr) {
    python = PYTHON_BIN;
  }
}

const args = [];
if (PY_ENTRY_MODE === "module") {
  args.push("-m", PY_ENTRY_TARGET);
}
args.push(inputDir, RULES_DIR, CATALOGS_DIR, HISTORICO_DIR, RESULTADOS_DIR);

await execFileAsync(python, args, {
  cwd: REPO_DIR,
  maxBuffer: 1024 * 1024 * 200,
  env: { 
    ...process.env,
    // Asegurar que Python encuentre el módulo
    PYTHONPATH: path.join(REPO_DIR, "src")
  }
});
```

**Razones de los cambios:**

1. **Soporte para `uv run`:** Ejecuta Python con el entorno virtual correcto automáticamente sin necesidad de activarlo explícitamente
2. **PYTHONPATH:** Asegura que Python encuentre el módulo `prevalidador` incluso si no está instalado con `pip install -e .`
3. **Logs mejorados:** Muestra el comando exacto que se ejecuta y el directorio de trabajo para debugging

---

### 8. Validación de archivos recibidos

#### Versión Desarrollo (Original)
```javascript
// No existe esta validación
```

#### Versión Producción (Adaptada)
```javascript
if (!req.files || req.files.length === 0) {
  throw new Error("No se recibieron archivos");
}

console.log(`📁 [${jobId}] Archivos recibidos: ${req.files.length}`);
req.files.forEach(f => console.log(`   - ${f.originalname} (${(f.size/1024).toFixed(2)} KB)`));
```

**Razón del cambio:**

Previene procesamiento innecesario cuando no se suben archivos y proporciona información detallada en logs para auditoría y debugging.

---

### 9. URL de descarga con prefijo

#### Versión Desarrollo (Original)
```javascript
res.json({ 
  ok: true, 
  jobId, 
  downloadUrl: `/api/download/${jobId}` 
});
```

#### Versión Producción (Adaptada)
```javascript
res.json({ 
  ok: true, 
  jobId, 
  downloadUrl: `/validador/api/download/${jobId}`,
  message: "Validación completada exitosamente",
  filesProcessed: req.files.length
});
```

**Razón del cambio:**

Similar al cambio en `app.js`, la URL debe incluir el prefijo `/validador/api/` para que Nginx la intercepte correctamente. Además, agrega información adicional en la respuesta (mensaje y contador de archivos).

---

### 10. Escuchar en todas las interfaces de red

#### Versión Desarrollo (Original)
```javascript
app.listen(PORT, () => {
  console.log(`✅ Servidor escuchando en http://localhost:${PORT}`);
});
```

#### Versión Producción (Adaptada)
```javascript
app.listen(PORT, '0.0.0.0', () => {
  console.log(`
╔════════════════════════════════════════════╗
║   ✅ VALIDADOR GIS v1.2 - INICIADO        ║
╠════════════════════════════════════════════╣
║  Puerto:        ${PORT}                        ║
║  Host:          0.0.0.0                    ║
║  Python:        ${PYTHON_BIN.padEnd(28)}║
║  Rama:          ${REPO_BRANCH.padEnd(28)}║
║  Directorio:    ${__dirname.substring(__dirname.length - 28).padEnd(28)}║
╠════════════════════════════════════════════╣
║  URLs:                                     ║
║  • http://localhost:${PORT}/api/health      ║
║  • http://localhost:${PORT}/api/validate    ║
╚════════════════════════════════════════════╝
  `);
});
```

**Razones de los cambios:**

1. **`'0.0.0.0'` en lugar de omitir el parámetro:**
   - Por defecto, Node.js escucha solo en `localhost` (127.0.0.1)
   - Nginx hace peticiones desde el host, no desde localhost estricto
   - `0.0.0.0` hace que el servidor escuche en TODAS las interfaces de red
   - Permite conexiones desde Nginx, Docker, u otros servicios del mismo servidor

2. **Banner mejorado:**
   - Información visual clara del estado del servidor
   - Muestra configuración importante (Python, rama, directorio)
   - Facilita verificación rápida de que todo inició correctamente

---

### 11. Health check mejorado

#### Versión Desarrollo (Original)
```javascript
app.get("/api/health", (_, res) => res.json({ ok: true }));
```

#### Versión Producción (Adaptada)
```javascript
app.get("/api/health", (_, res) => {
  res.json({ 
    ok: true,
    service: "validador",
    version: "1.2",
    timestamp: new Date().toISOString(),
    uptime: Math.floor(process.uptime()),
    config: {
      port: PORT,
      pythonBin: PYTHON_BIN,
      repoUrl: REPO_URL ? "configured" : "not configured",
      repoBranch: REPO_BRANCH
    }
  });
});
```

**Razón del cambio:**

Proporciona información útil para monitoreo:
- Verificar qué versión está corriendo
- Tiempo de actividad (uptime) para detectar reinicios inesperados
- Configuración actual para validar que está usando los parámetros correctos
- Se puede integrar con sistemas de monitoreo externos

---

### 12. Logging estructurado

#### Versión Desarrollo (Original)
```javascript
console.log("▶️ Ejecutando: ...");
// Logs básicos
```

#### Versión Producción (Adaptada)
```javascript
console.log(`\n${"=".repeat(60)}`);
console.log(`🔄 [${jobId}] Nueva validación iniciada`);
console.log(`${"=".repeat(60)}`);
console.log(`📁 [${jobId}] Archivos recibidos: ${req.files.length}`);
req.files.forEach(f => console.log(`   - ${f.originalname} (${(f.size/1024).toFixed(2)} KB)`));
// ... más logs con prefijo [jobId]
console.log(`${"=".repeat(60)}`);
console.log(`✅ [${jobId}] VALIDACIÓN COMPLETADA`);
console.log(`${"=".repeat(60)}\n`);
```

**Razón del cambio:**

- **JobId en cada log:** Permite seguir el ciclo de vida de una petición específica cuando hay múltiples validaciones concurrentes
- **Separadores visuales:** Facilita encontrar inicio y fin de cada validación en logs largos
- **Emojis como indicadores:** Rápida identificación visual del tipo de log (📁 archivos, ✅ éxito, ❌ error, 🐍 Python, etc.)
- **PM2 logs:** PM2 agrega timestamp automáticamente, el jobId permite correlacionar logs de diferentes momentos

Ejemplo de log en producción:
```
2025-09-30 10:30:15: ============================================================
2025-09-30 10:30:15: 🔄 [abc123] Nueva validación iniciada
2025-09-30 10:30:15: 📁 [abc123] Archivos recibidos: 2
2025-09-30 10:30:15:    - parcelas.xlsx (245.67 KB)
2025-09-30 10:30:15:    - edificaciones.xlsx (189.23 KB)
2025-09-30 10:30:16: 💾 [abc123] Guardando backup de entrada...
2025-09-30 10:30:18: 🐍 [abc123] Ejecutando prevalidador...
2025-09-30 10:31:42: ✅ [abc123] Prevalidador completado
2025-09-30 10:31:45: 📦 [abc123] ZIP creado
2025-09-30 10:31:45: ✅ [abc123] VALIDACIÓN COMPLETADA
```

---

### 13. Manejo de señales de terminación

#### Versión Desarrollo (Original)
```javascript
// No existe
```

#### Versión Producción (Adaptada)
```javascript
process.on('SIGTERM', () => {
  console.log('\n🛑 SIGTERM recibido, cerrando servidor...');
  process.exit(0);
});

process.on('SIGINT', () => {
  console.log('\n🛑 SIGINT recibido, cerrando servidor...');
  process.exit(0);
});
```

**Razón del cambio:**

PM2, Docker y systemd envían señales SIGTERM/SIGINT para shutdown graceful:
- **SIGTERM:** Señal estándar de terminación (ej: `pm2 stop`, `systemctl stop`)
- **SIGINT:** Ctrl+C en terminal

Sin estos handlers:
- El proceso puede quedar en estado zombie
- PM2 tiene que hacer kill forzado (-9) después del timeout
- Logs no muestran shutdown intencional vs crash

Con estos handlers:
- Cierre limpio y controlado
- PM2 puede hacer reload sin downtime
- Logs claros de por qué se detuvo el servicio

---

## Resumen de Cambios por Categoría

### Compatibilidad Multiplataforma
- Detección automática de Python (py vs python3)
- Soporte para uv (Linux) y pip (Windows)
- Rutas de venv multiplataforma

### Integración con Nginx
- Prefijos de URL `/validador/api/` en todas las rutas
- CORS configurado para dominio específico
- Escuchar en 0.0.0.0 en lugar de localhost

### Robustez y Producción
- Validación de entrada de archivos
- Manejo de errores mejorado con try-catch
- Git reset --hard para evitar conflictos
- Límites explícitos de upload
- Creación de directorios con verificación

### Monitoreo y Debugging
- Logging estructurado con jobId
- Health check con información detallada
- Manejo de señales de terminación
- Banner informativo al inicio

### Optimización
- Soporte para uv (10-100x más rápido que pip)
- PYTHONPATH configurado correctamente
- Logs más informativos para troubleshooting

---

## Notas de Migración

Si un desarrollador quiere ejecutar la versión de producción en su máquina de desarrollo:

1. Las rutas `/validador/api/` funcionarán si accede a `http://localhost:3009/validador/`
2. Puede revertir a rutas relativas `/api/` para desarrollo local editando `app.js`
3. El código detecta automáticamente el SO (Windows/Linux)
4. uv es opcional - si no está instalado, usa pip automáticamente

Si el servidor necesita volver a la versión de desarrollo:

1. Cambiar rama: `git checkout version-2`
2. Editar `app.js`: remover prefijo `/validador` de las URLs
3. Editar nginx: cambiar `location /validador/` para que haga proxy completo
4. Reiniciar PM2: `pm2 restart validador-server`

---

## Autores

**Desarrollo Original:** Pablo Aranzazu (@paaranzazuv)  
**Adaptaciones Servidor:** David Vasquez (@sparveriusdev) - Grupo T.I Insumosgeo  
**Fecha:** 30 de septiembre de 2025