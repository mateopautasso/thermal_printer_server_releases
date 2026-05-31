# Thermal Printer Server

Un servidor de impresión ligero, robusto y de alto rendimiento diseñado en **TypeScript** sobre el runtime **Bun**, optimizado para el entorno Windows. Este proyecto permite detectar impresoras, comprobar su accesibilidad técnica y enviar secuencias complejas de comandos de impresión térmica (ESC/POS) y texto plano.

## 🚀 Características Principales

*   **Detección Avanzada de Dispositivos:** Integración con WMI mediante un binario en C# para identificar impresoras en cola de impresión (Spooler) y candidatos raw USB de manera unificada.
*   **Prueba de Conexión Dinámica (Probing):** Diagnóstico a nivel Win32 (Spooler API, USB Handle y puertos serie) para verificar el estado de comunicación del hardware antes de imprimir.
*   **Motor de Renderizado ESC/POS:** Transformación de una lista estructurada de operaciones en formato JSON a secuencias de bytes ESC/POS válidas (soporte de anchos de papel, alineación, negrita, tamaño de fuente, tablas auto-ajustables, conversión de codepage CP858, imágenes base64).
*   **Renderizado en Texto Plano:** Endpoint dedicado para previsualizar el layout del ticket como texto plano sin necesidad de una impresora física.
*   **Completamente Autónomo:** No requiere la instalación de runtimes o SDKs en la máquina de destino, se distribuye como un ejecutable autocontenido.

---

## ⚙️ Descarga e Instalación

