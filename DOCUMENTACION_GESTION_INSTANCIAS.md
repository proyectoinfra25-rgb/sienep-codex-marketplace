# Gestión de Instancias

## Alcance implementado y fusionado

El módulo se administra desde `/instancias`, separado de la ficha de Estudiantes. La ficha del estudiante conserva la creación y el historial que ya existían.

### Administración general

- Listado paginado de instancias activas, con código, título, estudiante, fecha, categoría, estado y detalle.
- Filtros por código, título, estudiante, rango de fechas, categoría y estado.
- Acciones de detalle, creación, edición, clonación y baja lógica según permisos.
- La pantalla reutiliza `lib/instancias.ts`, `types/instancia.ts` y los formularios existentes; no duplica el flujo de alta desde la ficha del estudiante.
- Backend: búsqueda global paginada en `/api/instancias/buscar`, DTO de búsqueda sin comentario confidencial y filtros opcionales compatibles con PostgreSQL.

### Seguridad y confidencialidad

- El backend mantiene autorización por permiso en cada operación.
- La búsqueda general devuelve un DTO reducido, sin comentarios confidenciales.
- El frontend solo muestra las acciones habilitadas por los permisos disponibles en sesión.
- El detalle debe seguir siendo la fuente de autorización del backend; la ocultación visual no sustituye el control de acceso.

### Pruebas verificadas antes de RF13

- Frontend: `npm test` aprobó 109 pruebas; TypeScript, ESLint y build no reportaron errores en la versión fusionada de Gestión de Instancias.
- Backend: la regresión completa previa aprobó 167 pruebas.
- La corrección de búsqueda tipa los filtros nulos de JPQL para PostgreSQL y evita `SQLState 42P18` en Railway.

## RF13: Google Calendar

### Integración existente de Recordatorios

Recordatorios ya tiene OAuth, tokens cifrados, creación, actualización, eliminación, reintentos y persistencia de `google_event_id`. Esta es la única integración que debe reutilizarse.

### Estado actual de Instancias

La versión fusionada del detalle de Instancia todavía llama a `GET /api/calendario/instancias/{id}/evento-preparado`. Ese endpoint solo prepara datos; no crea un evento en Google.

Hay cambios **locales, no verificados ni desplegables todavía**, iniciados para completar RF13:

- `db/migrations/20261029_google_calendar_instancias.sql` agrega `id_instancia` a `google_calendar_eventos` y `google_calendar_oauth_states`.
- La migración conserva los vínculos actuales a Recordatorios, hace opcional `id_recordatorio`, agrega claves foráneas a Instancias y checks para exigir exactamente un destino por registro.
- Agrega un índice único parcial `(id_usuario, id_instancia)` para evitar eventos duplicados de una misma instancia para el mismo usuario.
- `GoogleCalendarEvento`, su repositorio, `CalendarioGoogleService` y `CalendarioGoogleController` contienen un inicio local de soporte para creación desde Instancia.
- `lib/instancias.ts` contiene una función local de sincronización, pero la pantalla aún no fue conectada definitivamente y el flujo OAuth, actualización, baja y pruebas siguen pendientes.

Por lo tanto, **RF13 no está completo**. No se debe subir, desplegar ni ejecutar esta migración hasta terminar, compilar y probar el flujo completo.

### Pendiente para completar RF13

1. Generalizar el estado OAuth para que apunte a una Instancia o un Recordatorio y redirija al frontend correcto.
2. Conectar el detalle de Instancia al endpoint real; si no hay conexión, iniciar OAuth, y al retorno sincronizar.
3. Crear el evento real y conservar su identificador para evitar duplicados en solicitudes posteriores.
4. Actualizar el evento guardado al editar fecha, hora o título de una Instancia.
5. Eliminar el evento, o conservarlo marcado como pendiente para reintento, cuando se da de baja una Instancia.
6. Evitar que comentarios confidenciales o datos sensibles se envíen a Google. El evento debe contener únicamente información pública mínima.
7. Completar pruebas con mocks: permisos, OAuth pendiente y conectado, creación, duplicado, actualización, baja, error 401/Google y regresión de Recordatorios e Instancias.
8. Ejecutar compilación, pruebas completas, lint y build antes de subir los cambios o aplicar la migración.

### Criterio de cierre de RF13

RF13 se considera terminado únicamente cuando una Instancia autorizada puede iniciar OAuth si es necesario, crear una sola vez su evento de Google Calendar, actualizarlo al modificarse, eliminarlo o reintentarlo al darse de baja, y todas las pruebas automatizadas pasan sin regresiones de Recordatorios. La migración debe estar aplicada primero en un entorno de prueba, nunca directamente en Railway sin respaldo y validación posterior.

## Aplicación segura futura de la migración en Railway

No ejecutar aún. Cuando RF13 esté terminado y las pruebas pasen:

1. Respaldar la base PostgreSQL desde Railway.
2. Verificar que los eventos y estados OAuth existentes tengan `id_recordatorio` no nulo.
3. Aplicar una sola vez `db/migrations/20261029_google_calendar_instancias.sql` en una transacción.
4. Confirmar las restricciones `ck_google_calendar_eventos_objetivo` y `ck_google_calendar_oauth_states_objetivo`.
5. Desplegar el backend compatible y verificar el arranque con `spring.jpa.hibernate.ddl-auto=validate`.
6. Probar primero una cuenta OAuth no productiva y una instancia de prueba autorizada.

### Reversión

Si la aplicación aún no creó eventos asociados a Instancias, la reversión consiste en eliminar los checks, el índice parcial, las claves foráneas y las columnas `id_instancia` agregadas por la migración. Nunca ejecutar la reversión si existen registros con `id_instancia` sin migrarlos o eliminarlos de forma controlada.
