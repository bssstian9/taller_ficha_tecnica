# Contrato de API — Vehículos (MS2)

**Proyecto:** Sistema de Gestión de Talleres Mecánicos — Grupo 10
**Consumido por:** `frontend/src/infrastructure/api/VehicleService.ts`
**Fuente de reglas de negocio:** `Requerimientos.md` (RF-06 a RF-08) y `sistematizacion_final.docx` §4.1, §8

Este documento describe los endpoints que el frontend ya está llamando (vía
`VehicleService`) para el registro y consulta de vehículos. El objetivo es que
el backend (API Gateway + MS2) implemente exactamente esta forma de request/
response, para no tener que tocar el código del frontend cuando el servicio
quede desplegado.

## Convenciones generales

- **Todo pasa por la API Gateway**, nunca directo a MS2:
  `http://localhost:8000/api/vehiculos...` (puerto y prefijo ya configurados
  en `apiClient.ts` vía `VITE_API_URL`).
- El Gateway valida el JWT y enruta por prefijo de ruta; MS2 **vuelve a
  validar** el token y aplica la autorización por rol (defensa en
  profundidad, §8).
- Header obligatorio en todos los endpoints: `Authorization: Bearer <token>`.
- Los errores siempre se devuelven como:
  ```json
  { "detail": "mensaje legible para el usuario" }
  ```
  (es el formato que el frontend ya sabe interpretar en `errors.ts`).
- `patent` (patente) es **única** a nivel de base de datos (§4.1). El backend
  es la fuente de verdad; el frontend solo hace un chequeo optimista antes de
  enviar el formulario.

### Forma del objeto `Vehicle`

Usada como response en todos los endpoints de lectura/creación:

```json
{
  "id": "string (uuid)",
  "patent": "string",
  "brand": "string",
  "model": "string",
  "year": "number",
  "mileage": "number",
  "clientId": "string (uuid del cliente dueño)"
}
```

---

## 1. Registrar vehículo

```
POST /api/vehiculos
```

| | |
|---|---|
| Rol requerido | `cliente` |
| Request body | `{ "patent": "ABCD-12", "brand": "Ford", "model": "Fiesta", "year": 2018, "mileage": 85120 }` |
| Response `201` | `Vehicle` (ver forma arriba) |
| Errores | `409` patente ya existe · `422` campo faltante o formato de patente inválido |

**Notas:**
- `clientId` **no** viene en el body — el backend lo obtiene del JWT (`sub`),
  nunca confiar en un valor enviado por el cliente.
- Formato de patente esperado por el frontend: `4 letras + guion + 2 números`
  (ej. `ABCD-12`). Si el backend valida otro formato, avisar para ajustar la
  regex del formulario.

---

## 2. Mis vehículos (portal Cliente)

```
GET /api/vehiculos/mios
```

| | |
|---|---|
| Rol requerido | `cliente` |
| Response `200` | `Vehicle[]` — solo los vehículos del cliente autenticado |

---

## 3. Catálogo completo (portal Administrador)

```
GET /api/vehiculos
```

| | |
|---|---|
| Rol requerido | `administrador` |
| Response `200` | `Vehicle[]` — todos los vehículos registrados |

**Mejora futura (no bloqueante):** soportar `?search=` para filtrar
server-side por patente/marca/modelo. Hoy el frontend trae todo y filtra en
el cliente.

---

## 4. Vehículos asignados (portal Mecánico)

```
GET /api/vehiculos/asignados
```

| | |
|---|---|
| Rol requerido | `mecanico` |
| Response `200` | `Vehicle[]` |

⚠️ **Importante:** según el MER (`04-mer-erd`), "asignado" no es un atributo
propio de `Vehiculo` — se deriva de qué `OrdenTrabajo` tiene
`mecanico_actual_id` igual al mecánico logueado, y desde ahí se llega al
`vehiculo_id`. Es probable que este endpoint termine viviendo en el módulo de
**Gestión de Órdenes** en vez de en el de Vehículos (un `JOIN` interno en
MS2). El contrato hacia el frontend puede mantenerse igual aunque cambie de
dónde se resuelve internamente.

---

## 5. Ficha técnica por id (los 3 portales)

```
GET /api/vehiculos/{id}
```

| | |
|---|---|
| Rol requerido | cualquiera autenticado |
| Response `200` | `Vehicle` |
| Errores | `404` si el vehículo no existe |

⚠️ **Sobre control de acceso:** este endpoint **no** debe bloquear por
propiedad (ej. impedir que el mecánico o el administrador vean la ficha de un
auto que no es "suyo") — ambos roles necesitan verla legítimamente. El
control de que un **cliente** solo pueda ver sus propios vehículos ya lo hace
el frontend comparando `vehicle.clientId` contra el usuario logueado.

Si el equipo prefiere reforzar esto también en el backend (defensa en
profundidad), debe devolver `403` — no `404` — para no confundir "no existe"
con "no te pertenece".

---

## Manejo de errores esperado por el frontend

`errors.ts` interpreta las respuestas así — útil para que el backend sepa qué
status devolver en cada caso:

| Status | Cuándo usarlo |
|---|---|
| `404` | El recurso no existe |
| `409` | Conflicto (ej. patente duplicada) |
| `422` | Validación de datos (formato, campo faltante) |
| `5xx` | Error del servidor |
| Sin response (timeout/red caída) | El frontend lo trata como "backend no disponible" y muestra un aviso de reintento sin bloquear la UI |
