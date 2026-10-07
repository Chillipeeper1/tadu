<p align="right"><a href="../decisions.md">🇬🇧 English</a> · <b>🇲🇽 Español</b></p>

# Decisiones de arquitectura

Siete decisiones con su contexto y, sobre todo, **lo que se sacrificó** al tomarlas. Son una
versión resumida de los ADR del proyecto.

| #   | Decisión                                                         |
| --- | ---------------------------------------------------------------- |
| 1   | Backend serverless en lugar de un servidor tradicional           |
| 2   | DynamoDB con una sola tabla                                      |
| 3   | Asignación FIFO automática, sin oferta de aceptar o rechazar     |
| 4   | Tokenizar la tarjeta en el cliente                               |
| 5   | Context API en lugar de Redux o Zustand                          |
| 6   | Google Maps para dibujar, Mapbox para calcular rutas             |
| 7   | Actualizaciones OTA atadas a la versión de la app                |

---

## 1. Serverless en lugar de un servidor

**Contexto.** Equipo de dos personas y una carga muy desigual: picos cuando alguien pide un taxi
y silencio el resto del tiempo. Se necesitaba infraestructura de producción sin dedicar tiempo a
operarla.

**Decisión.** Lambda + API Gateway + DynamoDB, definidos como código con AWS SAM.

**Se sacrifica.** Arranques en frío, depuración local más incómoda y dependencia de AWS.

**Se gana.** Escala automática, cero servidores que mantener y pago solo por uso.

## 2. DynamoDB con una sola tabla

**Contexto.** Los patrones de acceso eran pocos y bien conocidos: fila de choferes por base,
historial por pasajero, historial por chofer y detección de sesiones inactivas.

**Alternativas.** Varias tablas en DynamoDB, o una base relacional serverless.

**Decisión.** Una tabla con cuatro índices secundarios, uno por patrón.

**Se sacrifica.** El modelo hay que planearlo de antemano, no hay _joins_ y los reportes ad hoc
son incómodos sin exportar los datos.

**Se gana.** Latencia predecible a cualquier escala y transacciones atómicas entre entidades
distintas sin salir de la tabla, que es justo lo que necesita la asignación de viajes.

## 3. Asignación FIFO sin oferta

**Contexto.** Las apps de movilidad suelen mostrar la solicitud al chofer y dejarle aceptar o
rechazar. En un sitio de taxis el chofer ya está esperando físicamente su turno.

**Decisión.** Al pedir un viaje, el sistema asigna directamente al primero de la fila.

**Se sacrifica.** El chofer no puede declinar un viaje concreto. Su única salida es hacer
_check-out_.

**Se gana.** Se elimina el temporizador de oferta, la reasignación por rechazo y los estados
intermedios. El flujo es más corto y tiene menos puntos donde fallar.

## 4. Tokenizar la tarjeta en el cliente

**Contexto.** Manejar números de tarjeta en el propio backend exige certificación PCI completa,
algo desproporcionado para un MVP.

**Decisión.** La app obtiene un token directamente del procesador de pagos y el backend solo
recibe ese token.

**Se sacrifica.** Parte del flujo de cobro queda en el cliente, así que el servidor no puede
fiarse de lo que este le diga: el importe se recalcula siempre del lado del servidor.

**Se gana.** El nivel de cumplimiento más simple (SAQ-A) y ningún dato de tarjeta pasa por
servidores propios.

## 5. Context API en lugar de Redux o Zustand

**Contexto.** Hacía falta compartir sesión, perfil y estado del viaje entre pantallas.

**Decisión.** Contextos de React y estado local, sin librería de estado.

**Se sacrifica.** Herramientas de depuración del historial de estado, y los contextos anidados
pueden volverse verbosos si la app crece mucho.

**Se gana.** Una dependencia menos y menos superficie que mantener.

## 6. Google Maps para dibujar, Mapbox para rutas

**Contexto.** Dos necesidades distintas: mostrar un mapa conocido para el usuario y calcular
rutas, distancias y ajustes de pin a la calle.

**Decisión.** Proveedor mixto.

**Se sacrifica.** Dos SDKs, dos cuentas, dos claves y doble atribución obligatoria.

**Se gana.** El costo de calcular rutas se mantiene bajo en volúmenes de MVP, y la interfaz
sigue usando el mapa que el usuario ya conoce.

**Seguimiento posterior.** Más tarde se midieron las llamadas a ambos servicios y se redujeron
sin cambios visibles para el usuario (ver [`retos.md`](retos.md)).

## 7. Actualizaciones OTA atadas a la versión de la app

**Contexto.** Se quería poder publicar correcciones de JavaScript sin recompilar el binario
nativo, y decidir qué actualizaciones son compatibles con qué compilaciones.

**Alternativa.** Calcular una huella (_fingerprint_) del código nativo en cada compilación.

**Decisión.** Compatibilidad por versión de la app, que se sube a propósito.

**Se sacrifica.** Si alguien añade un módulo nativo y olvida subir la versión, una actualización
podría llegar a un binario sin ese módulo y fallar en ejecución.

**Se gana.** Control explícito y predecible sobre cuándo se corta la compatibilidad.
