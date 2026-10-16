# Atlas: implementación, reviews y CI

Ámbito: solo `leasyauto/atlas`. Complementa `atlas-rules.md`; no sustituye las instrucciones de Lucas. Para capturas aplica la autorización permanente y acotada descrita en [capturas y assets del PR](pr-screenshots.md); las demás mutaciones en GitHub conservan sus requisitos de autorización.

Origen: estudio del 13 de octubre de 2026 de 56 PRs de `lucast1574` (45 mergeados, 7 abiertos, 4 cerrados sin merge). En 33 había 218 intervenciones con texto de `mikelmulti460` y `yoelayan`, incluidos resúmenes y seguimientos; no son 218 defectos independientes. Ampliado con las reviews del 15 de octubre de #613 y [#618](guarantor-review.md), consultadas en GitHub. Las referencias son ejemplos históricos, no afirmaciones sobre defectos presentes en el head actual. No volver a descargar todo el historial para cada tarea.

## 1. Investigar el problema y delimitar la solución

Leer primero el ticket, comentario o log solicitado. En una review, leer también las respuestas posteriores y el diff del commit al que se refiere: distinguir observación vigente, corrección previa y nueva objeción. `isResolved` o `isOutdated` por sí solos no prueban que la causa esté corregida.

Conservar el grado de evidencia: «trazado por lectura», «hipótesis por verificar» y «reproducido» no son equivalentes. Si el reviewer declara que no verificó una condición, comprobar primero si el catálogo, fase o contrato actual la permite. Su propuesta orienta la investigación; no establece por sí sola una nueva regla de negocio ni demuestra que el snippet resuelve todos los consumidores.

Antes de editar, identificar:

- Comportamiento esperado, comportamiento actual y condición concreta que dispara el fallo.
- Configuración de producto que explica el comportamiento actual. No quitar una restricción existente para hacer desaparecer un supuesto bug.
- Dueño de la regla y entradas que la ejecutan: CRM, portal, servicios directos, tareas o integraciones, según el caso.
- Resultado visible y persistido que demostrará la corrección.

Para un cambio de comportamiento, conservar en las notas de trabajo una relación breve: escenario y superficie → resultado esperado → prueba o comprobación. Incluir el camino negativo pertinente (permiso/scope, estado, versión o conflicto) y las ramas/países que realmente cambian. No convertirla en una plantilla extensa para el usuario ni añadir casos sin relación con el diff.

Reproducir el fallo cuando sea posible. Si falta entorno, registrar la limitación y avanzar con análisis verificable; no afirmar reproducción ni escribir a reviewers sin autorización. Una decisión pendiente de Producto debe quedar explícita, con su consecuencia y recomendación, mientras continúa el trabajo que no dependa de ella. No pedir confirmación para decisiones rutinarias ya autorizadas.

