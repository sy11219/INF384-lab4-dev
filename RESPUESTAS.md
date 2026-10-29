# Respuestas · Laboratorio 4
Alumno: 20221377
Presupuesto: [13 ] min  Métrica DORA: [change failure rate]

## Parte 1 · Diagnóstico
### Defecto 1
- Job y etapa: smoke, en la etapa deploy.
- Problema que causa en el pipeline: Corre en paralelo con el despliegue (tiene needs:build-push en vez de needs:deploy), cuando debería correr después. Además, no recibe la versión esperada, por lo que no verifica qué versión corre.
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa y ajustar configuración (needs: deploy y EXPECTED_COMMIT: ${{ github.sha }}).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
### Defecto 2
- Job y etapa: load, en la etapa post-deploy.
- Problema que causa en el pipeline: la carga de unos 12 minutos (1 m + 10 m + 1 m con 30 usuarios virtuales) corre en cada PR y en cada push, cuando una prueba de carga no debería ir en cada commit. Además corre en paralelo con e2e contra la misma función, y modifica saldos mientras las E2E verifican.
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa (needs: e2e e if: github.event_name != 'pull_request').
- Evidencia (dato observado, con link al run o fragmento de la salida local):
### Defecto 3
- Job y etapa: e2e y load, en la etapa post-deploy.
- Problema que causa en el pipeline: desperdicia tiempo y entrega credenciales de más. smoke ya obtiene la dirección del despliegue (la Function URL). Aun así, e2e y load vuelven a configurar cada uno las credenciales de AWS y a repetir la búsqueda de la URL, un bucle de hasta 30 intentos cada 10 segundos. Son pasos duplicados en cada run, y le dan acceso a AWS a dos jobs que solo necesitan una dirección para hacer sus llamadas. La dirección del despliegue debe pasarse entre jobs como variable.
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración. smoke expone la URL como output (outputs.url). e2e y load la reciben con needs.smoke.outputs.url y ya no configuran credenciales ni buscan la URL. load pasa a needs: [smoke, e2e] para poder leer el output de smoke.
- Evidencia (dato observado, con link al run o fragmento de la salida local):
### Defecto 4
- Job y etapa: e2e, en la etapa post-deploy.
- Problema que causa en el pipeline: Desperdicia tiempo por instalar Chromium con --with-deps con pruebas que solo llaman a HTTP. Además, "transferir con monto negativo es 400" repite lo que ya validan “validarMonto rechaza un monto negativo” y “validarMonto rechaza cero”.
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): eliminar (el paso “Navegador”, la prueba con page.goto y la prueba de monto negativo) y modificar tests (cambiar el fixture page por request).
- Evidencia (dato observado, con link al run o fragmento de la salida local):
### Defecto 5
- Job y etapa: integration, en la etapa pre-build.
- Problema que causa en el pipeline: prueba de forma incorrecta: las pruebas dependen del estado que dejó la anterior (“el saldo de ana es 900” solo pasa si antes corrió “ana transfiere 100”, etc.). Solo funcionan sobre una base nueva y el resultado depende del orden, no del código.
- Link al commit:
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): modificar tests. Cada prueba usa su propia base (DB_PATH único en beforeEach) y prepara sus propias precondiciones.
- Evidencia (dato observado, con link al run o fragmento de la salida local):

## Parte 2 · Implementación
- Link al run en verde en main:
- Link al pull request:
- Defecto de la aplicación introducido (archivo y descripción):
- Job que debería detenerlo:
- Link al run fallido:

## Parte 3 · Decisión bajo restricción
Link al run medido:

| Job | Duración (min:seg) |
|---|---|
| unit | |
| integration | |
| sast | |
| sca | |
| build-push | |
| deploy | |
| smoke | |
| e2e | |
| load | |
| **Total** | |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | |
| p95 | |
| Máximo | |
| Tasa de error | |
| Requests por segundo | |

Conclusión sobre el comportamiento bajo carga:

SLO propuesto (percentil, valor, ventana) y justificación:

SLA propuesto y diferencia con el SLO:

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|

Total resultante: [ ] min
Link al run en main con los cambios:

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:

## Declaración de uso de IA
Herramientas usadas y para qué:
