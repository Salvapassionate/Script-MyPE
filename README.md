# Script-MyPE

Script de Google Apps Script que extrae los Documentos Tributarios Electrónicos (DTE de El Salvador) que llegan por Gmail, los clasifica por empresa como **Compras** o **Ventas**, los guarda ordenados en Google Drive y registra cada archivo en una hoja de cálculo.

Archivo del script: [`Extraer DTE de correos v3.txt`](Extraer%20DTE%20de%20correos%20v3.txt) (versión 3.1).

---

## ¿Qué hace?

1. **Busca correos en todo el buzón**: recibidos, archivados y enviados, de cualquier fecha. No revisa spam, papelera, chats ni borradores. Omite los correos que ya tienen una etiqueta `DTE/…`. Recorre el buzón por páginas: si no termina en una ejecución, la siguiente continúa donde quedó.
2. **Toma los adjuntos JSON, PDF y XML**, también los que vienen dentro de un ZIP. Detecta los ZIP aunque no tengan extensión, abre los ZIP que vienen dentro de otro ZIP y descarta los archivos ocultos que agregan Mac o Windows.
3. **Clasifica cada documento por separado** según el NIT, aunque un correo o un ZIP mezcle compras, ventas y documentos de otras empresas:
   - Si la empresa es la que emite el documento, es **Venta**; si es la que lo recibe, es **Compra**.
   - En la Factura de Sujeto Excluido (tipo 14) es al revés, porque la emite quien compra.
   - Reconoce el NIT en los distintos campos que usan los emisores (`nit`, `doi`, `documento`, `numDocumento`, `id`…), con o sin guiones. También acepta DUI.
   - Lee JSON que vienen dentro de otro objeto, guardados en UTF-16 o dañados al final del archivo.
4. **Lee del documento** la fecha de emisión, el código de generación y el tipo de DTE (01 Factura, 03 CCF, 05 Nota de Crédito, 14, 15, anulaciones…).
5. **Une cada JSON con su PDF y su XML.** Lo hace en este orden: por el código de generación en el nombre del archivo, por el nombre dentro de la misma carpeta del ZIP, por el código escrito dentro del XML, o, si hay un solo JSON y un solo PDF, los une entre sí.
6. **Archivos sin su pareja:** si falta el PDF o el JSON, agrega una fila "PDF - No encontrado" o "JSON - No encontrado" y lo anota en el log de errores.
   - Un **XML sin JSON** se clasifica por su propio contenido.
   - Un **PDF sin JSON** hereda los datos si ese código ya se registró antes. Si no, se guarda en No procesado y el correo recibe la etiqueta `DTE/Solo PDF` para revisión manual. El script **no lee el contenido de los PDF** (no usa OCR), porque tomaría demasiado tiempo dentro del límite de Apps Script.
7. **Enlaces de descarga:** si el correo no trae JSON adjunto pero parece de DTE, revisa los enlaces del correo (portales tipo Walmart o Edicom) para descargarlo. Nunca abre enlaces para darse de baja ni de rastreo.
8. **Evita duplicados** con el código de generación y el formato del archivo: un documento que ya está registrado no se vuelve a subir a Drive ni a la hoja.
9. **Guarda en Drive** según la fecha de emisión del DTE. Si no tiene fecha, usa la del correo:
   ```
   Registros DTE/
   └── <Empresa>/
       ├── <Empresa> - Registros DTE   (hoja de cálculo)
       └── 2025 - <Empresa>/
           └── 03-Marzo-2025 - <Empresa>/
               ├── Compras - <Empresa>/
               └── Ventas - <Empresa>/
   ```
   Lo que no se identifica va a la carpeta y hoja `No procesado`.
10. **Pone una sola etiqueta por correo en Gmail**, dentro de `DTE/`. Las etiquetas propias del cliente no se modifican.

    | Etiqueta | Cuándo se aplica |
    |---|---|
    | `DTE/No procesado` | Algún JSON o XML del correo no se pudo asignar a una empresa (queda para revisión manual) |
    | `DTE/Solo PDF` | Lo único pendiente son PDF sin su JSON que no se pudieron identificar (revisión manual) |
    | `DTE/<Empresa>-Compras` / `-Ventas` | Todos los documentos son de una empresa configurada; si hay de varias, gana la que tiene más archivos |
    | `DTE/Otros` | El correo no contiene ningún DTE |

    Aunque el correo lleve una sola etiqueta, cada archivo se guarda en la carpeta y la hoja de su propia empresa.

