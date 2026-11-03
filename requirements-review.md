# Revisión de requisitos · v0.1 → v0.3

Análisis de `requirements.md` (v0.1) contra los wireframes (`wireframes.excalidraw`) y las decisiones tomadas entre el 22 y el 24 de octubre de 2026.

- **v0.2:** se aplicó en `requirements.md` todo lo que ya estaba decidido en los wireframes.
- **v0.3 (24 oct):** el equipo resolvió las 10 preguntas abiertas y las 12 propuestas. La decisión de cada una está en las secciones 4 y 5, ya aplicada en `requirements.md` y en los wireframes. No queda nada pendiente de decidir.

Los IDs de páginas (PUB-, AUT-, ORG-, ADM-, SYS-, COA-, V2-) son los nombres de los frames del wireframe.

---

## 1. Resumen

| | Cantidad |
|---|---|
| Incoherencias o ambigüedades en v0.1 | 16 |
| Requisitos que faltaban y ya estaban decididos en los wireframes | 24 |
| Propuestas (resueltas el 24 oct: 7 al MVP, 5 a Próximamente) | 12 |
| Preguntas abiertas (resueltas el 24 oct) | 10 |

Cambios de fondo en v0.2:

1. **La cuenta es opcional para todos** y obligatoria solo para crear eventos. "Organizador" deja de ser un rol global: es un rol **por evento**.
2. **Los participantes sin cuenta** acceden a su inscripción con un enlace personal que llega por email.
3. **Alcance del MVP explícito**: el plan de cuotas pasa a v2. Equipos, co-administradores, notificaciones configurables, idioma, cancelaciones y reembolsos pasan a "Próximamente" (`[Next]`).
4. **Ledger corregido**: no hay saldos de participantes; las cuentas son por evento. La reversión sirve solo para corregir errores, y los reembolsos usan transacciones propias.
5. **Módulos nuevos**: descubrimiento público, acceso de participantes, administración de la plataforma, notificaciones y legal/privacidad. También hay NFR nuevos de seguridad y de datos/tiempo/dinero.

---

## 2. Incoherencias y ambigüedades encontradas en v0.1

| # | Dónde | Problema | Resolución en v0.2 |
|---|---|---|---|
| I-01 | §1.1 / §1.2 | El nombre aparece como "EventsWorld" y como "Event's World". | Se unifica como **Event's World**. |
| I-02 | §2.1 | "Organizer" se define como rol global, pero cualquiera puede crear una cuenta y "organizador" es simplemente quien crea un evento. | Roles globales: Visitante, Usuario, Super Admin. Roles por evento: Organizador (dueño) y Co-administrador `[Next]`. |
| I-03 | §2.1 | "Guest" se usa para dos roles distintos: el co-administrador ("Co-Administrator (Guest)") y el participante ("Participant / Guest"). | El co-administrador es un colaborador invitado. "Participante" es quien se inscribe, con o sin cuenta. |
| I-04 | §2.1 | Las capacidades del Super Admin ("event management and user auditing") son vagas. | Se detallan según ADM-01 a ADM-04: ver, suspender, auditar y cambiar roles. No aprueba pagos ni puede entrar como otro usuario. |
| I-05 | FR-01.1 | Pide "Full Name", mientras la lista original y los wireframes usan Nombres + Apellidos. No define edad mínima. | Nombres + Apellidos. Mayores de 18 años, validado en el backend (decisión del equipo en AUT-01). |
| I-06 | FR-01.1 | "Platform accounts are only required for Organizers" se lee como si solo los organizadores pudieran tener cuenta. | La cuenta es opcional para todos y obligatoria solo para crear eventos (PUB-08, AUT-01). |
| I-07 | FR-02.1 | Faltan campos que el wireframe exige: portada, categoría, lugar, fecha de inicio y de fin, edad mínima. Además, "maxIncriptionDate" tiene un error de tipeo. | Se completa la lista con sus validaciones (fin > inicio, fecha límite ≤ inicio). |
| I-08 | FR-02.2 | "The Organizer MUST upload their payment QR" contradice los eventos gratuitos que admite el wireframe. | El QR es obligatorio solo en eventos pagados. Se agregan el monto del QR, su vencimiento y advertencias no bloqueantes. |
| I-09 | FR-02.3 | "ALL remote event forms" es ambiguo: no queda claro si se refiere a eventos virtuales o a la inscripción en línea. | Se aclara que el comprobante es obligatorio en la **inscripción en línea de eventos pagados**. |
| I-10 | FR-02.2 / FR-03.2 / FR-03.3 / FR-03.4 / FR-03.6 | Varios requisitos del MVP dependen del plan de cuotas: filtros completo/parcial, pagos parciales y "dinero estimado con planes". | El plan de cuotas pasa a `[v2]`. Cada requisito se reescribe para el pago único y deja la parte de cuotas marcada como v2. |
| I-11 | FR-03.5 | Las acciones "Accept / Put in Observation / Mark as Error" no coinciden con el wireframe (Aprobar / Observar / Rechazar). "Mark as Error" no dice qué le pasa al participante. | Aprobar / Observar / Rechazar. El motivo es obligatorio al observar o rechazar, y cada acción envía un email al participante. |
| I-12 | FR-03.7 | "The administrator has to be able to edit this data": no queda claro si es el Super Admin o el organizador. | Lo hace el **organizador del evento**, con motivo obligatorio, historial, auditoría y aviso al participante. |
| I-13 | NFR-01.3 | Los comprobantes de pago se sirven por "CDN". Un comprobante contiene datos personales y bancarios, así que no debe ser público. | Dos buckets: uno público (portadas, por CDN) y otro privado (comprobantes, solo con URL firmadas de pocos minutos). |
| I-14 | NFR-03.2 | Pide cuentas para "Users, Events and the Platform". Pero los participantes ya no tienen saldo, y la plataforma no mueve dinero (no hay comisiones definidas). | Cuentas por evento: Caja del organizador, Ingresos y Reembolsos por pagar. Cada movimiento lleva el `registration_id`. La cuenta de plataforma depende de Q-03. |
| I-15 | NFR-03.2 | Dice que *Accounts* es una entidad inmutable. Lo inmutable son las transacciones y los movimientos (entries), no las cuentas. | Se corrige la redacción. |
| I-16 | NFR-03.4 | Usa la reversión para "refunds, cancellations, or corrections". Un reembolso no es un error: el dinero sí entró y luego salió. Registrarlo como reversión hace que la caja deje de coincidir con el extracto del banco. | Reversión = solo corregir errores (`[MVP]`, FR-03.8). Reembolsos y cancelaciones usan sus propias transacciones (`[Next]`, FR-03.9, ORG-14). |

