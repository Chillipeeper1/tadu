<p align="right"><a href="../challenges.md">🇬🇧 English</a> · <b>🇲🇽 Español</b></p>

# Retos de ingeniería

Casos reales del proyecto, contados por la causa y no solo por el síntoma. Cada uno cambió la
forma de trabajar.

---

## 1. Un CI en verde que no había corrido ninguna prueba

**Situación.** Una rama de trabajo quedó en rojo justo después de fusionar un PR que había pasado
todas las comprobaciones.

**Problema.** El trabajo que falló se llamaba "frontend (lint, tipos, tests, build)", así que
parecía un test roto. No lo era: **no llegó a correr ninguna prueba**. Murió en el paso 2 de 5,
la verificación de formato, por una línea en blanco que faltaba en un archivo de documentación.

**Causa.** El paso de formato revisa **todo el repositorio**, pero el filtro que decide qué
trabajos se ejecutan solo miraba el código. Un PR de pura documentación no coincidía con
ninguna ruta, el trabajo se saltó y la compuerta tradujo "saltado" como "éxito".

**Lo que lo hizo invisible.** El filtro no compara lo mismo en un PR que en un _push_: en un PR
compara contra la rama base; en un _push_, contra la rama por defecto del repositorio. El
contenido era idéntico y el resultado fue verde como PR y rojo como _push_.

**Resultado.** El filtro ahora incluye todo lo que alguna herramienta de verificación mira. La
regla general quedó escrita: _si una herramienta mira un archivo, tiene que estar en el filtro._

**Lección.** Antes de buscar el test roto, mirar en qué paso murió el trabajo.

---

## 2. Un test que fallaba por la noche

**Situación.** Un test pasaba toda la mañana y empezó a fallar esa misma noche, sin que nadie
hubiera tocado código.

**Causa.** Un dato de prueba con una fecha fija se comparaba contra la fecha actual en UTC. En
cuanto el reloj UTC cruzó esa fecha, que en zonas al oeste ocurre horas antes de la medianoche
local, el dato pasó de "vigente" a "vencido".

**Resultado.** Las fechas de prueba que significan "vigente" o "vencido" se calculan ahora
relativas al momento de ejecución, nunca como literales.

---

## 3. El precio lo decide el servidor

**Problema.** El cliente enviaba al backend el precio y la distancia del viaje. Un cliente
modificado podía pagar menos.

**Solución.**

- El servidor calcula la distancia y aplica la tabla tarifaria oficial, con límites mínimo y
  máximo.
- Al cobrar, el importe sale del precio que el servidor guardó al crear el viaje, **no** del que
  manda el cliente.
- Antes de cobrar se comprueba que el viaje pertenece a quien paga (protección contra IDOR).
- Si el viaje ya tiene un pago aprobado, un reintento responde éxito sin volver a cobrar.

Todo esto con pruebas que cubren precios manipulados, viajes ajenos y reintentos.

---

## 4. MFA que solo existía en la pantalla

**Problema.** El panel de administración pedía segundo factor al iniciar sesión, pero el
backend no lo comprobaba. Alguien con correo y contraseña podía hablar directamente con el
servicio de autenticación y llamar a la API de administración sin pasar por el desafío.

**Solución.** La función que autoriza a los administradores verifica ahora, en el servidor, que
el token provenga de una sesión con segundo factor. Sin él, la API responde 403.

**Lección.** Un control que solo vive en la interfaz no es un control.

---

## 5. Lo que filtraría un sistema de errores

**Situación.** Al añadir Sentry para observabilidad, apareció un riesgo que el plan no
contemplaba: por defecto adjunta las **variables locales** de cada marco de la pila, y su
depurador de datos solo tapa claves con nombre exacto. Una variable local llamada como el token
de pagos habría viajado en claro.

**Solución.** Se desactivó el envío de variables locales y se centralizó la política de datos
personales en un único módulo, compartido por todas las funciones mediante una capa de Lambda.

**Efecto secundario evitado.** Las pruebas borran la variable de configuración de Sentry antes
de correr: quien la tuviera exportada para desplegar habría enviado los errores de sus pruebas
al proyecto de producción. Se verificó forzando la variable.

---

## 6. Calidad medida por historial, no por foto

**Situación.** Se adoptó CodeScene como compuerta de PR. A diferencia de un linter, no busca
errores: mide **dónde duele el código que seguimos tocando**, cruzando salud del archivo y
frecuencia de cambio.

