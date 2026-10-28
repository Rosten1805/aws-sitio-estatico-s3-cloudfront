# Sitio Web Estático con Amazon S3 + CloudFront

Despliegue de un sitio web estático utilizando **Amazon S3** como almacenamiento de los ficheros del sitio y **Amazon CloudFront** como CDN para distribuirlo de forma rápida y segura (HTTPS).

## 📐 Arquitectura

![Arquitectura: usuario → CloudFront (Origin Access Control) → S3 Bucket (Bucket Policy), con acceso directo al bucket bloqueado](docs/arquitectura.png)

El diseño original planteaba bloquear el acceso directo al bucket de S3, permitiendo el acceso solo a través de **CloudFront** mediante **Origin Access Control (OAC)** y una **Bucket Policy** restrictiva.

> ⚠️ **Cambio de diseño:** finalmente se optó por habilitar el **Alojamiento de sitios web estáticos (Static Website Hosting)** de S3, lo que requiere que el bucket sea **público** (CloudFront usará el endpoint de website hosting como *origen personalizado*, no OAC, ya que ese endpoint no soporta peticiones firmadas con SigV4). El diagrama de arriba se actualizará para reflejar este cambio.

## 🎯 Objetivo del proyecto

- Alojar un sitio web estático (HTML/CSS/JS) en un bucket de S3.
- Servirlo a través de CloudFront para mejorar el rendimiento y añadir HTTPS — **el endpoint de S3 Website Hosting por sí solo no soporta SSL/TLS**, por lo que alojar un sitio ahí sin CloudFront por delante no es buena práctica (el tráfico va sin cifrar).
- Restringir el acceso directo al bucket (solo accesible a través de CloudFront).
- Documentar el proceso completo paso a paso con capturas de pantalla.

## 🌐 Sitio web

El sitio desplegado es la **"Homer's Web Page"**, una página estática (HTML/CSS/JS) ubicada en [`homers-web-page-master/`](homers-web-page-master/):

```
homers-web-page-master/
├── index.html
├── estilo.css
├── fuentes/   (fuente Cooper Black)
├── gifs/      (animaciones)
├── img/       (imágenes del telón e icono)
├── js/        (code.js)
└── sonidos/   (audio de fondo)
```

> ⚠️ Nota de licencias: las fuentes Cooper Black incluidas (`COOPBL.TTF` de Microsoft y `CooperBlackStd.otf` de Adobe) son fuentes comerciales con copyright de terceros. Se mantienen en el repositorio por tratarse de un proyecto personal/educativo, sin uso comercial.

## 🛠️ Servicios de AWS utilizados

- **Amazon S3** — almacenamiento del contenido estático.
- **Amazon CloudFront** — distribución del contenido (CDN) y HTTPS.
- **AWS Certificate Manager (ACM)** — certificado SSL/TLS (si se usa dominio propio).
- **Amazon Route 53** — gestión del dominio (si aplica).
- **IAM** — políticas de acceso al bucket.

## ✅ Requisitos previos

- Cuenta de AWS.
- AWS CLI instalado y configurado (opcional, para despliegues por línea de comandos).
- Un dominio propio (opcional, si se quiere usar un nombre de dominio personalizado en vez de la URL de CloudFront).

## 📋 Pasos realizados

> Esta sección se irá completando con cada paso del proyecto, junto con sus capturas de pantalla correspondientes.

## 🥇 Parte 1: Alojar la web directamente en el bucket S3 (enfoque ingenuo)

Primero se aloja el sitio usando únicamente **S3 Static Website Hosting**, sin CloudFront por delante. Esta parte sirve para comprobar en primera persona **por qué no es buena práctica** dejarlo así en producción (sin HTTPS y con coste por tráfico directo al origen).

### 1. Creación del bucket S3

Se crea un bucket de **Uso general** (tipo recomendado para este caso de uso), con espacio de nombres global.

> Nota: en el formulario de creación se mostraba por defecto la región Europa (Estocolmo) `eu-north-1`, pero el bucket se creó finalmente en **Europa (Irlanda) `eu-west-1`**, confirmado en la pestaña Propiedades del bucket.

- **Nombre del bucket:** `demo-web-estatica-proyecto`
- **Propiedad del objeto:** ACL deshabilitadas (recomendado) — todos los objetos pertenecen a esta cuenta y el acceso se gestiona solo mediante políticas (bucket policy), no mediante ACLs.

![Creación de bucket en S3 - nombre demo-web-estatica-proyecto, región eu-north-1, ACL deshabilitadas](screenshots/01-crear-bucket.png)

### 2. Subida de los ficheros del sitio

Se sube el contenido de [`homers-web-page-master/`](homers-web-page-master/) directamente a la raíz del bucket, conservando la estructura de subcarpetas (`fuentes/`, `gifs/`, `img/`, `js/`, `sonidos/`) para que las rutas relativas de `index.html` y `estilo.css` sigan funcionando.

- **25 archivos, 3.2 MB** subidos correctamente (`index.html`, `estilo.css` y los recursos en sus subcarpetas).

![Carga de archivos al bucket S3 - 25 archivos, 3.2 MB, estructura de carpetas conservada](screenshots/02-subida-archivos.png)

Resultado final en la raíz del bucket `demo-web-estatica-proyecto`: `estilo.css`, `index.html` y las carpetas `fuentes/`, `gifs/`, `img/`, `js/`, `sonidos/` conservadas correctamente.

![Raíz del bucket S3 con index.html, estilo.css y las subcarpetas fuentes, gifs, img, js, sonidos](screenshots/03-estructura-bucket.png)

### 3. Habilitación de alojamiento de sitios web estáticos

En la pestaña **Propiedades** del bucket → **Alojamiento de sitios web estáticos** → **Editar**:

