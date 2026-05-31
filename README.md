# Thermal Printer Server

Un servidor de impresión ligero, robusto y de alto rendimiento optimizado para el entorno Windows. Este proyecto permite detectar impresoras, comprobar su accesibilidad técnica y enviar secuencias complejas de comandos de impresión térmica (ESC/POS) y texto plano.

## 🚀 Características Principales

*   **Detección Avanzada de Dispositivos:** Identifica impresoras en cola de impresión (Spooler) y dispositivos USB unificados.
*   **Prueba de Conexión Dinámica (Probing):** Diagnóstico Win32 para verificar el estado de comunicación del hardware antes de imprimir.
*   **Motor de Renderizado ESC/POS:** Transformación de una lista estructurada de operaciones en formato JSON a secuencias de bytes ESC/POS (alineación, negrita, tamaño, tablas, imágenes).
*   **Renderizado en Texto Plano:** Previsualización del layout del ticket sin impresora física.
*   **Autónomo:** Binario compilado que no requiere runtimes externos.

---

## ⚙️ Descarga e Instalación

### 1. Requisitos Previos
*   **Windows OS** (requerido para APIs específicas de impresión).

### 2. Descarga
Descarga la última versión estable desde la sección de [Releases](https://github.com/mateopautasso/thermal_printer_server_releases/releases). Encontrarás un archivo `.zip` (ej. `thermal_printer_server-vX.Y.Z-win-x64.zip`).

### 3. Instalación como Servicio (Inicio Automático)
Para que el servidor se inicie automáticamente de forma **invisible** (sin ventana) al arrancar Windows:

1. Extrae el archivo `.zip` en una carpeta permanente (ej. `C:\ThermalPrinterServer`).
2. Haz doble clic en el archivo **`manage.bat`**. Solicitará permisos de administrador.
3. En el **Manager Interactivo**, selecciona **"Instalar servicio"**.
   * Te solicitará el puerto (por defecto `8001`).
4. Si necesitas acceder al servidor desde otros equipos de tu red local, selecciona la opción **"Abrir puerto en firewall"** dentro del manager.

---

## 🔌 Documentación de la API HTTP

Por defecto, el servidor se ejecuta en el puerto **`8001`**. Todos los endpoints (excepto `/api/status`) devuelven las respuestas estructuradas bajo un sobre JSON común.

### Formato del Sobre de Respuesta (JSON Envelope)

*   **Respuesta Exitosa:**
    ```json
    {
      "success": true,
      "data": {
        // Datos de la respuesta
      }
    }
    ```
*   **Respuesta de Error:**
    ```json
    {
      "success": false,
      "error": {
        "message": "Descripción del error",
        "code": "ERROR_CODE"
      }
    }
    ```

---

### 1. Estado del Servidor
Verifica si el servidor está en ejecución. Este endpoint no utiliza sobre JSON.

*   **URL:** `/api/status`
*   **Método:** `GET`
*   **Respuesta:** `200 OK` (Cuerpo: `OK`)

---

### 2. Listar Impresoras
Obtiene la lista de impresoras físicas y virtuales detectadas.

*   **URL:** `/api/printers`
*   **Método:** `GET`

---

### 3. Comprobar Accesibilidad de Impresora
Diagnóstico de bajo nivel para verificar accesibilidad.

*   **URL:** `/api/printers/check`
*   **Método:** `GET`
*   **Parámetros:** `printerName` (string), `sourceType` (string).
*   **Ejemplo:** `/api/printers/check?printerName=POS-80&sourceType=Spooler`

---

### 4. Renderizar Ticket en Texto Plano
Previsualiza el ticket como texto.

*   **URL:** `/api/tickets/render/plain`
*   **Método:** `POST`
*   **Cuerpo (JSON):**
    ```json
    {
      "paperWidth": 42,
      "operations": [
        { "action": "align", "arguments": "center" },
        { "action": "text", "arguments": "MI COMPAÑÍA" }
      ]
    }
    ```

---

### 5. Imprimir Ticket
Envía las operaciones a una impresora física.

*   **URL:** `/api/tickets/print`
*   **Método:** `POST`
*   **Cuerpo (JSON):**
    ```json
    {
      "printerName": "POS-80",
      "sourceType": "Spooler",
      "operations": [
        { "action": "align", "arguments": "center" },
        { "action": "text", "arguments": "MI COMPAÑÍA" },
        { "action": "cut", "arguments": null }
      ]
    }
    ```

---

## 🎫 Detalle del Formato de Operaciones de Ticket

Las operaciones en `operations` definen el diseño del comprobante.

1. **`text`**: Escribe una cadena y realiza un salto de línea. Arg: `string`.
2. **`line`**: Dibuja una línea separadora (`-`). Arg: `null`.
3. **`feed`**: Avanza el papel. Arg: `number`.
4. **`font_size`**: Tamaño de fuente. Arg: `1 | 2 | 3`.
5. **`bold`**: Negrita. Arg: `boolean`.
6. **`align`**: Alineación. Arg: `'left' | 'center' | 'right'`.
7. **`table`**: Genera una tabla auto-ajustable. Arg: objeto con `content` (array de arrays) y opcionalmente `align`.
8. **`image`**: Imagen monocromática. Arg: `string` (base64).
9. **`codepage`**: Cambia la página de códigos. Arg: `string` (ej. `cp858`).
10. **`cut`**: Corte de papel. Arg: `null`.
