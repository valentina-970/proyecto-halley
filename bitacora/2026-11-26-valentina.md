# 2026-11-26 · Valentina

## Comunicación entre los cuatro servicios

- **Problema.** Haskell, Prolog, Java y Elixir corren como procesos separados y deben intercambiar eventos, snapshots y respuestas.
- **Alternativas.** (1) HTTP con cuerpos JSON. (2) Sockets TCP con un JSON por línea (descartada: habría que escribir el protocolo de mensajes, los timeouts y el manejo de errores en cuatro lenguajes).
- **Decisión.** HTTP + JSON con los puertos 8080 a 8083 y el contrato de `contrato/README.md`. Los cuatro lenguajes traen servidor y cliente HTTP, y un servicio caído se ve como un error de conexión.
- **Prueba.** Los ejemplos de `contrato/ejemplos/` se validan con `python -m json.tool` o en cualquier visor JSON; cada servicio se prueba primero contra ellos.