- **Alojamiento de sitios web estáticos:** Habilitar
- **Tipo de alojamiento:** Alojar un sitio web estático
- **Documento de índice:** `index.html`

![Edición de alojamiento de sitios web estáticos - habilitado, documento de índice index.html](screenshots/04-habilitar-website-hosting.png)

AWS avisa de que, para que el contenido sea accesible desde el punto de enlace del sitio web, el bucket debe permitir lectura pública.

Una vez habilitado, S3 genera el **punto de enlace del sitio web (website endpoint)**:

🔗 **http://demo-web-estatica-proyecto.s3-website-eu-west-1.amazonaws.com**

![Punto de enlace de sitio web del bucket generado por S3](screenshots/05-endpoint-website.png)

Al visitarlo, el navegador avisa de que el sitio **no tiene conexión segura (HTTPS)**:

![Aviso del navegador: el sitio no admite una conexión segura, solo HTTP](screenshots/06-aviso-no-https.png)

Esto ocurre porque los *endpoints* de S3 Website Hosting **solo sirven contenido por HTTP**, nunca HTTPS. Este es precisamente el motivo por el que se añade **CloudFront** por delante del bucket en el siguiente paso: para servir el sitio con HTTPS (mediante un certificado de ACM) y mejorar el rendimiento con CDN.

**Motivo adicional: coste por tráfico directo a S3.** Cada visita/recarga de la web genera peticiones `GET` contra el bucket, y S3 cobra tanto por **solicitudes** (por cada 1.000 peticiones) como por **transferencia de datos saliente** (por GB servido a internet) — no solo por almacenamiento:

![Tabla de precios de almacenamiento S3 Standard](screenshots/13-precios-s3.png)

Con tráfico recurrente o archivos pesados (vídeo, imágenes grandes), servir el contenido directamente desde S3 sin caché dispara ese coste en cada recarga. **CloudFront** cachea el contenido en ubicaciones *edge*, por lo que las visitas repetidas se sirven desde la caché sin llegar al bucket origen, reduciendo tanto las solicitudes como la transferencia de datos facturadas a S3.

### 4. Configuración de permisos del bucket

Al visitar el endpoint del sitio web aparece un error **403 Forbidden / AccessDenied**, porque el bucket sigue siendo privado por defecto.

![Error 403 Forbidden - AccessDenied al visitar el endpoint del sitio web](screenshots/07-error-403.png)

**4.1. Desactivar "Bloquear acceso público"** — pestaña **Permisos** → *Bloquear acceso público (configuración del bucket)* → Editar:

| Antes (todo bloqueado) | Después (todo desactivado) |
|---|---|
| ![Bloqueo de acceso público activado](screenshots/08-bloqueo-acceso-publico-activado.png) | ![Bloqueo de acceso público desactivado](screenshots/09-bloqueo-acceso-publico-desactivado.png) |

AWS pide confirmar la acción escribiendo `confirmar`, ya que puede hacer públicos los objetos del bucket:

![Confirmación para desactivar el bloqueo de acceso público](screenshots/10-confirmar-bloqueo-publico.png)

**4.2. Añadir una Bucket Policy de lectura pública** — tras desactivar el bloqueo, seguía apareciendo el error 403: faltaba conceder explícitamente el permiso de lectura mediante una política de bucket (`Permisos` → `Política de bucket` → Editar):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForStaticWebsite",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::demo-web-estatica-proyecto/*"
    }
  ]
}
```

![Política de bucket guardada correctamente con el Resource corregido a demo-web-estatica-proyecto](screenshots/11-politica-bucket.png)

AWS confirma *"Se editó correctamente la política de buckets"*, con el **Bloqueo de acceso público: Desactivado** y el `Resource` ya apuntando al bucket correcto (`demo-web-estatica-proyecto`). Con esto, el bucket permite lectura pública de los objetos (`s3:GetObject`) a cualquiera, necesaria para que el endpoint de Static Website Hosting sirva el contenido correctamente.

**Resultado:** la web ya carga correctamente, sin error 403, sin scroll vertical y con el reproductor de audio funcionando:

![Homer's Web Page cargando correctamente desde S3, sin error 403](screenshots/12-web-funcionando.png)

**Conclusión de la Parte 1:** la web funciona, pero con dos problemas de fondo:
- ❌ **Sin HTTPS** — el endpoint de S3 no soporta SSL/TLS.
- ❌ **Coste por tráfico directo** — cada recarga genera solicitudes y transferencia de datos facturadas directamente contra el bucket, sin ningún tipo de caché.

## 🥈 Parte 2: Cómo alojar una web estática correctamente (con CloudFront)

Para corregir las limitaciones de la Parte 1, se coloca **Amazon CloudFront** delante del bucket: añade HTTPS mediante un certificado de **ACM** y cachea el contenido en ubicaciones *edge*, reduciendo las peticiones y la transferencia de datos que llegan a S3.

### 5. Creación de la distribución de CloudFront

_Pendiente de documentar._

### 6. Configuración de HTTPS / certificado SSL

_Pendiente de documentar._

### 7. Configuración de dominio personalizado (opcional)

_Pendiente de documentar._

### 8. Pruebas finales

_Pendiente de documentar._

## 📸 Capturas de pantalla

> Las capturas se añadirán en la carpeta [`screenshots/`](screenshots/) y se irán enlazando aquí a medida que avancemos.

## 💰 Costes estimados

_Pendiente de documentar (estimación de costes de S3 + CloudFront según tráfico esperado)._

## 🧹 Limpieza de recursos

_Pendiente de documentar (pasos para eliminar los recursos de AWS y evitar costes innecesarios)._

## 📚 Recursos y enlaces útiles

_Pendiente de añadir enlaces a documentación oficial de AWS consultada durante el proyecto._
