# Contrato de comunicación

Todos los servicios hablan HTTP con cuerpos JSON en UTF-8. Las horas van en UTC con el formato `2041-06-14T01:20:33Z`. Cada servicio expone `GET /salud`, que responde `{"servicio": "<nombre>", "estado": "ok"}`.

Los ejemplos completos están en `contrato/ejemplos/`; si este documento y un ejemplo no coinciden, se corrigen los dos en un PR propio.

| Servicio | Puerto | Dueño |
| --- | --- | --- |
| Custodio e informe (Java) | 8080 | Juan Esteban |
| Integrador (Haskell) | 8081 | Isabela |
| Razonador (Prolog) | 8082 | Valentina |
| Estación central (Elixir) | 8083 | Sergio |
| MySQL | 3306 | Valentina (esquema), Juan Esteban (único escritor) |

## 1. Tipos compartidos

### Crudo

Una línea original de un archivo, tal como llegó. `ordinal` es el número de la línea de datos dentro de su archivo (sin contar el encabezado), empezando en 1. En `energia.json` es la posición del registro.

```json
{"id": "raw-tarjetas-0004", "fuente": "tarjetas", "ordinal": 4, "linea": "2041-06-14T01:20:33,IV-07,P4,LAB-B,GEN,OK"}

### Evento

{
  "id": "ev-0007",
  "evidencia_id": "raw-tarjetas-0004",
  "ts": "2041-06-14T01:20:33Z",
  "precision": "exacta",
  "margen_s": 0,
  "fuente": "tarjetas",
  "tipo": "paso_puerta",
  "persona": {"estado": "resuelta", "candidatos": ["IV-07"]},
  "datos": {"tarjeta": "IV-07", "puerta": "P4", "desde": "LAB-B", "hacia": "GEN", "resultado": "OK"}
}

### Problema

{"id": "pb-002", "evidencia_id": "raw-tarjetas-0005", "tipo": "duplicado", "motivo": "Misma tarjeta, puerta, hora y resultado que raw-tarjetas-0004", "usado": false}

### Correccion

{"id": "co-0001", "tipo": "desfase_reloj", "fuente": "radio", "segundos": -420, "motivo": "El reloj de la consola de radio adelanta 7 minutos", "investigador": "ana"}

## Snapshot

{
  "id": "v58-original",
  "version": 58,
  "evidencia": "original",
  "eventos": [],
  "personal": [{"tarjeta": "JS-01", "nombre": "Tomás Okafor", "cargo": "Jefe de estación", "accesos": ["P2", "P3", "P4", "P5", "P7", "P8"]}],
  "estacion": [{"puerta": "P1", "modulo_1": "COM", "modulo_2": "DOR", "lector": false}]
}

{"tipo": "quien_en_modulo", "modulo": "GEN", "desde": "2041-06-14T01:30:00Z", "hasta": "2041-06-14T01:30:00Z"}

{
  "resultado": "confirmado",
  "personas": [{"id": "IV-07", "estado": "confirmado", "evidencia": ["ev-0007"], "explicacion": "Entró a GEN por P4 a las 01:20:33 y ..."}],
  "evidencia": ["ev-0007"],
  "explicacion": "..."
}

## 2. Endpoints

### Integrador (Haskell, 8081)

| Método y ruta | Cuerpo | Respuesta |
| --- | --- | --- |
| `POST /integrar` | `{"directorio": "datos"}` | `{"crudos", "eventos", "problemas"}` |
| `POST /validar-lectura` | `{"fuente", "linea", "evidencia_id"}` | `{"evento", "problema"}` (uno de los dos es `null`) |
| `POST /aplicar-correcciones` | `{"eventos", "correcciones"}` | `{"eventos"}` |

`/validar-lectura` acepta las fuentes `tarjetas`, `presencia` y `temperatura`.

### Razonador (Prolog, 8082)

| Método y ruta | Cuerpo | Respuesta |
| --- | --- | --- |
| `POST /consultar` | `{"pregunta", "snapshot"}` | Respuesta |

Nunca responde 500: una pregunta que no entiende devuelve `desconocido` con el motivo.

### Custodio (Java, 8080)

| Método y ruta | Cuerpo | Respuesta |
| --- | --- | --- |
| `POST /evidencia/lote` | `{"crudos", "eventos", "problemas"}` | `{"version", "insertadas"}` |
| `POST /evidencia/lectura` | `{"clave", "fuente", "linea", "evidencia_id", "evento", "problema"}` | `{"version", "nueva", "afecta"}` |
| `POST /correcciones` | Corrección sin `id` | `{"version", "correccion_id"}` |
| `GET /snapshot?version=n&evidencia=original` | — | Snapshot |
| `GET /estado` | — | `{"version", "evidencias", "correcciones"}` |
| `POST /consultas` | `{"investigador", "version", "evidencia", "pregunta", "respuesta"}` | `{"consulta_id"}` |
| `GET /consultas/{id}` | — | La consulta guardada |
| `GET /consultas/{id}/reconstruir` | — | `{"consulta_id", "reconstruccion"}` (`IDENTICA` o `DIFERENTE`) |
| `GET /cadena/verificar` | — | `{"integridad": "correcta", "filas"}` o `{"integridad": "rota", "tabla", "orden"}` |
| `GET /informe?evidencia=original` | — | HTML |

`afecta` es `{"modulos": [...], "personas": [...], "desde": ts, "hasta": ts}`. Repetir una lectura con la misma `clave` devuelve la misma versión con `"nueva": false`.

### Estación central (Elixir, 8083)

| Método y ruta | Cuerpo | Respuesta |
| --- | --- | --- |
| `POST /lectura` | `{"fuente", "linea"}` | 202 `{"clave", "estado"}` cuando ya está en disco |
| `POST /consultas` | `{"investigador", "pregunta", "evidencia"}` | `{"consulta_id", "version", "evidencia", "respuesta"}` |
| `POST /correcciones` | Corrección sin `id` | Lo que responde el Custodio |
| `POST /suscripciones` | `{"investigador", "pregunta", "evidencia"}` | `{"suscripcion", "respuesta"}` |
| `GET /alertas/stream?investigador=ana` | — | Server-Sent Events, un `data:` por alerta |
| `GET /estado` | — | `{"recibidas", "entregadas", "pendientes", "version", "procesos_consulta"}` |

`evidencia` es `original` (por defecto) o `corregida`. Si un servicio no responde, la Estación contesta 503 con `{"estado": "SERVICIO_NO_DISPONIBLE", "servicio": "<nombre>"}`.

## 3. Convenciones

- Todo en UTC. Salidas deterministas: mismos insumos, mismos bytes; nunca la hora actual dentro de los datos.
- Clave de idempotencia de una lectura en vivo: SHA-256 en hexadecimal de `fuente + "\n" + linea`, calculada por la Estación. El `evidencia_id` en vivo es `raw-<fuente>-` más los 10 primeros caracteres de la clave. En la carga inicial, el Custodio usa la clave `lote:<evidencia_id>`.
- Toda llamada a otro servicio lleva timeout (5 s); un servicio caído produce un error explícito.
- Cada servicio escribe su número de proceso en `trabajo/pids/<servicio>.pid` al arrancar, para que los scripts de robustez puedan detenerlo.

#### Pestaña 2: `contrato/ejemplos/snapshot.json`
```json
{
  "id": "v58-original",
  "version": 58,
  "evidencia": "original",
  "eventos": [
    {
      "id": "ev-0001",
      "evidencia_id": "raw-bitacora-0001",
      "ts": "2041-06-12T12:00:00Z",
      "precision": "aproximada",
      "margen_s": 43200,
      "fuente": "bitacora",
      "tipo": "bitacora",
      "persona": {"estado": "resuelta", "candidatos": ["JS-01"]},
      "datos": {
        "texto": "Día 4. Petrov dice que perdió su tarjeta de acceso. Le haré una nueva cuando pase la tormenta.",
        "menciones": [{"texto": "Petrov", "estado": "resuelta", "candidatos": ["LP-04"]}],
        "tarjeta_perdida": "LP-04"
      }
    },
    {
      "id": "ev-0003",
      "evidencia_id": "raw-tarjetas-0001",
      "ts": "2041-06-14T00:12:04Z",
      "precision": "exacta",
      "margen_s": 0,
      "fuente": "tarjetas",
      "tipo": "paso_puerta",
      "persona": {"estado": "resuelta", "candidatos": ["IV-07"]},
      "datos": {"tarjeta": "IV-07", "puerta": "P2", "desde": "DOR", "hacia": "LAB-A", "resultado": "OK"}
    },
    {
      "id": "ev-0004",
      "evidencia_id": "raw-tarjetas-0002",
      "ts": "2041-06-14T00:30:41Z",
      "precision": "exacta",
      "margen_s": 0,
      "fuente": "tarjetas",
      "tipo": "paso_puerta",
      "persona": {"estado": "resuelta", "candidatos": ["IV-07"]},
      "datos": {"tarjeta": "IV-07", "puerta": "P3", "desde": "LAB-A", "hacia": "LAB-B", "resultado": "OK"}
    },
    {
      "id": "ev-0006",
      "evidencia_id": "raw-tarjetas-0003",
      "ts": "2041-06-14T01:08:15Z",
      "precision": "exacta",
      "margen_s": 0,
      "fuente": "tarjetas",
      "tipo": "paso_puerta",
      "persona": {"estado": "resuelta", "candidatos": ["MR-03"]},
      "datos": {"tarjeta": "MR-03", "puerta": "P5", "desde": "RAD", "hacia": "GEN", "resultado": "OK"}
    },
    {
      "id": "ev-0007",
      "evidencia_id": "raw-tarjetas-0004",
      "ts": "2041-06-14T01:20:33Z",
      "precision": "exacta",
      "margen_s": 0,
      "fuente": "tarjetas",
      "tipo": "paso_puerta",
      "persona": {"estado": "resuelta", "candidatos": ["IV-07"]},
      "datos": {"tarjeta": "IV-07", "puerta": "P4", "desde": "LAB-B", "hacia": "GEN", "resultado": "OK"}
    },
    {
      "id": "ev-0014",
      "evidencia_id": "raw-presencia-0005",
      "ts": "2041-06-14T01:30:00Z",
      "precision": "minuto",
      "margen_s": 60,
      "fuente": "presencia",
      "tipo": "presencia",
      "persona": null,
      "datos": {"modulo": "GEN", "personas": 2, "estado": "OK"}
    },
    {
      "id": "ev-0020",
      "evidencia_id": "raw-tarjetas-0006",
      "ts": "2041-06-14T01:35:02Z",
      "precision": "exacta",
      "margen_s": 0,
      "fuente": "tarjetas",
      "tipo": "paso_puerta",
      "persona": {"estado": "resuelta", "candidatos": ["MR-03"]},
      "datos": {"tarjeta": "MR-03", "puerta": "P5", "desde": "GEN", "hacia": "RAD", "resultado": "OK"}
    },
    {
      "id": "ev-0021",
      "evidencia_id": "raw-tarjetas-0008",
      "ts": "2041-06-14T01:39:47Z",
      "precision": "exacta",
      "margen_s": 0,
      "fuente": "tarjetas",
      "tipo": "paso_puerta",
      "persona": {"estado": "desconocida", "candidatos": []},
      "datos": {"tarjeta": "LP-04", "puerta": "P7", "desde": "GEN", "hacia": "ALM", "resultado": "OK"}
    },
    {
      "id": "ev-0022",
      "evidencia_id": "raw-presencia-0008",
      "ts": "2041-06-14T01:40:00Z",
      "precision": "minuto",
      "margen_s": 60,
      "fuente": "presencia",
      "tipo": "presencia",
      "persona": null,
      "datos": {"modulo": "*", "personas": null, "estado": "SIN_ENERGIA"}
    },
    {
      "id": "ev-0025",
      "evidencia_id": "raw-energia-0002",
      "ts": "2041-06-14T01:40:00Z",
      "precision": "exacta",
      "margen_s": 0,
      "fuente": "energia",
      "tipo": "corte_energia",
      "persona": null,
      "datos": {"afecta": ["COM", "DOR", "LAB-A", "LAB-B", "GEN", "ALM"], "nota": "RAD continua con baterias"}
    },
    {
      "id": "ev-0030",
      "evidencia_id": "raw-temperatura-0007",
      "ts": "2041-06-14T01:55:00Z",
      "precision": "exacta",
      "margen_s": 0,
      "fuente": "temperatura",
      "tipo": "temperatura",
      "persona": null,
      "datos": {"sensor": "T-ALM", "modulo": "ALM", "celsius": 2.9}
    },
    {
      "id": "ev-0037",
      "evidencia_id": "raw-energia-0003",
      "ts": "2041-06-14T02:15:00Z",
      "precision": "exacta",
      "margen_s": 0,
      "fuente": "energia",
      "tipo": "energia_restablecida",
      "persona": null,
      "datos": {}
    },
    {
      "id": "ev-0051",
      "evidencia_id": "raw-radio-0007",
      "ts": "2041-06-14T02:24:00Z",
      "precision": "minuto",
      "margen_s": 60,
      "fuente": "radio",
      "tipo": "radio",
      "persona": {"estado": "resuelta", "candidatos": ["MR-03"]},
      "datos": {"hablante": "Marta Ruiz", "texto": "La dejé en generadores a la una y media. Estaba bien.", "menciones": []}
    },
    {
      "id": "ev-0053",
      "evidencia_id": "raw-radio-0008",
      "ts": "2041-06-14T02:31:00Z",
      "precision": "minuto",
      "margen_s": 60,
      "fuente": "radio",
      "tipo": "radio",
      "persona": {"estado": "desconocida", "candidatos": []},
      "datos": {"hablante": "Luka", "texto": "Durante el corte escuché la puerta exterior. Pensé que era el viento.", "menciones": []}
    }
  ],
  "personal": [
    {"tarjeta": "JS-01", "nombre": "Tomás Okafor", "cargo": "Jefe de estación", "accesos": ["P2", "P3", "P4", "P5", "P7", "P8"]},
    {"tarjeta": "DV-02", "nombre": "Daniel Vidal", "cargo": "Médico", "accesos": ["P2", "P3"]},
    {"tarjeta": "MR-03", "nombre": "Marta Ruiz", "cargo": "Ingeniera eléctrica", "accesos": ["P2", "P4", "P5", "P7"]},
    {"tarjeta": "LP-04", "nombre": "Lucas Petrov", "cargo": "Radiooperador", "accesos": ["P2", "P5", "P7"]},
    {"tarjeta": "AK-05", "nombre": "Aiko Kimura", "cargo": "Bióloga", "accesos": ["P2", "P3"]},
    {"tarjeta": "IV-07", "nombre": "Irene Valcárcel", "cargo": "Glacióloga", "accesos": ["P2", "P3", "P4"]}
  ],
  "estacion": [
    {"puerta": "P1", "modulo_1": "COM", "modulo_2": "DOR", "lector": false},
    {"puerta": "P2", "modulo_1": "DOR", "modulo_2": "LAB-A", "lector": true},
    {"puerta": "P3", "modulo_1": "LAB-A", "modulo_2": "LAB-B", "lector": true},
    {"puerta": "P4", "modulo_1": "LAB-B", "modulo_2": "GEN", "lector": true},
    {"puerta": "P5", "modulo_1": "GEN", "modulo_2": "RAD", "lector": true},
    {"puerta": "P6", "modulo_1": "RAD", "modulo_2": "COM", "lector": false},
    {"puerta": "P7", "modulo_1": "GEN", "modulo_2": "ALM", "lector": true},
    {"puerta": "P8", "modulo_1": "ALM", "modulo_2": "EXT", "lector": true}
  ]
}