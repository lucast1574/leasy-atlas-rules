---
name: leasy-atlas-rules
description: Aplicar las reglas de Lucas y los criterios de review y validación de CI exclusivamente en leasyauto/atlas, al implementar cambios, corregir fallos de CI, revisar PRs o atender reviews. No aplicar a otros repositorios aunque compartan stack u organización.
---

# Reglas de Leasy Atlas

Confirma el ámbito por la solicitud, URL del PR o identidad del remoto: `leasyauto/atlas`. El nombre Atlas por sí solo no identifica este proyecto.

Lee [las reglas de Lucas](references/atlas-rules.md) antes de trabajar en Leasy Atlas. Es la copia persistente de las instrucciones proporcionadas por Lucas el 12 de octubre de 2026. Aplica solo las secciones pertinentes a la tarea.

Al implementar, corregir código o CI, preparar una entrega o atender técnicamente una review, lee también [el flujo de implementación, reviews y CI](references/review-and-ci.md). Recoge el estudio de los comentarios de Mikel y Yoel y las lecciones incorporadas hasta el 15 de octubre de 2026. Aplica su autorrevisión final antes de publicar o declarar el cambio listo para revisión. Para listar PRs o resumir comentarios, basta con leer las fuentes solicitadas; ese trabajo no requiere checkout ni tests.

Si el cambio toca locks de leads/procesos, sincronización o aprobación documental, políticas por país, readiness o diálogos anidados, lee además [los criterios derivados de la review de #618](references/guarantor-review.md). Incluyen las ocho observaciones de Yoel del 15 de octubre: distinguir hallazgos de hipótesis no reproducidas y comprobar los consumidores actuales antes de aplicar una propuesta.

Las instrucciones actuales de Lucas prevalecen. Las propuestas históricas de reviewers son evidencia contextual: comprobar si aplican al código actual. No convertir un snippet condicional, una convención de ramas o una recomendación antigua en autorización ni en una regla que sustituya las instrucciones explícitas de Lucas.

El ámbito exclusivo indicado por Lucas prevalece sobre cualquier ejemplo o mención de otros repositorios. No trasladar estas reglas a otros proyectos ni convertirlas en preferencias globales.

Para mutaciones en GitHub (push de código, crear/editar PR, comentarios, issues), exige autorización textual explícita de Lucas para la acción. Una instrucción directa como «responde a Mikel en GitHub» ya autoriza esa respuesta: no pedir otra confirmación. La excepción autorizada por Lucas el 14 de octubre de 2026 es subir automáticamente las capturas de cambios visibles de Atlas a `lucast1574/pr-assets`; sigue [capturas y assets del PR](references/pr-screenshots.md). Esta autorización no publica el código ni crea o edita el PR. Consultar PRs, comentarios y código es lectura; proceder cuando lo solicite. No interpretar los ejemplos de comandos como autorización para ejecutarlos.

Cuando cambie la UI visible de Atlas, aunque el cambio de código esté solo en backend, captura el resultado real, súbelo a `lucast1574/pr-assets` y prepara el cuerpo del PR con la imagen embebida mediante el enlace devuelto por GitHub. Hazlo automáticamente dentro de la entrega, sin volver a pedir permiso para la captura o su subida. Lee [capturas y assets del PR](references/pr-screenshots.md) para las condiciones y el flujo. Publica ese cuerpo únicamente cuando Lucas autorice crear o actualizar el PR.

Al pedir revisar un comentario, leer primero el comentario; no ejecutar tests antes de identificar el trabajo necesario. Al pedir un ticket, preparar primero el ticket, análisis y código pertinentes; verificar después.

## Criterios de trabajo

- Antes de implementar, explicar el problema observado y localizar la regla existente, sus llamadores y el recorrido afectado. Detectar decisiones de producto sin inventarlas ni bloquear el trabajo independiente.
- Corregir la causa y sus consumidores reales. Una capacidad nueva debe funcionar en las superficies que la exponen; una regla de escritura debe protegerse en el servicio, aunque se omita la UI.
- Mantener un único dueño de cada decisión. Evitar parámetros contradictorios, APIs solo usadas por tests, fallbacks que ocultan fallos y refactors ajenos al problema.
- Elegir pruebas que fallen ante la regresión concreta y que recorran la frontera afectada. Probar componentes interactivos dentro del formulario o modal real cuando el riesgo esté en su composición. Confirmar recolección y ejecución en los jobs y países pertinentes.
- Derivar comandos, versiones, servicios y selección de tests de la configuración actual del repositorio. La guía histórica de comandos no sustituye a los workflows actuales de CI.
- Validar el estado final del diff después de la última corrección. Informar qué pasó, qué falló y qué no pudo ejecutarse; no presentar SQLite, tests omitidos o checks de otro commit como equivalentes al CI del head actual.
- Al responder una review, vincular cada observación con el cambio real y su evidencia. Reescribir la descripción según la implementación final antes de preparar su publicación autorizada.

Un CI verde o un conteo alto de tests no sustituyen comprobar el escenario que cambia. No declarar el trabajo listo con un defecto relevante conocido o un check aplicable fallido sin atender. Corregir lo autorizado y explicitar las limitaciones; un CI todavía pendiente debe reportarse como pendiente. No prometer perfección ni ausencia de futuras reviews.

Las rutas Windows y la guía Docker son contexto de la máquina descrita por Lucas; comprobar el entorno actual antes de usarlas. No asumir que existen ni ejecutar comandos de reset como parte de una tarea rutinaria.
