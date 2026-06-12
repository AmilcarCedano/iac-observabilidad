# Respuestas a las preguntas del laboratorio

**Alumno:** Amilcar Cedano
**Curso:** Infraestructura como Código
**Laboratorio:** Grafana, Prometheus y Loki

---

## 1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos /metrics?

Porque cada uno guarda cosas distintas. Prometheus solo guarda números que van cambiando
en el tiempo, por ejemplo el % de CPU o cuántas peticiones llegan por segundo. Con eso
puedo ver que algo anda mal, pero no me dice el detalle de qué pasó. Los logs en cambio
son el texto de cada evento: qué pedido falló, qué error salió, con qué código. Prometheus
no guarda texto, no está hecho para eso. Ahí entra Loki, que guarda todas las líneas de log
y me deja filtrarlas por etiquetas, por ejemplo `{tier="application"} | json | level="ERROR"`
para ver solo los errores de la aplicación. En la práctica los uso juntos: con Prometheus me
doy cuenta de que algo subió o se cayó, y con Loki reviso los logs para entender por qué.

## 2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?

Que todo es reproducible y no dependo de acordarme de los clics que hice. En este laboratorio
los datasources de Prometheus y Loki ya estaban configurados en archivos dentro de
`grafana/provisioning/`, así que cuando levanté el stack con `docker compose up -d --build`
ya aparecían listos en Grafana sin que yo creara nada a mano. Eso me sirvió de verdad: cuando
algo falla puedo hacer `docker compose down -v` y volver a levantar todo igualito en minutos.
Además, como la configuración está en archivos, queda versionada en Git: si alguien la cambia
se ve en el historial y se puede regresar. Esa es justamente la idea de infraestructura como
código que vimos en el curso.

## 3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?

Lo comprobé en mi propia prueba: cuando generé carga, el panel del backend marcó más de 200%
mientras que el del host apenas llegó a 25%. La diferencia es que miden cosas distintas. El
panel del contenedor mide solo lo que consume el proceso del backend (100% es más o menos un
núcleo completo, por eso puede pasar de 100 si usa más de uno). El panel del host promedia
todos los núcleos de mi laptop y todos los programas que tengo abiertos, entonces aunque el
backend esté ahogado, el promedio general casi ni se mueve porque los demás núcleos están
libres. Para alertar sobre una aplicación concreta usaría la métrica del contenedor/proceso,
porque es la que refleja solo esa aplicación. Si alertara con la del host, podría tener la
app saturada y la alarma nunca sonaría.

## 4. ¿Qué diferencia hay entre el evaluation interval y el pending period de una alarma?

El evaluation interval es cada cuánto Grafana revisa la condición. En mi regla lo puse en 10
segundos, o sea cada 10s ejecuta la consulta y revisa si la CPU pasó de 50. El pending period
es cuánto tiempo seguido tiene que cumplirse la condición para que la alarma pase a Firing.
Yo lo puse en 30s, entonces la regla primero queda en "Pending" y recién si la CPU se mantiene
arriba de 50 durante 30 segundos pasa a "Firing" (lo vi en la práctica: salió "Firing for 31s"
después de generar la carga). El pending period sirve para no alarmar por gusto: si la CPU da
un pico de 5 segundos y baja, la alarma no llega a dispararse porque el pico no se sostuvo.
