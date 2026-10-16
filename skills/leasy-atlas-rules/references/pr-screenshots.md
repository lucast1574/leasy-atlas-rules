# Capturas de UI y assets de PR de Atlas

## Alcance y autorización

Lucas pidió expresamente el 14 de octubre de 2026 que los cambios visibles de UI tengan captura, que esta se suba automáticamente a su repositorio personal `lucast1574/pr-assets` y que el enlace de GitHub se adjunte al cuerpo del PR. Esta autorización persiste para ese flujo en `leasyauto/atlas`; no volver a pedirla para capturar o subir esas imágenes. Incluye cambios de backend que alteran contenido visible, como el formato de una columna.

La subida de la imagen no autoriza publicar código ni crear/editar el PR. Si Lucas todavía no autorizó el PR, completar la captura, su subida y el cuerpo local con el enlace; publicar el cuerpo al recibir esa autorización. Si la creación o actualización del PR ya está autorizada, adjuntar la imagen como parte de esa operación.

Destino verificado: [lucast1574/pr-assets](https://github.com/lucast1574/pr-assets), rama predeterminada `main`, repositorio público. Usar datos ficticios de desarrollo para las capturas; no subir datos personales reales, credenciales ni contenido confidencial a ese destino público. Si la captura contiene esos datos, reproducir el escenario con fixtures antes de subirla.

## Flujo de entrega

1. Capturar el escenario corregido en el navegador o superficie real de Atlas después del último cambio que invalide la evidencia. Verificar visualmente el resultado. No presentar un mockup como prueba de la implementación ni una captura antigua como evidencia del estado final. Capturar los estados adicionales necesarios para demostrar el cambio; no exigir capturas ajenas al ticket.
2. Guardar el archivo local y subirlo al repositorio personal, bajo `atlas/<TICKET>/`, siguiendo la estructura existente. Usar nombres descriptivos; añadir un hash del contenido o una versión cuando sea necesario para preservar capturas anteriores. Si el mismo archivo ya está subido y es idéntico, reutilizar su enlace. No sobrescribir ni borrar assets de otros PRs.
3. Para PNG/JPEG/WebP, usar un mecanismo que conserve los bytes: Git o la API de Contents con `content` en base64. Las herramientas limitadas a archivos UTF-8 no sirven para subir binarios. No cambiar de cuenta, crear otro repositorio o ampliar permisos como alternativa automática.
4. Conservar el `download_url` o enlace equivalente devuelto por GitHub que renderice la imagen. Verificar que apunta al archivo subido y que puede leerse. No inventar un enlace de `user-attachments`: esos enlaces solo se usan si GitHub los devuelve mediante una subida real.
5. Incluir en el cuerpo del PR Markdown de imagen, `![Descripción breve del resultado](URL_VERIFICADA)`, con una nota corta del escenario y de que son datos de prueba cuando corresponda. Una ruta local o un enlace de texto aislado no reemplaza la imagen embebida. Al crear o editar el PR de forma autorizada, verificar que el cuerpo remoto contiene el enlace correcto.

Si falla la captura o la subida, conservar el trabajo y explicar el paso pendiente. No declarar adjuntada la imagen sin verificarlo. Ante un resultado de subida desconocido, leer primero el path previsto antes de reintentar para evitar duplicados. Si falta acceso al repositorio, pedir el dato o acceso concreto que falta, sin sustituir el destino elegido por Lucas.