**Lo que enseñó.**

- Un arreglo de corrección bajó un archivo de 7.89 a 7.87 y bloqueó un PR que mejoraba otros
  siete. La respuesta no fue suprimir la regla, sino extraer lo mínimo hasta que la función
  quedó más corta que antes del cambio.
- **Mover código no es mejorarlo.** Sacar casi 800 líneas de estilos a otro archivo mejoró la
  nota del original y aun así se rechazó el PR: el muro seguía siendo el mismo muro.
- A veces el mensaje señala en la dirección contraria. Una función con **menos** puntos de
  decisión que antes bajó de nota solo porque un auxiliar pasó de estar dentro del componente a
  estar fuera dentro del mismo archivo. Moraleja: contar los puntos de decisión antes de creerle
  al mensaje.
- Antes de interpretar los números hubo que **limpiar el análisis**: cinco correos de autor para
  dos personas (incluidos 167 _commits_ con un correo ilegible) distorsionaban la propiedad del
  código y el factor de bus. Se resolvió con un archivo de mapeo de identidades.

---

## 7. Menos llamadas a servicios de pago por uso

**Mapbox.** Auditar el seguimiento de viajes mostró que cada viaje real pedía una ruta desde un
punto aleatorio para calcular un "llega en X minutos" que era un dato inventado. La simulación
se limitó a las demos, se dejó de reintentar los errores que siempre fallan (los definitivos)
y se unificaron dos implementaciones del cliente en una. El inventario de lo que falta
optimizar quedó documentado.

**Google Places.** El autocompletado sin token de sesión cobra cada predicción. La librería
generaba el token pero, en el modo que usa el proyecto, no lo enviaba. Se conectó, se limitaron
los campos que pide el detalle de cada lugar y se redujo la verificación de coordenadas a lugares
y no a direcciones. Falta confirmar el efecto en la facturación una vez desplegado, y está
anotado así.

---

## 8. Un pin que cae sobre la calle

**Problema.** El punto de recogida era el centro del mapa tal cual: podía quedar dentro de un
parque, a media manzana o sobre un edificio, y el chofer no sabía por qué calle llegar.

**Solución.** Cada vez que el mapa se detiene, el punto se ajusta a la calle transitable más
cercana, como en Uber o DiDi. Mapbox no ofrece un servicio de "punto más cercano", pero sí ajusta
a la red vial cada punto de una ruta, así que se pide una ruta del punto a sí mismo y se lee el
resultado.

**Decisión incómoda.** Si el ajuste falla, no se deja confirmar el pin. Sin él no hay garantía de
que esté sobre una calle, así que para esta función Mapbox dejó de ser una dependencia opcional.

**Detalle de ingeniería.** La lógica de "solo cuenta el último ajuste pedido" y "no entrar en un
bucle de ajustar y mover el mapa" vive en una clase sin React, con sus propias pruebas.

---

## 9. Documentación que apuntaba al estado anterior

Una pasada de limpieza encontró que el documento de diseño mandaba una pieza de infraestructura a
una copia muerta de una función, que dos documentos listaban como "deuda aceptada" algo que ya
no existía, y que se afirmaba un _fallback_ que ya había sido eliminado. La regla que salió:
**cuando un arreglo cambia un contrato, un nombre o un comportamiento, quien lo mencione se
corrige en el mismo cambio.** Una documentación que describe un bug es peor que ninguna, porque
se le cree.

---

## Cómo se trabaja

Estos casos moldearon un flujo que se aplica a cada tarea, por pequeña que parezca:

1. Leer el CHANGELOG y compararlo con el repositorio real.
2. Hacer el cambio.
3. Correr la compuerta completa (`verify`).
4. Revisar todo el diff buscando lo que la compuerta no atrapa: código muerto, depuración
   olvidada, decisiones que las siguientes dejaron sin sentido.
5. Probar lo que se puede probar y **decir explícitamente lo que no** (comportamiento nativo,
   cobros reales, dispositivos).
6. Abrir el PR. El CHANGELOG se escribe al final, cuando ya describe lo que de verdad quedó.

Y una regla sin excepciones: **nunca hacer que algo parezca que funciona.** Nada de ajustar el
test a lo que el código hace, silenciar la regla o bajar el umbral para que deje de fallar.
