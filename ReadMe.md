# RPA Análisis de Productos (PIX RPA)

Automatización integral que consume una API pública, persiste datos en BD, genera reportes en Excel, integra con OneDrive (Microsoft Graph) y entrega el reporte via formulario web.

---

## Descripción del Proyecto
Este bot RPA, construido sobre la **Plantilla Universal de PIX RPA**, ejecuta un flujo diario para una tienda online ficticia:
1. **Obtención de datos** desde la API pública *Fake Store API* (`/products`).
2. **Respaldos**: guarda la respuesta bruta en `.json` y la sube a OneDrive.
3. **Normalización + Almacenamiento** en una **Base de Datos** (evita duplicados por `id`).
4. **Reporte en Excel** con dos hojas: *Productos* y *Resumen* (estadísticas globales y por categoría).
5. **Carga a OneDrive** del reporte generado.
6. **Entrega por formulario web** (Google Forms/Jotform/Typeform) adjuntando el Excel.
7. **Evidencias** (logs y captura de confirmación del formulario).

> **Objetivo**: Orquestar un proceso de extremo a extremo, sin intervención del usuario, con autenticación **client credentials** frente a **Microsoft Graph**.

---

## Arquitectura Lógica
- **Fuente de datos**: [Fake Store API](https://fakestoreapi.com/docs#tag/Products)
- **RPA**: PIX Studio / Plantilla Universal PIX RPA
- **Persistencia**: MySQL Server. (archivo local)
- **Reportería**: Excel (`.xlsx`) con estadísticas
- **Nube**: OneDrive (Microsoft Graph API)
- **Entrega**: Formulario web con campos requeridos + adjunto https://forms.gle/tWJrNtVDWoY1QcXGA

---

## Requisitos / Dependencias

### 1 Accesos y credenciales
- **Azure AD App Registration** (aplicación confidencial) para **Microsoft Graph**
  - `TENANT_ID`
  - `CLIENT_ID`
  - Permisos de aplicación para OneDrive/SharePoint (ej.: `Files.ReadWrite.All`)
- **OneDrive** del usuario/tenant donde se almacenarán:
  - `RPA/Logs/`  (JSON de respaldo)
  - `RPA/Reportes/`  (Excel generado)

### 2 Entorno RPA
- **PIX Studio** instalado
- Plantilla Universal de **PIX RPA** (proyecto base)

### 3 Base de datos
- **MySQL Server**

### 4 Librerías / Paquetes (según implementación del bot)
- Cliente HTTP (para `GET https://fakestoreapi.com/products`)
- Driver de BD (ODBC)
- Generación de Excel (`.xlsx`)
- Cliente para **Microsoft Graph** (HTTP) con **Client Credentials**
- Chrome Driver

> **Nota**: Todas las dependencias ya están integradas en el bot entregado. Este README documenta su uso y configuración.

---

## Estructura de Carpetas (local)
```
/Reportes/                  ← Reporte_YYYY-MM-DD.xlsx
/Evidencias/                ← formulario_confirmacion.jpg
/Descargas/                 ← Productos_YYYY-MM-DD.json
```

En **OneDrive**:
```
/RPA/Logs/Productos_YYYY-MM-DD.json
/RPA/Reportes/Reporte_YYYY-MM-DD.xlsx
```

---

## Esquema de Base de Datos
**Tabla**: `Productos`

Campos:
- `id` (entero, **PRIMARY KEY**)
- `title` (texto)
- `price` (decimal)
- `category` (texto)
- `description` (texto)
- `fecha_insercion` (timestamp)

**Regla**: Evitar duplicados. Antes de insertar, validar que `id` no exista.

> Estructura:
```sql
CREATE TABLE `productos` (
  `id` int NOT NULL AUTO_INCREMENT,
  `title` text NOT NULL,
  `price` decimal(10,2) NOT NULL,
  `category` text,
  `description` text,
  `fecha_insercion` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1235 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

---

## Pasos para Ejecución

### 0 Configuración inicial
1. **Variables/Assets de Configuración en PIX**:
   - `FAKESTORE_API_URL` = `https://fakestoreapi.com/products`
   - `GRAPH_TENANT_ID` = `00000000-0000-0000-0000-000000000000`
   - `GRAPH_CLIENT_ID` = `11111111-1111-1111-1111-111111111111`
   - `GRAPH_CLIENT_SECRET` = `****`
   - `ONEDRIVE_BASE_PATH` = `/RPA`  
   - `ONEDRIVE_LOGS_PATH` = `/RPA/Logs`  
   - `ONEDRIVE_REPORTES_PATH` = `/RPA/Reportes`
   - `RUTA_LOCAL_LOGS` = `./Logs`
   - `RUTA_LOCAL_REPORTES` = `./Reportes`
   - `RUTA_LOCAL_DESCARGAS` = `./Descargas`
   - `RUTA_LOCAL_EVIDENCIAS` = `./Evidencias`
   - `DB_PROVIDER` = `SQLite`  (o `PostgreSQL`/`SQLServer`)
   - `DB_CONNECTION` = `./datos.db` (SQLite)  
     *(o cadena de conexión correspondiente para el motor elegido)*
   - `FORM_URL` = `<<Pega aquí tu enlace de formulario>>`
   - `FORM_COLABORADOR` = `Nombre Apellido` (valor por defecto)

2. **Permisos Graph** (app registration): conceder *admin consent* a los permisos requeridos.

3. **Rutas** locales: no es necesario crearlas manualmente; el bot las valida/crea si no existen.

---

### 1 Descarga y respaldo
- `GET ${FAKESTORE_API_URL}`
- Guardar respuesta como `./Descargas/Productos_YYYY-MM-DD.json`
- Subir a OneDrive → `${ONEDRIVE_LOGS_PATH}/Productos_YYYY-MM-DD.json`

### 2 Transformación + Upsert en BD
- Proyectar campos: `id, title, price, category, description`
- Insertar evitando duplicados por `id`
- Registrar `fecha_insercion` = fecha/hora de ejecución

### 3 Generación del Excel
- **Archivo**: `./Reportes/Reporte_YYYY-MM-DD.xlsx`
- **Hoja 1 – Productos**: dump completo de la tabla `Productos`
- **Hoja 2 – Resumen**:
  - Total de productos
  - Precio promedio general
  - Precio promedio por categoría
  - Cantidad de productos por categoría

### 4 Carga del Excel a OneDrive
- Subir a `${ONEDRIVE_REPORTES_PATH}/Reporte_YYYY-MM-DD.xlsx`
- Si existe, **sobrescribe** o crea **nueva versión** (según implementación)

### 5 Entrega por Formulario Web
- Navegar a `${FORM_URL}`
- Completar campos:
  - **Nombre del colaborador**: `${FORM_COLABORADOR}` (o el que indique el orquestador)
  - **Fecha de generación**: fecha del reporte
  - **Archivo**: adjuntar `Reporte_YYYY-MM-DD.xlsx`
- Enviar
- Capturar confirmación y guardar como `./Evidencias/formulario_confirmacion.png`

---

## Autenticación con Microsoft Graph (Client Credentials)
1. Obtener **token**: `POST https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token`  
   **Body (x-www-form-urlencoded)**:
   - `client_id={CLIENT_ID}`
   - `grant_type=client_credentials`
   - `scope=Files.ReadWrite`

2. Carga de archivos a OneDrive (usuario):
   - `PUT https://graph.microsoft.com/v1.0/me/drive/root:{path}:/content`
   - Body binario del archivo (JSON o XLSX)

---

## Validación / Pruebas
- Ejecutar el bot en ambiente de prueba
- Confirmar:
  - JSON en `OneDrive:/RPA/Logs/`
  - Excel en `OneDrive:/RPA/Reportes/`
  - Envío y confirmación del formulario (captura en `/Evidencias/`)
  - Tabla `Productos` con datos sin duplicados

---

## 📎 Enlace del Formulario Usado
Pega aquí el enlace real del formulario que emplea tu bot:

**Formulario**: `<<https://forms.gle/tWJrNtVDWoY1QcXGA>>`

## Licencia
Uso interno / educativo.