Ejemplos: [#532](https://github.com/leasyauto/atlas/pull/532#issuecomment-5879338306), donde el arreglo ignoraba la anticipación configurada; [#478](https://github.com/leasyauto/atlas/pull/478#discussion_r4019431465), donde agregar un tipo de equipo exigía revisar el modelo de capacidades.

## 2. Seguir los consumidores y preservar los contratos

Usar búsquedas de llamadores y contratos para seguir la cadena pertinente: UI → payload real → vista/schema → servicio → persistencia/outbox → presenter/historial. No ampliar el PR a consumidores sin relación demostrable.

- Si un componente compartido envía campos nuevos, comprobar todos los endpoints que lo reciben. No basta con actualizar CRM si también se usa en agencias.
- Enforcar permisos, bloqueos y reglas en la operación pública que escribe. Probar el rechazo al omitir la UI cuando esa sea la superficie de riesgo. La presentación debe explicar los bloqueos y ofrecer acciones coherentes con los permisos.
- Compartir la decisión, no solo una constante. Dashboard y notificaciones pueden tener filtros de destinatario distintos, pero deben derivar la misma ventana temporal de una sola consulta.
- Un helper nuevo debe tener llamadores reales y reemplazar la regla anterior. No dejar una copia probada en tests junto a otra usada en producción.
- Mantener campos obligatorios si el backend siempre los emite. Evitar defaults y coerciones que conviertan un payload incompleto en estado aparentemente válido.
- Preservar la diferencia entre `None`, cero, vacío, no aplicable y error técnico. Validar el dato declarado antes de derivar o sugerir otro valor.
- Las capacidades independientes deben seguir siendo independientes: recoger viajes no implica autorizar su revisión tras aprobar el agregado; un `false` explícito debe respetarse.
- Medir el costo de consultas por la vista real cuando el cambio afecta carga o serialización. Incluir documentos existentes y paginación si son relevantes; un expediente vacío puede ocultar consultas repetidas. No prometer ahorro por un parámetro que producción no usa.
- Cuando CRM y portal comparten datos de dominio, conservar sus políticas y presenters propios. No introducir dependencia del servicio de dominio sobre el detalle CRM ni builders/flags solo para adaptar superficies. Reutilizar la serialización existente y cargar en lote los registros aplicables cuando evita trabajo repetido; comprobarlo con registros cargados, no solo slots vacíos.
- En vistas genéricas con registry, conectar las proyecciones específicas mediante el punto de extensión de la acción, sin importar cada dominio en la vista. Un resolutor de lectura puede reutilizar las instancias ya cargadas y autorizadas de la página, agrupadas por acción; no releerlas por ID ni consultar dominios que ninguna fila necesita. Esto no cambia la regla de que los servicios de escritura reciben IDs y cargan su estado fresco.
- Separar la presentación de la elegibilidad de una operación correctiva. Un metadato cosmético inconsistente no debe bloquear incidentalmente una reversión ni derribar toda la bandeja. Si el contrato permite omitir ese dato, dejarlo ausente y registrar el error con identificadores de diagnóstico; manejar solo los casos conocidos, sin ocultar fallos de dominio. Un bloqueo financiero debe ser explícito en el servicio, con código estable y mensaje traducido. Probar solicitud/aprobación y ejecución directa, conservando sus permisos y validaciones.
- Aplicar la misma proyección histórica en lista y detalle. Preservar el snapshot y la evidencia persistidos; un campo legible ya guardado se deriva de ese snapshot, no del nombre actual. Probar que renombrar al cliente no cambia lo solicitado y que los tests no fijan como contrato una divergencia accidental entre superficies.
- En formularios que precargan datos existentes, distinguir selección visible de edición explícita. Revisar qué significan campo omitido, vacío y `null` en el servicio: registrar asistencia sin tocar el vehículo no debe borrar el modelo ni cambiar su versión, aunque el modelo precargado ya no esté disponible. Cubrir también el cambio explícito y el reinicio del estado de edición al cambiar de rama.
- Cuando un fallo de interacción pertenece a un componente compartido, buscar primero sus capacidades y consumidores. Evitar reproducir su manejo de foco/Escape con selectores DOM y listeners globales en cada pantalla. La corrección compartida debe respetar la restricción de Lucas sobre `src/components/`; preparar su alcance y obtener autorización explícita si la tarea vigente no la cubre. Ver [diálogos anidados](guarantor-review.md#diálogos-anidados).

Ejemplos: [#401, contrato CRM/portal](https://github.com/leasyauto/atlas/pull/401#discussion_r3856739226); [#494, capacidades independientes](https://github.com/leasyauto/atlas/pull/494#discussion_r4065646607); [#584, parámetros sin consumidores](https://github.com/leasyauto/atlas/pull/584#pullrequestreview-5507043831).

## 3. Elegir pruebas que detecten la regresión

Para cada comportamiento modificado, decidir qué fallo concreto detectaría su prueba. Reutilizar suites y fixtures pertinentes; no añadir infraestructura, suites duplicadas ni tests para detalles sin efecto observable.

- Un test de servicio no demuestra el contrato del POST ni la interacción del modal. Cubrir la frontera en la que estuvo el fallo, con campos que la UI realmente envía.
- No mockear la decisión bajo prueba. Para fases contractuales, usar filas reales si se quiere comprobar la consulta de fases; parametrizar una variable ignorada no agrega casos.
- Comprobar status HTTP exacto y efectos persistidos o ausencia de efectos, según el caso. Evitar aserciones con `or`, excepciones genéricas o ausencia de un mensaje como sustituto de éxito.
- Para cambios de flujo, ejecutar la transición relevante. Crear directamente el estado final puede ocultar un camino imposible, como viajes pendientes que ya no pueden aprobarse.
- Para orden determinista, hacer que el orden esperado difiera del de inserción y cubrir la carga paginada si usa otro camino.
- Para interacción, usar render de cliente y eventos pertinentes: la secuencia hover/foco/pointerdown/clic, Escape, clic fuera, timers y cleanup cuando corresponda. SSR y typecheck no prueban handlers. Si el componente vive dentro de un formulario, dropdown o modal, probar también esa composición real; un callback simulado del componente aislado no demuestra que el formulario se conserve ni que el evento no cierre al padre.
- Para menús o diálogos anidados modificados, comprobar Escape desde el control y una opción, cierre exclusivo del elemento activo, retorno del foco y conservación de los datos pendientes. Comprobar también que cerrar normalmente el padre sigue funcionando. Esperar el cambio observable de foco/DOM con la herramienta del runner, sin sleeps arbitrarios ni eliminar la aserción por ser asíncrona.
- Para formularios condicionales y peticiones de disponibilidad modificados, comprobar el payload tras cambiar de rama o limpiar la selección. Una respuesta tardía no debe restaurar opciones o datos descartados. Simular el transporte cuando haga falta, conservando los componentes y la lógica cuya interacción se quiere probar.
- Fijar reloj, timezone e idioma cuando se afirme una fecha literal. Fijar el reloj antes de crear fixtures con validaciones temporales: un token válido hoy no debe invalidar el test al avanzar el calendario. No copiar el mismo cálculo de implementación como único oráculo ni depender de tiempos máximos frágiles del runner.
- En seguridad, probar el contenido y el efecto: una respuesta con PII no debe filtrarse dentro de un campo llamado `note`; validar solo nombres de claves es insuficiente.
- En dependencias, probar el uso real en Atlas. No probar la librería contra sí misma ni duplicar en un assert la versión mínima que ya impone el resolver.
- Cuando sea práctico y relevante, demostrar que la prueba falla sin la corrección, en un entorno aislado que conserve el trabajo actual. Una prueba de funcionalidad nueva no debe seguir verde en la base sin ese cambio. No exigir esta comparación a tests de caracterización o cobertura previa ajenos al fix.
- Para presupuestos exactos de consultas, declarar los permisos y el estado que corresponden al escenario. Comprobar los tests aislados y seleccionados tras recoger la suite completa: los imports pueden cambiar el orden de un registry y activar consultas de permisos ajenas al comportamiento medido. Si hay influencia del estado dejado por otros tests, reproducir también la secuencia o suite del job. No inflar el presupuesto, sustituir igualdad por un máximo ni mockear autorización para ocultar la diferencia. Un permiso explícito de bandeja es apropiado en un test de presentación; no debe añadirse a un caso que prueba rechazo o alcance de acceso. Confirmar que el test corregido sigue detectando la consulta redundante en la base.

Ejemplos: [#401, falso positivo HTTP](https://github.com/leasyauto/atlas/pull/401#discussion_r3856740251); [#446, helper sin uso](https://github.com/leasyauto/atlas/pull/446#discussion_r4039212409); [#524, compatibilidad real](https://github.com/leasyauto/atlas/pull/524#discussion_r4166797298).

Caso comprobado: [#603](https://github.com/leasyauto/atlas/pull/603) tenía CI aprobado y ocho tests del flujo de citas aprobados, pero Escape desde el desplegable cerraba el modal y perdía el formulario pendiente. El test añadido sobre `LeadAppointmentModal` falló antes del arreglo. [La corrección](https://github.com/leasyauto/atlas/commit/7bf372ba34118ead49ffc269a9b28cf7564cbde3) cubre botón/opción, foco, comentario conservado y segundo Escape. Es una razón para probar composición cuando cambia la interacción, no para ejecutar todos los tests de UI en cada cambio de backend.

## 4. Preparar y verificar el CI del cambio actual

### Descubrir el contrato de ejecución

Antes de elegir la validación, inspeccionar los archivos vigentes que la definen: `AGENTS.md`, `.github/workflows/`, `pyproject.toml`, lockfiles, configuración de pytest/conftest, `package.json`, configuración de frontend y Compose/Docker cuando se usen. Identificar versiones instaladas, directorio de ejecución, comandos, variables, servicios, matriz de países y selección por markers/paths/labels. Las instrucciones explícitas de Lucas siguen prevaleciendo.

No inferir un script por su nombre habitual ni actualizar dependencias para arreglar una diferencia local de herramientas. Usar el runtime y el entorno previstos por el repo. Instalar o arrancar servicios necesarios dentro del alcance autorizado; no resetear datos ni cambiar CI para acomodar el entorno local.

### Elegir checks según el diff

Conservar los checks de entrega exigidos por `atlas-rules.md` y usar la configuración actual para ejecutarlos correctamente. Aplicar además las comprobaciones pertinentes:

| Cambio | Evidencia necesaria |
| --- | --- |
| Python | Lint y formato como CI; tests del comportamiento tocado y consumidores afectados. `ruff check` no sustituye a `ruff format --check`. |
| Svelte/TS | Typecheck, lint y pruebas de interacción afectadas; story real si se modifica un componente conforme a las reglas de Lucas. Build cuando lo exige el job aplicable o cambia el empaquetado. |
| Workflow/catálogo/i18n | Ramas afectadas del flujo, registro completo de claves y catálogos de países pertinentes; compilación según CI. No aceptar el fallback como evidencia de traducción completa. |
| ORM/esquema/concurrencia | Consistencia y plan de migraciones; PostgreSQL real para constraints, locks o índices que SQLite no reproduce. Validar migraciones en BD de prueba si corresponde. |
| Dependencias/lockfiles | Comprobación del lock y compatibilidad del uso real; resolver con la herramienta del repo, sin arreglar el lock a mano. Build de producción si lo requiere el flujo de seguridad. |

No ejecutar indiscriminadamente toda la matriz en cada edición. Primero validar la corrección identificada; ampliar a consumidores, países y jobs afectados. Si cambia una regla común, justificar el alcance por sus consumidores y por la selección real de CI, no solo por el archivo editado.

### Comprobar que los tests corren

- Reproducir el módulo de settings, backend de BD y alcance del job que protege el cambio. Una tanda focalizada en PostgreSQL no demuestra que el mismo test pase en la suite base de SQLite ni bajo la colección completa de CI; cada ejecución aporta evidencia distinta.
- Verificar recolección y selección de casos nuevos o movidos con los mismos flags/markers de CI. Un nombre de archivo con `postgres` no añade `@pytest.mark.postgres`.
- Comprobar resultados y skips. Cero tests seleccionados o una suite completamente omitida no es validación exitosa del comportamiento.
- Distinguir países efectivamente ejecutados de parametrizaciones que no cambian el país activo. Revisar servicios, timezone e idioma de cada caso pertinente.
- No usar SQLite con `--no-migrations` como sustituto de validación PostgreSQL o migraciones. Puede aportar evidencia parcial, identificada como tal.
- Si un caso multipaís o de PostgreSQL debe proteger el cambio en CI, comprobar que la selección del job incluye ese caso concreto. Ejecutarlo manualmente en otro país no hace que entre al job diferencial. Ajustar markers solo cuando corresponda a su contrato; no marcar todos los tests como multipaís ni ampliar la matriz por rutina.

### Resolver fallos sin ocultarlos

Leer el log del primer fallo relevante y su causa antes de editar. Distinguir error del cambio, fallo previo demostrado e incidencia de infraestructura con evidencia; no atribuirlo al entorno por intuición. No relajar asserts, introducir skips, ocultar excepciones ni desactivar jobs para conseguir verde. Cambiar selección o expectativas solo si el contrato lo requiere y explicar por qué conserva cobertura.

Si faltan servicios o acceso, completar las validaciones disponibles y declarar exactamente qué falta. No pedir push para descubrir errores que pueden comprobarse localmente. Un reintento de infraestructura necesita una razón concreta; no repetir runs sin cambios ni diagnóstico. Reejecutar workflows en GitHub requiere la autorización de mutación pertinente.

### Validar el estado final

Después de la última edición, repetir los checks invalidados por ella y revisar el diff completo, incluidos archivos nuevos. Si se incorpora la base, repetir los checks afectados por esa integración; no hacer reset, rebase o force-push automáticamente. Respetar la rama/base indicadas por Lucas y la autorización necesaria para actualizar GitHub.

Asociar la evidencia al estado probado: commit o diff local, comando, runtime, país/BD, resultado y log cuando aporte diagnóstico. Usar logs separados por ejecución. Para comparar base y corrección, esperar que terminen los procesos que leen ese checkout antes de cambiar sus fuentes; preferir un checkout aislado. No atribuir a una ejecución la salida de otra ni reutilizar un resultado anterior tras cambiar su escenario.

Antes de una publicación ya autorizada, dejar evidencia de comandos, resultados, países/BD y limitaciones. Después de publicar, consultar checks del SHA exacto del head: un run verde de un commit anterior no valida el actual. Usar los jobs requeridos y los condicionales pertinentes; no asumir que el agregado verde prueba un job omitido. Si siguen pendientes, informar pendiente. Si el CI falla, continuar el diagnóstico dentro de lo autorizado; otro push necesita estar cubierto por la autorización de Lucas.

Reportar por separado: checks locales aprobados, CI remoto del SHA actual, y verificaciones no ejecutadas o bloqueadas. No declarar «CI verde» basándose solo en pruebas locales. No prometer ausencia absoluta de fallos futuros.

Ejemplos: [#446, tests fuera del job PostgreSQL](https://github.com/leasyauto/atlas/pull/446#discussion_r4017789252); [#544, fecha dependiente del país](https://github.com/leasyauto/atlas/pull/544#discussion_r4264129497).

Caso comprobado: en [#613, review de Yoel](https://github.com/leasyauto/atlas/pull/613#pullrequestreview-5533104054), el resumen se corrigió con un hook del registry, sin releer solicitudes, y una proyección compartida en lista/detalle que no bloquea la reversión por metadatos del contrato. Sin embargo, [el primer CI falló](https://github.com/leasyauto/atlas/actions/runs/38742866389): dos tests de consultas pasaban aislados, pero la colección completa importaba cinco acciones de leads antes que coins y añadía diez comprobaciones de permisos. [La corrección](https://github.com/leasyauto/atlas/commit/72c212cf4453b1a45372ed5a08f5194f57cecf69) declaró `approvals.view` en las fixtures de presentación. Los conteos siguieron siendo exactos y los dos tests siguieron fallando contra el código anterior por una consulta extra. Se verificaron aislados, con colección completa y con el job PE completo; [el CI del nuevo SHA pasó](https://github.com/leasyauto/atlas/actions/runs/38744771538). Los permisos y cifras de este caso no son presupuestos generales para otras vistas.

## 5. Revisar datos existentes y consecuencias de negocio

Aplicar según la superficie del cambio; no convertir cada tarea en una auditoría completa:

- JSON de creación nuevo no modifica checklists ya persistidos. Explicar si el cambio aplica solo a nuevos registros o necesita una corrección explícita para los existentes.
- Una corrección de datos necesita candidatos inequívocos, dry-run e idempotencia. Reportar casos ambiguos sin inventar motivos, reclasificar personas ni alterar evidencia firmada.
- Un cambio de política debe preservar el recorrido de registros históricos acordado. Probar la entrada real de readiness/conversión; una excepción en una etapa posterior no ayuda si una validación anterior impide llegar.
- Conservar snapshots de la evidencia efectiva de cada decisión. Cambiar el valor actual no debe reescribir lo que respaldó una resolución anterior.
- En dinero, comprobar moneda, conversión histórica, precisión y consistencia entre consumidores. Un saldo parcial no puede presentarse como total confiable. La concurrencia debe proteger la entidad que comparte el invariante, como el requerimiento.
- En multipaís, distinguir titular de aval y no ampliar políticas de otros países al extender un catálogo compartido.
- Para locks, sincronización documental, readiness y backfills de políticas por país, aplicar los [criterios de #618](guarantor-review.md). Revisar orden de locks entre entradas concurrentes, ausencia de escrituras e historial en un no-op, selección documental vigente y coherencia entre actualización normal y backfill; leer ese detalle solo cuando el cambio afecta esas superficies.

Ejemplos: [#512, registros v2 existentes](https://github.com/leasyauto/atlas/pull/512#discussion_r4085607448); [#563, backfill ambiguo](https://github.com/leasyauto/atlas/pull/563#discussion_r4169544224).

## 6. Cerrar la review con evidencia

Mantener para el trabajo actual un seguimiento breve: observación → causa → cambio → prueba/evidencia → pendiente. No copiarlo como plantilla extensa en todas las respuestas.

Volver a recorrer el escenario original después de corregir. Buscar si persiste la misma causa en otro consumidor y si el fix introduce otra fuente de verdad. No declarar resuelto porque existe un commit, una respuesta del autor o un test con el nombre esperado.

Evaluar las condiciones de cada sugerencia. En [#544](https://github.com/leasyauto/atlas/pull/544#discussion_r4264129529), una operación individual propuesta solo si hacía falta quedó sin consumidores y duplicó el envío. Elegir la solución mínima conectada al flujo real; explicar el criterio si difiere del snippet.

Preparar la descripción con el comportamiento final: alcance, consumidores, datos existentes, diferencias por país, cambios de contrato compartido y pruebas realmente ejecutadas. Agrupar por comportamiento, retirar beneficios no medidos, snippets viejos y conteos de tests desactualizados. Cuando cambie la UI visible, incluir automáticamente capturas verificadas, subidas a `lucast1574/pr-assets` y embebidas con el enlace devuelto por GitHub, siguiendo [capturas y assets del PR](pr-screenshots.md).

Revisar tanto los comentarios inline como las reviews generales y la conversación del PR. Al responder comentarios inline, no marcarlos como resueltos: Lucas pidió conservar su estado de resolución (13 de octubre de 2026). Una respuesta o un cambio publicado no autoriza resolver el hilo.

Al buscar comentarios pendientes para Lucas, usar su criterio: la última intervención humana del hilo no es suya, aunque GitHub muestre `isResolved=true` o el hilo esté outdated. Comprobar si la review general ya tuvo respuesta en la conversación; no duplicar respuestas antiguas ni responder avisos automáticos sin una acción pendiente. No incluir PRs históricos cuando pidió únicamente abiertos. Antes de publicar, releer el último comentario y verificar el head vigente para evitar responder a una versión superada o enviar dos veces la misma respuesta. Si pidió dejar una decisión de Producto pendiente, conservarla explícitamente pendiente y no cubrirla con una respuesta genérica de «todo corregido».

Responder punto por punto con referencias de código verificadas y el nombre de la prueba que protege cada corrección. Separar decisión de Producto pendiente, riesgo previo fuera de alcance y defecto introducido, sin usar esa clasificación para ocultar un camino roto que la nueva funcionalidad expone. Publicar, responder o resolver hilos únicamente con autorización textual de Lucas.

## 7. Autorrevisión antes de entregar

Después de implementar y validar, hacer una pasada de revisión sobre el diff final contra la base vigente, separada de la lectura hecha mientras se escribía. Recorrer también los consumidores afectados que no aparecen en el diff. Usar la rúbrica de Mikel/Yoel como criterio técnico, sin inventar convenciones ni cambios de Producto.

Comprobar según lo que cambió:

| Pregunta de revisión | Evidencia que permite cerrar el punto |
| --- | --- |
| ¿La regla tiene un dueño y consumidores reales? | Llamadores localizados; sin parámetro ignorado, override solo de tests, doble cálculo ni API nueva sin uso. |
| ¿La operación protege la regla al omitir la UI? | Frontera real de escritura con permiso/scope/estado pertinentes; rechazo sin efectos parciales cuando corresponda. |
| ¿El test detecta la causa y el efecto? | Escenario observable, datos reales para la decisión probada y aserción precisa; no solo nombre del test, fixture final o mock que decide el resultado. |
| ¿La interacción funciona dentro de su contenedor? | Recorrido del formulario/modal real y conservación de datos/foco cuando el cambio afecta eventos o composición. |
| ¿Una precarga o sincronización altera datos sin una edición real? | Payload omitido cuando corresponde y persistencia/versiones intactas; repetir la sincronización no añade escrituras ni historial del tópico sin cambios. |
| ¿Los caminos concurrentes y las políticas documentales son coherentes? | Si aplica: mismo orden de locks entre entradas, prueba PostgreSQL real, una fuente por política y selección de la versión/contexto documental vigentes; criterios de #618 verificados. |
| ¿Se cubren la rama, país y datos existentes afectados? | Entrada/transición real, alcance de nuevos registros frente a históricos, claves completas y selección efectiva del test aplicable. |
| ¿La descripción coincide con lo entregado? | Campos, textos, permisos, superficies, límites y resultados actuales; sin prometer envío, bloqueo, ahorro de queries, QA o retroactividad no implementados/verificados. |
| ¿Se está entregando la versión validada? | Diff final limpio de archivos temporales, hooks aprobados y resultados ligados al commit actual; CI pendiente/fallido/omitido identificado sin llamarlo verde. |

Corregir los defectos relevantes encontrados dentro del alcance autorizado y repetir solo las verificaciones que el arreglo invalide. No ampliar el PR con refactors o cambios de política para completar la tabla. Si falta una definición o verificación necesaria, explicar el límite concreto; no ocultarlo marcando un checklist ni declarar «todo resuelto».

Antes de quitar draft cuando Lucas lo solicite, comprobar que no queda un defecto relevante conocido del cambio sin atender y que las validaciones locales necesarias pasaron. Tras el push, comprobar el SHA remoto y `isDraft` y consultar los checks de ese SHA. Si CI continúa ejecutándose, informar «listo para review, CI pendiente»; si falla, diagnosticar y corregir dentro de la autorización. Esta guía no autoriza publicar, solicitar reviewers, resolver hilos ni programar seguimiento por sí sola.