### Salidas

- **Hoja `<Empresa> - Registros DTE`**, con una fila por archivo y estas columnas: Tipo de DTE, Remitente, Asunto, Fecha del correo, Mes, Año, Nombre del archivo, Formato, URL en Drive, Carpeta, Fecha de emisión, Ver correo, Fecha de proceso, URL externa, Código de generación, Tipo de documento.
- **Hoja `Log de Errores DTE`**: ZIP dañados, JSON corruptos o que no son DTE, errores de Drive o de la hoja, y documentos a los que les falta su pareja.
- **Correo de alerta** al administrador, solo si ocurre un fallo crítico.
- **Registro de ejecución** (en el editor de Apps Script), para seguir el avance mientras corre:
  ```
  >>> [PREPARACIÓN] Etiquetas de Gmail listas (9) - 2.1 s
  >>> [PREPARACIÓN] Carpetas y hojas listas (4 hojas de registro) - 8.4 s
  >>> [BANDEJA] Página 1: 100 hilos por revisar (puede haber más en la siguiente página) | Tiempo disponible: 228 s
  [1/100] "Newsletter" -> DTE/Otros | 0.8 s
  [2/100] "Factura CCF marzo" -> DTE/ATS Group-Compras | Guardados: 2 | 3.1 s
        Revisando enlace externo: portal.proveedor.com
  >>> [PROGRESO] Revisados: 100 | Con DTE: 12 | Archivos guardados: 30 | Tiempo disponible: 40 s
  >>> [FIN] Hilos revisados: 112 | ... | Duración: 238.5 s | Quedan hilos pendientes para la próxima ejecución.
  ```
  `[TIEMPO]` indica en qué correo se detuvo por límite de tiempo; la siguiente ejecución lo retoma.

### Protecciones

- Un bloqueo impide que corran dos ejecuciones a la vez.
- **Límite de tiempo:** Google detiene cualquier ejecución a los 6 minutos. El script deja de empezar trabajo nuevo a los 4 minutos, revisa el tiempo también dentro de cada correo y no abre enlaces si queda menos de 1 minuto. Lo que falte continúa en la siguiente ejecución, y el anti-duplicados evita repetir lo que ya se guardó.
- Si un mismo correo agota el tiempo 2 veces, se etiqueta `DTE/No procesado` y se anota en el log como "Hilo demasiado largo", para que no bloquee las siguientes ejecuciones.
- Un correo con error se anota en el log y no detiene el resto.
- Si falla la escritura en la hoja, reintenta. Si aun así falla, quita de Drive el archivo que no quedó registrado.

---

## Instalación

1. Crear un proyecto en [script.google.com](https://script.google.com) con la cuenta del cliente y pegar el contenido del script.
2. Editar `CONFIG` al inicio del script:
   - `CLIENT_NAME`: nombre del cliente, que aparece en el asunto de las alertas.
   - `COMPANIES`: una entrada por empresa con sus NIT (y el DUI si es persona natural). No usar `/` en el nombre.
     ```javascript
     COMPANIES: [
       { name: 'Otra Empresa', nits: ['06140000000000', '02073991-2'] }
     ],
     ```
   - `RECIPIENT_EMAIL`: correo que recibe las alertas.
   - `DTE_ROOT_FOLDER_ID`: ID de la carpeta raíz en Drive. Si se deja vacío, el script crea "Registros DTE".
 

3. Ejecutar `processDTEAttachments` una vez a mano con **Ejecutar** (no con **Depurar**: el depurador es mucho más lento y llega al límite de 6 minutos) y aceptar los permisos.
4. Crear un activador (Activadores → + → `processDTEAttachments` → según tiempo):
   - Cuenta Gmail gratuita: **cada 30–60 minutos**. Google da 90 minutos diarios de ejecución por activadores.
   - Google Workspace: cada 10–15 minutos, con 6 horas diarias.

---

