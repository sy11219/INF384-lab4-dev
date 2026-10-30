# Respuestas · Laboratorio 4
Alumno: 20221377
Presupuesto: [13 ] min  Métrica DORA: [change failure rate]

## Parte 1 · Diagnóstico
### Defecto 1
- Job y etapa: smoke, en la etapa deploy.
- Problema que causa en el pipeline: Corre en paralelo con el despliegue (tiene needs:build-push en vez de needs:deploy), cuando debería correr después. Además, no recibe la versión esperada, por lo que no verifica qué versión corre.
- Link al commit: https://github.com/sy11219/INF384-lab4-dev/commit/f7991b842c9df3c49ece7cf006d1d75bf65288f6
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa y ajustar configuración (needs: deploy y EXPECTED_COMMIT: ${{ github.sha }}).
- Evidencia (dato observado, con link al run o fragmento de la salida local): Antes (https://github.com/sy11219/INF384-lab4-dev/actions/runs/40962708861): smoke inicia a las 22:03:39 GMT, antes de que deploy termine a las 22:03:54 GMT.
  En el paso Smoke solo aparece la línea `salud:` y no la línea `commit:`: nada verifica la versión.
  Después (https://github.com/sy11219/INF384-lab4-dev/actions/runs/40966463356/job/123772953334):
  smoke corre después de deploy y confirma que responde el commit del run (b5e181b):
  salud: {"ok":true,"version":"1.0.0","commit":"b5e181b4af3beaa61cb4df8471c06f9429ab977a"}
  commit: b5e181b4af3beaa61cb4df8471c06f9429ab977a
### Defecto 2
- Job y etapa: load, en la etapa post-deploy.
- Problema que causa en el pipeline: la carga de unos 12 minutos (1 m + 10 m + 1 m con 30 usuarios virtuales) corre en cada PR y en cada push, cuando una prueba de carga no debería ir en cada commit. Además corre en paralelo con e2e contra la misma función, y modifica saldos mientras las E2E verifican.
- Link al commit: https://github.com/sy11219/INF384-lab4-dev/commit/4a69a4e9f43c788170438bdb94b66f5d5673a7c1
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa (needs: e2e e if: github.event_name != 'pull_request').
- Evidencia (dato observado, con link al run o fragmento de la salida local): Antes (https://github.com/sy11219/INF384-lab4-dev/actions/runs/40962708861): en el grafo, e2e y load dependen de smoke y corren en paralelo;
  load dura 12:12 y corre también en pull requests.
  Después (https://github.com/sy11219/INF384-lab4-dev/actions/runs/40966463356): e2e termina a
  las 22:33:00 GMT y load inicia a las 22:33:06 GMT; load corre después de e2e.
  En el run del pull request (https://github.com/sy11219/INF384-lab4-dev/pull/1), load aparece como skipped.
### Defecto 3
- Job y etapa: e2e y load, en la etapa post-deploy.
- Problema que causa en el pipeline: desperdicia tiempo y entrega credenciales de más. smoke ya obtiene la dirección del despliegue (la Function URL). Aun así, e2e y load vuelven a configurar cada uno las credenciales de AWS y a repetir la búsqueda de la URL, un bucle de hasta 30 intentos cada 10 segundos. Son pasos duplicados en cada run, y le dan acceso a AWS a dos jobs que solo necesitan una dirección para hacer sus llamadas. La dirección del despliegue debe pasarse entre jobs como variable.
- Link al commit: https://github.com/sy11219/INF384-lab4-dev/commit/b5e181b4af3beaa61cb4df8471c06f9429ab977a
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración. smoke expone la URL como output (outputs.url). e2e y load la reciben con needs.smoke.outputs.url y ya no configuran credenciales ni buscan la URL. load pasa a needs: [smoke, e2e] para poder leer el output de smoke.
- Evidencia (dato observado, con link al run o fragmento de la salida local): Antes (https://github.com/sy11219/INF384-lab4-dev/actions/runs/40962708861): e2e y load ejecutan cada uno "Run aws-actions/configure-aws-credentials@v6"
  y "URL", los mismos pasos que ya ejecutó smoke.
  Después (https://github.com/sy11219/INF384-lab4-dev/actions/runs/40966463356): e2e y load ya no tienen esos pasos; el paso Playwright recibe la URL desde smoke.
### Defecto 4
- Job y etapa: e2e, en la etapa post-deploy.
- Problema que causa en el pipeline: Desperdicia tiempo por instalar Chromium con --with-deps con pruebas que solo llaman a HTTP. Además, "transferir con monto negativo es 400" repite lo que ya validan “validarMonto rechaza un monto negativo” y “validarMonto rechaza cero”.
- Link al commit: https://github.com/sy11219/INF384-lab4-dev/commit/3af725d89ac0192c4bf3aa233f4c5ff610dc5d52
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): eliminar (el paso “Navegador”, la prueba con page.goto y la prueba de monto negativo) y modificar tests (cambiar el fixture page por request).
- Evidencia (dato observado, con link al run o fragmento de la salida local): Antes (https://github.com/sy11219/INF384-lab4-dev/actions/runs/40962708861): e2e ejecuta el paso "Navegador" (30s) y el job dura 52s.
  Después (https://github.com/sy11219/INF384-lab4-dev/actions/runs/40966463356): e2e ya no tiene el
  paso "Navegador"; el job dura 16s.
### Defecto 5
- Job y etapa: integration, en la etapa pre-build.
- Problema que causa en el pipeline: prueba de forma incorrecta: las pruebas dependen del estado que dejó la anterior (“el saldo de ana es 900” solo pasa si antes corrió “ana transfiere 100”, etc.). Solo funcionan sobre una base nueva y el resultado depende del orden, no del código.
- Link al commit: https://github.com/sy11219/INF384-lab4-dev/commit/e2db37fc09e77498f1b3862690da7a83345bd9e0
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): modificar tests. Cada prueba usa su propia base (DB_PATH único en beforeEach) y prepara sus propias precondiciones.
- Evidencia (dato observado, con link al run o fragmento de la salida local): Antes: correr el test dos veces seguidas causa error: ✖ diego transfiere todo su saldo a elena (0.367ms). Después: todas las pruebas pasan. ✔ ana transfiere 100 a beto (30.0221ms)
✔ el saldo de ana refleja el debito (17.9697ms)
✔ el saldo de beto refleja el credito (15.5728ms)
✔ el historial de ana registra la transferencia (16.3616ms)
✔ ana no puede transferir mas que su saldo (14.5527ms)
✔ una transferencia rechazada no cambia saldos (13.824ms)
✔ diego transfiere todo su saldo a elena (15.3189ms)
✔ diego sin saldo no puede transferir (14.7782ms)
✔ el historial de elena muestra solo la transferencia de diego (17.1169ms)
✔ transferir al mismo usuario es 422 (14.2757ms)
✔ transferir a un usuario inexistente es 400 (16.9099ms)
✔ un cuerpo que no es JSON es 400 (0.3701ms) 
(Se hizo node --test test/integration/transferir.test.js)

## Parte 2 · Implementación
- Link al run en verde en main: https://github.com/sy11219/INF384-lab4-dev/actions/runs/40966463356
- Link al pull request: https://github.com/sy11219/INF384-lab4-dev/pull/1
- Defecto de la aplicación introducido (archivo y descripción): src/db.js fue editado en la línea 49. Se resta en vez de sumar al acreditar.
- Job que debería detenerlo: integration (etapa pre-build). El defecto está en la consulta SQL de db.js. Las unitarias usan un store falso en memoria y su cobertura solo incluye wallet.js y util.js, así que no ejecutan ese SQL ("el mock lo oculta", S7 pág. 10). Las pruebas sobre el despliegue tampoco lo verían: smoke no transfiere y e2e solo valida la respuesta de /transferir, que sigue siendo correcta. La integración ejecuta el handler contra un SQLite real y verifica los saldos guardados.
- Link al run fallido: https://github.com/sy11219/INF384-lab4-dev/actions/runs/40967542738

## Parte 3 · Decisión bajo restricción
Link al run medido: https://github.com/sy11219/INF384-lab4-dev/actions/runs/40966463356

| Job | Duración (min:seg) |
|---|---|
| unit |00:12|
| integration |00:16|
| sast |00:16|
| sca |00:19|
| build-push |00:49|
| deploy |00:17|
| smoke |00:07|
| e2e |00:16|
| load |12:07|
| **Total** |14:08|

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 |61.55 ms|
| p95 |63.87 ms|
| Máximo |382.04 ms|
| Tasa de error |0.00% (0 de 35608)|
| Requests por segundo |49.41 req/s|

Conclusión sobre el comportamiento bajo carga: Con 30 usuarios virtuales sostenidos durante 10 minutos (35 608 requests, 49.4 req/s), Monedero Andino responde de forma estable: la mediana es 56.7 ms y el p95 63.9 ms, apenas 7 ms por encima, sin errores (0 de 35 608). El máximo es un valor aislado.

SLO propuesto (percentil, valor, ventana) y justificación: p95 < 150 ms, medido en una ventana móvil de 28 días sobre /transferir y /saldo. Esto considerando el p95 medido de 63.9 ms, deja un margen para absorber la variación entre runs.

SLA propuesto y diferencia con el SLO: SLA para clientes: el 95% de las operaciones responde en menos de 500 ms y la disponibilidad mensual es de al menos 99.5%.

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|Perfil de carga 30s + 4m + 30s|load|7|

Total resultante: 07:09 min
Link al run en main con los cambios: https://github.com/sy11219/INF384-lab4-dev/actions/runs/40976505209

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada): Con menor tiempo de carga, se acumulan menos transacciones, por lo que la latencia no crece tanto. Con datos reales, es posible que se acabe incumpliendo el SLO, por lo que se consideraría fallido el despliegue. Por ello, la métrica DORA aumentaría.

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón: sast se queda, ya que corre en paralelo con los primeros jobs y dura poco (23s), por lo que no afecta el tiempo total. Por otra parte, como sast evita que código inseguro se despliegue, quitarlo haría que el código llegue a producción, aumentando el CFR. Si se quitara, el ahorro en tiempo no justificaría empeorar la métrica.	

## Declaración de uso de IA
Herramientas usadas y para qué:
Claude para ayuda en general y para definir el defecto a introducir en la pregunta 2.