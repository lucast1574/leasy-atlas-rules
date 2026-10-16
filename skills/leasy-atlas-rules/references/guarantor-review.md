# Locks, políticas documentales y diálogos en Atlas

Ámbito exclusivo: `leasyauto/atlas`. Consultar cuando el cambio afecte estas superficies; complementa [reviews y CI](review-and-ci.md). No obliga a ampliar una tarea ajena a ellas.

Fuente: ocho observaciones inline de Yoel en [la review de #618 del 15 de octubre de 2026](https://github.com/leasyauto/atlas/pull/618#pullrequestreview-5537404489) y su seguimiento sobre el modal. Son evidencia de la revisión de ese momento, no una afirmación sobre el head actual. El reviewer no reprodujo el deadlock ni midió las escrituras y dejó explícitamente sin verificar la posibilidad de contextos documentales simultáneos. Mantener esas distinciones hasta comprobarlas.

## Orden de locks entre entradas

[Observación P1](https://github.com/leasyauto/atlas/pull/618#discussion_r4286275317): guardar aval, emitir y firmar tomaban lead → proceso; `update()` podía tomar proceso → lead al inicializar en precontrato.

Antes de añadir un lock, seguir todas las entradas que comparten esas filas, incluidos inicialización, update y llamadas internas. Escribir el orden efectivo de adquisición; comprobar que ninguna rama alcanzable lo invierte. En este flujo, mantener lead → proceso y reutilizar el helper que ya opera con el lead bloqueado, si sigue siendo el contrato actual. No volver a entrar por un método que adquiere los mismos recursos en otro orden ni crear una abstracción nueva sin necesidad.

Evidencia pertinente: PostgreSQL con dos conexiones/transacciones independientes; update en precontrato frente a guardado de aval y frente a emisión, en ambos órdenes de arranque. Coordinar la superposición sin sleeps arbitrarios, propagar errores de ambos participantes y comprobar terminación y estado persistido. Dos llamadas secuenciales o SQLite no prueban ausencia de deadlock. Un test que termina sin solapar los caminos tampoco demuestra el escenario.

## Sincronización sin escrituras redundantes

[Observación P2](https://github.com/leasyauto/atlas/pull/618#discussion_r4286275345): `ensure_document_evaluation_topic` guardaba el tópico aunque su metadata no cambiara, añadiendo historial de `AuditedModel` en cada sincronización.

Comparar los campos de política antes de guardar; conservar metadata ajena a esa política. Compartir la aplicación de la política entre sincronización y backfill cuando ambas hacen la misma operación. Revisar también el parseo repetido del workflow y las consultas, sin añadir cachés que puedan dejar una política obsoleta.

Evidencia pertinente: una segunda sincronización idéntica no añade historial ni UPDATE del tópico; metadata legacy sí se corrige. Medir en el camino real con las herramientas de consultas del repo, diferenciando lecturas de escrituras. Un conteo total menor no demuestra por sí solo ausencia de escrituras. No exigir que subir un archivo nuevo deje intacto el historial propio del archivo: el no-op aquí es la política del tópico sin cambios.

## Configuración por país y un dueño de requisitos

[Configuración duplicada](https://github.com/leasyauto/atlas/pull/618#discussion_r4286275366): evitar mapas paralelos de países en módulos compartidos y listas hardcodeadas en backfills. Consultar la configuración tipada de `countries/<code>/` y derivar de ella los países aplicables. Reutilizar constantes de tipos documentales existentes, como `LEAD_DOCUMENT_TYPE_GUARANTOR_UTILITY_BILL`, en lugar de repetir literales.

Evidencia pertinente: usar `atlas.core.testing.country_data` según su contrato vigente; un país sin esta política debe devolver ausencia de requisitos (MX en el caso revisado). La incorporación de una configuración de país no debe exigir editar un segundo mapa en código compartido. No convertir la ausencia de política en la obligación de subir documentos de otro país.

[Checks duplicados](https://github.com/leasyauto/atlas/pull/618#discussion_r4286275382): localizar quién decide qué documentos exige el aval. `GUARANTOR_DOCUMENTS_UPLOADED` y `GUARANTOR_DOCUMENTS_APPROVED` no deben mantener listas divergentes ni duplicar el mismo pendiente cuando aprobación ya implica carga. Reutilizar el dueño existente o retirar la redundancia si el contrato lo permite; no añadir otra capa solo para unir dos copias.

Evidencia pertinente: SV sin archivos, con archivos sin aprobar y con archivos aprobados. Comprobar los checks concretos y el estado resultante, no solo un booleano final.

## Ramas alcanzables y titulares clientes

[Observación P2](https://github.com/leasyauto/atlas/pull/618#discussion_r4286275386): una guarda `not applicant_is_client` no cambiaba readiness porque un check anterior ya bloqueaba a ese titular; a la vez, retirar documentos de la puerta de Firma y actualizar tópicos podía cambiar su política por otra vía.

Recorrer readiness en orden y demostrar qué ramas son alcanzables. Evaluar conjuntamente titular cliente/no cliente, carga normal, reemplazo/borrado, backfill y puertas de emisión/Firma. Si la sincronización actualiza clientes y el backfill los excluye, comprobar y explicar esa diferencia. No eliminar requisitos para clientes ni extender la excepción por intuición: una decisión de producto pendiente debe seguir explícita mientras avanza el trabajo independiente.

Evidencia pertinente: titular cliente con aval y documentos; comprobar readiness, metadata del tópico tras la subida y requisitos de la puerta afectada. En la descripción, declarar cualquier relajación real de política y el alcance sobre datos existentes.

## Documento y versión que deciden

[Hipótesis P2 por verificar](https://github.com/leasyauto/atlas/pull/618#discussion_r4286275398): un diccionario por `document_type` sin contexto ni orden podía elegir otra fila y discrepar de `linked_document_id`.

Comprobar primero qué duplicados/contextos permite el catálogo actual. Para la selección, reutilizar el contexto oficial, las reglas de reemplazo y el criterio vigente de `sync_documents`; fijar orden determinista con desempate si puede haber varias filas candidatas. Una aprobación antigua no debe aprobar una versión nueva ni una fila de otro contexto.

Evidencia pertinente cuando el escenario sea alcanzable: mismo tipo en contextos distintos, versión reemplazada y documento vigente enlazado; comprobar cuál decide readiness. No fabricar un estado imposible para presentar la sospecha como un defecto reproducido.

## Documentación del mecanismo real

[Observación P2](https://github.com/leasyauto/atlas/pull/618#discussion_r4286275405): se describía una migración cuando el cambio usaba un management command manual; otras guías conservaban gates retirados.

Seguir las referencias de la política en documentación de dominio, servicios y workflows. Distinguir migración automática, comando manual y proyección en lectura. Para un backfill, documentar invocación, países/candidatos, exclusiones, dry-run e idempotencia según lo realmente implementado; no inventar flags. Alinear nombres de checks y puertas con el código/JSON final. Actualizar la descripción del PR si cambió el mecanismo de aplicar la política a datos existentes.

## Diálogos anidados

[Observación P2](https://github.com/leasyauto/atlas/pull/618#discussion_r4286310646): `ContractPartyModal` desactivaba Escape del `Modal` y replicaba foco/listeners con consultas DOM; el problema común estaba en los listeners de Escape de padre e hijo.

Revisar primero `closeOnEscape`, `containFocus` y el manejo de otros diálogos activos del componente compartido. Elegir una corrección acorde al apilamiento real, incluidos portales si los usa: un selector de descendientes sugerido por la review no demuestra por sí solo todos los casos. Evitar restaurar foco en cada clic del panel. Restaurarlo al disparador cuando se cierre el hijo, si el disparador sigue disponible.

La propuesta del reviewer no autoriza editar `src/components/` ni ampliar por sí sola el PR. Respetar la autorización explícita de Lucas y preparar el arreglo compartido o seguimiento concreto cuando quede fuera de alcance.

Evidencia pertinente: Escape cierra solo el diálogo superior y devuelve el foco al disparador; después cierra el padre normalmente. Mantener la prueba de composición de `ContractPartyModal`, y comprobar que un clic dentro del panel, con el hijo cerrado, no mueve el foco. Probar los consumidores afectados por cualquier cambio compartido, sin inventar una suite global para toda tarea de UI.
