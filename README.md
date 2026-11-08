# API SII Factronica · XML a PDF

`sii_herramientas_xmltopdf` es uno de los **24 endpoints de la suite API SII Factronica**. Su función es convertir el XML de un Documento Tributario Electrónico (DTE) en una representación PDF, incluyendo su timbre electrónico PDF417.

El proyecto admite **PHP 5.6 o PHP 8** e incluye las bibliotecas TCPDF y PDF417 necesarias para generar los archivos, sin instalación mediante Composer.

## Funcionalidad

1. Recibe un archivo XML mediante una petición HTTP `POST`.
2. Comprueba que contenga un único elemento `Documento` y los nodos `TipoDTE`, `Folio` y `TED`.
3. Guarda el XML y genera una imagen PNG del timbre PDF417.
4. Crea el PDF con los datos del emisor y receptor, el detalle, los totales y el timbre.
5. Devuelve una respuesta JSON con la URL del PDF generado.

Cada solicitud utiliza un identificador único para sus archivos XML, PNG y PDF. Por defecto se genera la versión original del documento. La plantilla incluye la leyenda `BORRADOR | WWW.FACTRONICA.CL`.

Esta herramienta convierte documentos existentes: no realiza el envío al SII ni valida la firma electrónica o la aceptación tributaria del DTE.

## Requisitos

- **PHP 5.6 o PHP 8**, según `config.php`.
- Servidor web con soporte PHP, como Apache o Nginx con PHP-FPM.
- Extensiones PHP: **DOM/libxml**, **GD** con soporte PNG y **JSON**.
- Extensión PHP **cURL** para TCPDF en PHP 8 y para el cliente incluido en `ejemplo/index.php`.
- Permisos de escritura del proceso PHP en `xml/`, `png/`, `pdf/` y en el directorio temporal de PHP.
- Comando `curl` si se desea probar el endpoint desde una terminal.

## Instalación

1. Copiar o clonar este proyecto en una carpeta del servidor web.
2. Configurar el servidor para ejecutar sus archivos con PHP 5.6 o PHP 8 y seleccionar el mismo entorno en `config.php`.
3. Desde la carpeta que contiene el `index.php` del endpoint, preparar los directorios de salida:

   ```bash
   mkdir -p xml png pdf
   chmod 775 xml png pdf
   ```

   El propietario o grupo de estas carpetas debe permitir que el usuario del servidor web escriba en ellas.

4. Revisar estos valores en el `php.ini` utilizado por el servidor:

   ```ini
   file_uploads = On
   upload_max_filesize = 10M
   post_max_size = 12M
   ```

   El endpoint admite archivos de hasta **10 MB**. `post_max_size` debe ser mayor para incluir el contenido adicional de la petición multipart. Reiniciar el servicio PHP o el servidor web después de modificar su configuración.

5. Verificar que la ruta `pdf/` se sirva por HTTP para poder abrir la URL devuelta.

En los ejemplos siguientes se supone que el proyecto está disponible en:

```text
http://localhost/sii_herramientas_xmltopdf/
```

Reemplazar esta dirección por la URL real de la instalación.

## Token de acceso

Definir `api_token` en `config.php` con un secreto sin espacios y enviar en cada petición el encabezado `Authorization: Bearer TU_TOKEN_SECRETO`. El valor inicial está vacío: la API bloquea las solicitudes con HTTP 500 hasta configurar el secreto. Con un token configurado, su ausencia o un valor incorrecto devuelve HTTP 401 antes de procesar el XML. Se compara mediante `hash_equals` y no se acepta el token por URL.

El servidor debe transmitir el encabezado Authorization a PHP. Usar HTTPS en la URL pública. Para probar con `ejemplo/index.php`, reemplazar su variable `$token` por el mismo `api_token` definido en `config.php`. El ejemplo envía el encabezado `Authorization: Bearer` y puede abrirse desde el navegador. Este ejemplo con credenciales debe mantenerse en un entorno de pruebas con acceso restringido.

## Ejecutar el endpoint

### Solicitud

| Propiedad | Valor |
| --- | --- |
| Método | `POST` |
| Ruta | `/sii_herramientas_xmltopdf/index.php` |
| Tipo de contenido | `multipart/form-data` |
| Campo de archivo | `xml` |
| Extensión del archivo | `.xml` |
| Tamaño máximo | 10 MB |

El XML debe contener un único DTE con su timbre TED. Para una primera prueba se incluye `ejemplo/dte.xml`. El generador extrae el timbre a partir de las etiquetas literales `<TED version` y `</TED>`; utilizar la estructura del archivo de ejemplo como referencia.

Desde la carpeta del proyecto:

```bash
curl -i -X POST \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer TU_TOKEN_SECRETO' \
  -F 'xml=@ejemplo/dte.xml;type=application/xml' \
  http://localhost/sii_herramientas_xmltopdf/index.php
```

cURL configura automáticamente el encabezado `Content-Type` y el separador multipart al utilizar `-F`.

### Respuesta correcta

El endpoint responde con **HTTP 201 Created** y un JSON como este:

```json
{
  "success": true,
  "url_pdf": "http://localhost/sii_herramientas_xmltopdf/pdf/dte_IDENTIFICADOR.pdf"
}
```