### 1. Requisitos Previos
*   **Windows OS** (debido al uso de APIs específicas de Windows y binarios C# auxiliares integrados).

### 2. Descarga
Dirígete a la sección de [Releases](../../releases) de este repositorio y descarga el archivo comprimido `.zip` de la última versión disponible (ej. `thermal_printer_server-vX.Y.Z-win-x64.zip`).

### 3. Instalación como Servicio (Inicio Automático)
Para que el servidor se inicie automáticamente de forma **invisible** (sin ventana de consola) al encender el equipo, se incluye un Manager Interactivo:

1. Extrae el contenido del archivo `.zip` en una ubicación permanente (por ejemplo, `C:\ThermalPrinterServer`).
2. Haz doble clic en el archivo **`manage.bat`**. (Solicitará permisos de Administrador).
3. Utiliza el menú numérico para seleccionar **Instalar servicio**. Te preguntará qué puerto deseas utilizar (por defecto `8001`). Se creará una Tarea Programada de Windows que corre como `SYSTEM`.
4. Si el servidor va a recibir peticiones desde *otros equipos de la red* (no solo desde `localhost`), selecciona la opción **Abrir puerto en firewall** dentro del manager.

---

## 🔌 Documentación de la API HTTP

Por defecto, el servidor se ejecuta en el puerto **`8001`**. Todos los endpoints (excepto `/api/status`) devuelven las respuestas estructuradas bajo un sobre JSON común.

### Formato del Sobre de Respuesta (JSON Envelope)

*   **Respuesta Exitosa:**
    ```json
    {
      "success": true,
      "data": {
        // Objeto con los datos específicos de la respuesta
      }
    }
    ```
*   **Respuesta de Error:**
    ```json
    {
      "success": false,
      "error": {
        "message": "Descripción detallada del error ocurrido",
        "code": "CÓDIGO_DE_ERROR_IDENTIFICADOR"
      }
    }
    ```

---

### 1. Estado del Servidor
Verifica si el servidor está en ejecución. Este endpoint no utiliza sobre JSON para maximizar la velocidad y simplificar el monitoreo de salud.

*   **URL:** `/api/status`
*   **Método:** `GET`
*   **CORS:** Soporta peticiones preflight (`OPTIONS`).
*   **Respuesta Exitosa:**
    *   **Código HTTP:** `200 OK`
    *   **Tipo de Contenido:** `text/plain`
    *   **Cuerpo:** `OK`

---

### 2. Listar Impresoras
Obtiene la lista de impresoras físicas y virtuales detectadas en el sistema operativo Windows mediante consultas WMI.

*   **URL:** `/api/printers`
*   **Método:** `GET`
*   **Respuesta Exitosa:**
    *   **Código HTTP:** `200 OK`
    *   **Cuerpo:**
        ```json
        {
          "success": true,
          "data": {
            "printers": [
              {
                "id": "Microsoft Print to PDF",
                "displayName": "Microsoft Print to PDF",
                "sourceType": "Spooler",
                "status": "Idle",
                "isDefault": false,
                "driverName": "Microsoft Plotter Driver",
                "portName": "PORTPROMPT:",
                "deviceInstanceId": "SWD\\PRINTRAW\\MICROSOFT_PRINT_TO_PDF",
                "manufacturer": "Microsoft",
                "hardwareIds": [
                  "LPTENUM\\MicrosoftMicrosoft_P415D"
                ],
                "description": "Impresora virtual PDF"
              }
            ]
          }
        }
        ```
*   **Respuesta de Error:**
    *   **Código HTTP:** `500 Internal Server Error`
    *   **Cuerpo:**
        ```json
        {
          "success": false,
          "error": {
            "message": "Failed to parse printers JSON output...",
            "code": "PRINTER_DETECTION_ERROR"
          }
        }
        ```

---

### 3. Comprobar Accesibilidad de Impresora
Realiza un diagnóstico de bajo nivel para verificar si una impresora está conectada y accesible para recibir datos.

*   **URL:** `/api/printers/check`
*   **Método:** `GET`
*   **Parámetros de Consulta (Query Params):**
    *   `printerName` (Requerido, string): El nombre exacto de la impresora en el sistema (por ejemplo, `Microsoft Print to PDF` o `impresora-termica`).
    *   `sourceType` (Requerido, string): Canal de comunicación. Valores típicos: `Spooler`, `USB`, `Serial`.
*   **Ejemplo de Petición:** `/api/printers/check?printerName=Microsoft+Print+to+PDF&sourceType=Spooler`
*   **Respuesta Exitosa:**
    *   **Código HTTP:** `200 OK`
    *   **Cuerpo:**
        ```json
        {
          "success": true,
          "data": {
            "accessible": true,
            "success": true,
            "probeLevel": "OpenPrinter",
            "printerAccessible": true,
            "openPrinterSucceeded": true,
            "startDocSucceeded": false,
            "writeSucceeded": false,
            "bytesWritten": 0,
            "win32ErrorCode": null,
            "message": "Printer is accessible via OpenPrinter API."
          }
        }
        ```
*   **Respuestas de Error:**
    *   **Falta de parámetros (`400 Bad Request`):**
        ```json
        {
          "success": false,
          "error": {
            "message": "printerName is required and must be a string",
            "code": "VALIDATION_ERROR"
          }
        }
        ```
    *   **Fallo interno del probe (`500 Internal Server Error`):**
        ```json
        {
          "success": false,
          "error": {
            "message": "Failed to parse probe JSON output...",
            "code": "PRINTER_CHECK_ERROR"
          }
        }
        ```

---

### 4. Renderizar Ticket en Texto Plano
Renderiza una lista de operaciones de ticket como texto plano, sin enviar nada a una impresora física. Útil para previsualizar el layout del comprobante desde el cliente antes de imprimir.

*   **URL:** `/api/tickets/render/plain`
*   **Método:** `POST`
*   **Tipo de Contenido:** `application/json`
*   **Cuerpo de la Petición:**
    *   `operations` (Requerido, array de objetos): Lista de comandos de diseño del ticket.
    *   `paperWidth` (Opcional, number): Ancho del papel en caracteres. Usa el valor por defecto si se omite.
*   **Ejemplo de Petición (Cuerpo JSON):**
    ```json
    {
      "paperWidth": 42,
      "operations": [
        { "action": "align", "arguments": "center" },
        { "action": "bold", "arguments": true },
        { "action": "text", "arguments": "MI COMPAÑÍA" },
        { "action": "bold", "arguments": false },
        { "action": "line", "arguments": null },
        { "action": "text", "arguments": "1     Café Espresso   $1.200" },
        { "action": "feed", "arguments": 2 },
        { "action": "cut", "arguments": null }
      ]
    }
    ```
*   **Respuesta Exitosa:**
    *   **Código HTTP:** `200 OK`
    *   **Cuerpo:**
        ```json
        {
          "success": true,
          "data": {
            "ticket": "           MI COMPAÑÍA\n------------------------------------------\n1     Café Espresso   $1.200\n"
          }
        }
        ```
*   **Respuestas de Error:**
    *   **Error de Validación (`400 Bad Request`):**
        ```json
        {
          "success": false,
          "error": {
            "message": "Operation[2]: \"font_size\" arguments must be 1, 2, or 3",
            "code": "VALIDATION_ERROR"
          }
        }
        ```
    *   **Error interno (`500 Internal Server Error`):**
        ```json
        {
          "success": false,
          "error": {
            "message": "...",
            "code": "RENDER_ERROR"
          }
        }
        ```

---

### 5. Imprimir Ticket
Envía una serie de operaciones de formato y contenido estructurado a una impresora específica. La API procesa las operaciones, las compila a bytes ESC/POS y las envía al hardware mediante el binario `ThermalPrinterWriter.Cli.exe`.

*   **URL:** `/api/tickets/print`
*   **Método:** `POST`
*   **Tipo de Contenido:** `application/json`
*   **Cuerpo de la Petición:**
    *   `printerName` (Requerido, string): Nombre del dispositivo.
    *   `sourceType` (Requerido, string): Tipo de conexión (`Spooler`, `USB`, etc.).
    *   `operations` (Requerido, array de objetos): Lista de comandos de diseño del ticket.
*   **Ejemplo de Petición (Cuerpo JSON):**
    ```json
    {
      "printerName": "impresora-termica-forcom",
      "sourceType": "Spooler",
      "operations": [
        { "action": "align", "arguments": "center" },
        { "action": "bold", "arguments": true },
        { "action": "font_size", "arguments": 2 },
        { "action": "text", "arguments": "MI COMPAÑÍA" },
        { "action": "font_size", "arguments": 1 },
        { "action": "bold", "arguments": false },
        { "action": "line", "arguments": null },
        { "action": "text", "arguments": "Cant  Producto        Total" },
        { "action": "line", "arguments": null },
        { "action": "text", "arguments": "1     Café Espresso   $1.200" },
        { "action": "line", "arguments": null },
        { "action": "feed", "arguments": 3 },
        { "action": "cut", "arguments": null }
      ]
    }
    ```
*   **Respuesta Exitosa:**
    *   **Código HTTP:** `200 OK`
    *   **Cuerpo:**
        ```json
        {
          "success": true,
          "data": {
            "printed": true
          }
        }
        ```
*   **Respuestas de Error:**
    *   **Error de Validación (`400 Bad Request`):** Si falta algún campo obligatorio o si una de las operaciones no cumple con los formatos de argumento esperados.
        ```json
        {
          "success": false,
          "error": {
            "message": "Operation[3]: \"font_size\" arguments must be 1, 2, or 3",
            "code": "VALIDATION_ERROR"
          }
        }
        ```
    *   **Error del Sistema de Impresión (`500 Internal Server Error`):**
        ```json
        {
          "success": false,
          "error": {
            "message": "Print failed: OpenPrinter failed. Win32 error: 1801.",
            "code": "PRINT_ERROR"
          }
        }
        ```

---

## 🎫 Detalle del Formato de Operaciones de Ticket

Las operaciones enviadas en el arreglo `operations` definen visualmente el diseño del comprobante. Cada operación debe tener la propiedad `action` (tipo de comando) y `arguments` (parámetros del comando).

A continuación se detallan todas las operaciones disponibles:

### 1. `text`
Escribe una cadena de caracteres en la línea actual y realiza un salto de línea (`LF`). Realiza traducción automática de caracteres unicode del idioma español a codificación de página de códigos de la impresora (por defecto CP858).
*   **Argumentos:** `string` (Cadena de texto a imprimir).
*   **Ejemplo:** `{ "action": "text", "arguments": "¡Hola, mundo español Ñandú!" }`

### 2. `line`
Dibuja una línea horizontal separadora compuesta por guiones (`-`). El número de caracteres se adapta automáticamente al ancho del papel y al tamaño actual de la fuente de la impresora.
*   **Argumentos:** `null`
*   **Ejemplo:** `{ "action": "line", "arguments": null }`

### 3. `feed`
Avanza el papel de impresión el número especificado de líneas. Útil para dar margen al finalizar el ticket o antes del corte físico.
*   **Argumentos:** `number` (Entero positivo >= 1).
*   **Ejemplo:** `{ "action": "feed", "arguments": 3 }`

### 4. `font_size`
Establece el tamaño multiplicador de la fuente para las siguientes operaciones de texto.
*   **Argumentos:** `1 | 2 | 3`
    *   `1`: Tamaño normal (1x de ancho y alto).
    *   `2`: Tamaño doble (2x de ancho y alto).
    *   `3`: Tamaño triple (3x de ancho y alto).
*   **Ejemplo:** `{ "action": "font_size", "arguments": 2 }`

### 5. `bold`
Activa o desactiva el formato en negrita (bold) en la impresora térmica para los siguientes textos.
*   **Argumentos:** `boolean` (`true` para activar, `false` para desactivar).
*   **Ejemplo:** `{ "action": "bold", "arguments": true }`

### 6. `align`
Modifica la alineación del texto impreso a partir de ese momento.
*   **Argumentos:** `'left' | 'center' | 'right'` (Alineación a la izquierda, centrada o derecha).
*   **Ejemplo:** `{ "action": "align", "arguments": "center" }`

### 7. `table`
Genera una estructura de cuadrícula o tabla con alineación por columna. Las columnas se auto-dimensionan y auto-ajustan proporcionalmente según el ancho de los datos de entrada para evitar desbordes físicos de línea.
*   **Argumentos:** Un objeto que contiene:
    *   `content` (Requerido, array de arrays de strings): Una lista de filas, donde cada fila es una lista de celdas en formato string. La primera fila se asume como encabezado de tabla y el renderizador le añade una línea divisoria debajo de forma automática.
    *   `align` (Opcional, array de strings): Define la alineación para cada columna. Valores aceptados por elemento: `'left'`, `'center'`, `'right'`.
*   **Ejemplo:**
    ```json
    {
      "action": "table",
      "arguments": {
        "content": [
          ["Producto", "Cant", "Total"],
          ["Café Doble", "2", "$2.400"],
          ["Medialuna", "3", "$1.500"]
        ],
        "align": ["left", "center", "right"]
      }
    }
    ```

### 8. `image`
Envía una imagen binaria en formato base64 que se interpretará como mapa de bits monocromático en la impresora térmica.
*   **Argumentos:** `string` (Cadena en base64 no vacía que representa los datos de imagen).
*   **Ejemplo:** `{ "action": "image", "arguments": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=" }`

### 9. `codepage`
Establece la página de códigos a utilizar en la impresora física mediante el comando `ESC t`. Esto modifica la tabla de caracteres internos del hardware.
*   **Argumentos:** `string`. Valores comunes soportados en mapeo interno: `cp437`, `cp850`, `cp858`, `cp860`, `cp863`, `cp865`, `cp1252`.
*   **Ejemplo:** `{ "action": "codepage", "arguments": "cp858" }`

### 10. `cut`
Envía la instrucción física de corte de papel (full cut) a la impresora.
*   **Argumentos:** `null`
*   **Ejemplo:** `{ "action": "cut", "arguments": null }`

---