---

## 3. Requisitos que faltaban (ya decididos en los wireframes)

| # | Requisito nuevo en v0.2 | Origen |
|---|---|---|
| F-01 | Acceso del participante sin cuenta con un enlace personal (token) por email | FR-05.1 · PUB-05 |
| F-02 | Recuperar el enlace por email, con respuesta neutra y límite de envíos | FR-05.2 · PUB-06 |
| F-03 | "Mis inscripciones" para quien tiene cuenta, vinculadas por email verificado; las pasadas quedan como historial | FR-05.3 · PUB-08 |
| F-04 | Confirmar el email escribiéndolo dos veces. Solo el organizador puede cambiarlo después | FR-03.3, FR-05.4 · PUB-03/04 |
| F-05 | Código de inscripción (EW-XXXXX) | FR-03.3 · PUB-04 |
| F-06 | Estados del evento y sus transiciones (borrador, publicado, cerrado, finalizado) | FR-02.4 · ORG-11 |
| F-07 | Restricción de edición con inscritos: precio, moneda y borrado de campos, más los "datos críticos" de la nota de ORG-02 | FR-02.5 |
| F-08 | Monto que cobra el QR, con advertencias si no coincide con el precio o si el monto es libre | FR-02.2 · ORG-03 |
| F-09 | Vencimiento del QR, con aviso si vence antes del cierre de inscripciones | FR-02.2 · ORG-03 |
| F-10 | Enlace público del evento, copiable y con QR para afiches | FR-02.6 · ORG-05 |
| F-11 | Lista "Mis eventos" con filtros por estado y alertas | FR-02.7 · ORG-01 |
| F-12 | Explorar eventos y detalle público (con metadatos Open Graph) | FR-04.1, FR-04.2 · PUB-01/02 |
| F-13 | Menú público ☰ con y sin sesión | FR-04.3 · PUB-09 |
| F-14 | Revertir una aprobación hecha por error | FR-03.8 · ORG-07 |
| F-15 | Registrar un pago presencial de alguien que fue rechazado | FR-03.10 · ORG-07 |
| F-16 | Inscripción presencial: queda Aprobada al instante y calcula el vuelto en efectivo | FR-03.4 · ORG-09 |
| F-17 | Panel, eventos, usuarios y auditoría del Super Admin | FR-06.x · ADM-01..04 |
| F-18 | Emails transaccionales (inscripción, aprobado, observado, rechazado, enlace, datos editados) | FR-07.1 · SYS-02 |
| F-19 | Términos y privacidad, con registro de la versión aceptada por cada persona | FR-08.1, FR-08.2 · PUB-07 |
| F-20 | Inicio de sesión: mensaje genérico, bloqueo tras 5 intentos y destino según el rol | FR-01.6 · AUT-03 |
| F-21 | Perfil: tema claro/oscuro/sistema (el idioma es `[Next]`) | FR-01.7 · ORG-12 |
| F-22 | Protección de formularios públicos sin CAPTCHA: límites por IP y email, y validación de archivos | NFR-04.2, NFR-04.3 |
| F-23 | Tokens de enlace personal seguros: aleatorios, guardados como hash y revocables | NFR-04.1 |
| F-24 | Diseño completo del ledger: tipos de transacción del MVP y de [Next] | NFR-03.x · ORG-14 |

---

## 4. Propuestas · resueltas el 24 oct

Descartadas antes de esta revisión: duplicar evento, QR de cobro predeterminado, "no tiene email" y n.º de operación.