El identificador es ilustrativo: cambia en cada solicitud. Abrir `url_pdf` en el navegador o descargarla para consultar el documento. La respuesta contiene la URL; el PDF se guarda en el servidor.

### Respuestas de error

| Estado HTTP | Motivo |
| --- | --- |
| `401` | Token de acceso ausente o incorrecto. |
| `400` | Falta el archivo `xml`, falla la carga o su extensión no es `.xml`. |
| `405` | Se utilizó un método distinto de `POST`. |
| `413` | El archivo supera el límite de tamaño de PHP o los 10 MB del endpoint. |
| `422` | XML inválido, presencia de `DOCTYPE` o estructura DTE requerida incompleta. |
| `500` | Error al crear directorios, guardar archivos o generar el PDF. |

Ejemplo de error:

```json
{
  "success": false,
  "error": "Debe adjuntar un archivo en el campo xml."
}
```

Si PHP rechaza toda la petición porque supera `post_max_size`, el archivo puede no aparecer en `$_FILES` y el endpoint puede responder `400` por ausencia del campo `xml`.

## Ejecutar el ejemplo PHP incluido

Con el servidor web en funcionamiento, abrir:

```text
http://localhost/sii_herramientas_xmltopdf/ejemplo/index.php
```

El ejemplo envía `ejemplo/dte.xml` al endpoint mediante cURL y muestra su respuesta JSON. Calcula automáticamente la URL del endpoint a partir del protocolo, el host y la ruta de la solicitud. Para usar otro XML, cambiar `$archivo_xml` en `ejemplo/index.php`.

El envío utilizado por el ejemplo es compatible con PHP 5.6:

```php
<?php
$url_api = 'http://localhost/sii_herramientas_xmltopdf/index.php';
$archivo_xml = __DIR__ . '/dte.xml';

$curl = curl_init($url_api);
curl_setopt_array($curl, array(
    CURLOPT_POST => true,
    CURLOPT_POSTFIELDS => array(
        'xml' => new CURLFile($archivo_xml, 'application/xml', 'dte.xml')
    ),
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER => array('Accept: application/json', 'Authorization: Bearer TU_TOKEN_SECRETO'),
    CURLOPT_CONNECTTIMEOUT => 10,
    CURLOPT_TIMEOUT => 120
));

$respuesta = curl_exec($curl);
if ($respuesta === false) {
    echo curl_error($curl);
} else {
    echo $respuesta;
}
curl_close($curl);
```

El archivo `ejemplo/index.php` incluye además controles de disponibilidad de cURL, lectura del XML, conexión y validez de la respuesta JSON. Los fallos de conexión o las respuestas que no sean JSON se informan con HTTP `502`.

## Seleccionar PHP y TCPDF

Editar `config.php` en la raíz del proyecto:

```php
<?php
return array(
    'php_version' => '5.6', // Cambiar a '8' para PHP 8.x.
    'api_token' => 'TU_TOKEN_SECRETO'
);
```

- `'5.6'` (predeterminado): carga TCPDF 6.2.4 y el generador PDF417 original.
- `'8'`: carga TCPDF 6.11.4 y su generador PDF417 oficial.

La opción selecciona las bibliotecas; **no cambia la versión de PHP del servidor**. Configurar también Apache/PHP-FPM con el intérprete correspondiente. Si la selección no coincide con el intérprete, la conversión responde HTTP 500 y registra el motivo en el log de PHP. Solo se admiten los valores de texto `'5.6'` y `'8'`.

La copia de [TCPDF 6.11.4](https://github.com/tecnickcom/TCPDF/tree/6.11.4) incluye el motor, códigos de barras y fuentes básicas (Helvetica, Courier, Times, Symbol y ZapfDingbats). Conserva su licencia LGPL en `tcpdf_6_11_4/LICENSE.TXT`. No requiere Composer. TCPDF está descontinuado por su autor; se mantiene la rama 6 para conservar la integración existente.

Conversión HTTP del XML de ejemplo verificada con PHP 5.6 y PHP 8.2. Las demás versiones PHP 8.x requieren validación en el servidor de destino.

## Configuración de la representación PDF

En el `index.php` principal se puede ajustar esta variable:

| Variable | Función |
| --- | --- |
| `$mostrar_pagina` | `1`: original (predeterminado), `2`: cedible, `3`: ambas versiones. |

Esta opción se modifica en el código; no se recibe como parámetro HTTP.

## Estructura del proyecto

```text
sii_herramientas_xmltopdf/
├── index.php          # Endpoint y plantilla del PDF
├── config.php         # Selección PHP 5.6 o PHP 8
├── ejemplo/
│   ├── index.php      # Cliente PHP de ejemplo con cURL
│   └── dte.xml        # XML para probar la conversión
├── tcpdf_6_2_4/       # Biblioteca TCPDF 6.2.4 y PDF417
├── tcpdf_6_11_4/      # Biblioteca TCPDF para PHP 8
├── src/               # Funciones auxiliares e integración del timbre
├── fonts/             # Fuentes utilizadas por TCPDF
├── xml/               # XML recibidos
├── png/               # Imágenes del timbre PDF417
└── pdf/               # PDF generados
```

Los archivos generados permanecen en sus respectivas carpetas; el endpoint no implementa eliminación automática. La URL de salida se construye con el protocolo, host y ruta que recibe PHP: estos valores deben corresponder a la dirección pública de la instalación.