| # | Propuesta | Vista | Decisión | Dónde quedó |
|---|---|---|---|---|
| P-01 | Verificar el email del organizador antes de publicar | AUT-02 | **MVP** | FR-01.4 |
| P-02 | Recuperar / restablecer contraseña | AUT-04 | **MVP** | FR-01.5 |
| P-03 | Login con Google | AUT-01/03 | **MVP** | FR-01.8 |
| P-04 | Evento oculto de Explorar (solo con enlace) | ORG-11 | Próximamente | FR-02.6 |
| P-05 | Exportar participantes a CSV | ORG-06 | **MVP** | FR-03.12 |
| P-06 | Check-in con QR de entrada | ORG-13, PUB-05 | Próximamente | FR-03.11, FR-05.5 |
| P-07 | Contacto del organizador en el detalle del evento | PUB-02, ORG-02 | **MVP**: campo obligatorio (WhatsApp o email) en el paso 1 | FR-02.1, FR-04.2 |
| P-08 | Agregar al calendario (.ics) | PUB-04 | Próximamente | FR-03.3 |
| P-09 | Estado "Suspendido" para eventos, aplicado por el Super Admin | ADM-02 | **MVP** | FR-02.4, FR-06.2 |
| P-10 | Alertas del Super Admin (comprobantes > 48 h, QR vencido, tasa de rechazos) | ADM-01 | **MVP**. "Reportado por usuarios": próximamente | FR-06.1 |
| P-11 | Solicitud de borrado de datos (anonimizar sin tocar el ledger) | PUB-07, PUB-08 | Próximamente | FR-08.3 |
| P-12 | NFR de calidad: accesibilidad, logs y monitoreo, backups | — | Próximamente | NFR-06 |

---

## 5. Preguntas · resueltas el 24 oct

| # | Pregunta | Decisión | Dónde quedó |
|---|---|---|---|
| Q-01 | ¿Qué inscripciones ocupan cupo? | Pendiente, En observación y Aprobado. Rechazado libera el cupo. El cupo se valida en el servidor con bloqueo. | FR-03.13 |
| Q-02 | ¿Qué datos no se editan con inscritos? | Se bloquean fechas, país, ciudad, lugar, zona horaria, precio, moneda y monto del QR. El cupo solo baja hasta el número de inscritos y los campos del formulario se ocultan, no se borran. Título, descripción, portada, categoría, edad mínima y contacto siguen editables. | FR-02.5 · nota en ORG-02 |
| Q-03 | ¿Event's World cobrará comisión? | No en el MVP, así que no hay cuenta de plataforma. El esquema queda listo para agregar comisiones por inscripción y/o suscripciones o planes. | NFR-03.2 · roadmap "Future" |
| Q-04 | ¿Eventos virtuales o híbridos? | Solo presenciales en el MVP. Virtual e híbrido: próximamente. | FR-02.1 · ORG-02 |
| Q-05 | ¿Varias inscripciones con el mismo email en un evento? | Sí; el formulario avisa, pero lo permite. | FR-03.14 · nota en PUB-03 |
| Q-06 | ¿Qué pasa si el QR vence con inscripciones abiertas? | Las inscripciones pagadas se pausan hasta subir un QR nuevo. Se avisa al organizador por email y en Mis eventos. | FR-02.2, FR-07.1 · ORG-03, ORG-01, PUB-02 |
| Q-07 | ¿Hasta cuándo se puede re-subir un comprobante? | Hasta la fecha límite de inscripción. Después pasa a Rechazada definitiva y libera el cupo. | FR-03.5 · PUB-05 |
| Q-08 | ¿Cuánto tiempo se guardan comprobantes y datos de pago? | 5 años, provisional hasta tener el texto legal. | FR-08.4 |
| Q-09 | ¿Cómo se maneja la zona horaria? | Se guarda en UTC más la zona del evento y siempre se muestra en la hora del evento. Nuevo campo "Zona horaria" en el paso 1. | NFR-05.1, FR-02.1 · ORG-02 |
| Q-10 | ¿Qué tipos de campo soporta el formulario? | 8 tipos: texto corto, texto largo, número, selección única, selección múltiple, fecha, teléfono y casilla. Archivo extra: próximamente. | FR-02.3 · nota en ORG-04 |

---

## 6. Impacto en la arquitectura (para la siguiente fase)

- **Autorización en dos niveles:** un rol global en la cuenta y una tabla `event_members(event_id, user_id, role)` para los roles por evento. Las políticas del backend deben validar la pertenencia al evento en cada endpoint `/panel/eventos/:id/...`.
- **Ledger de doble entrada por evento:** se diseña desde ya con todos los tipos de transacción, para no migrar cuando llegue [Next].
- **Acceso por token:** tabla `registration_access_tokens` con el hash, el vencimiento y la revocación del token. Al abrir el enlace, el token se canjea por una sesión corta.
- **Almacenamiento:** R2 con bucket público y bucket privado. Las URL firmadas se generan en el backend después de autorizar.
- **Auditoría:** registro append-only, separado del ledger, de toda acción sensible.
