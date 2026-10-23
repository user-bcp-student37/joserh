# Procesamiento y estado confiable de pagos QR simulados

Como billetera digital que procesa pagos QR en un entorno simulado,
quiero identificar el comercio y su documento de deuda mediante el QR, validar el monto y estado de esa deuda y ejecutar el pago autorizado mediante un mock externo,
para mostrar un resultado confiable y persistido de cada transacción del cliente, sin integrar un proveedor real ni realizar extornos en este MVP.

## Criterios de aceptación

Cada criterio corresponde exclusivamente al paso del mismo número en el flujo de datos: seis criterios y seis pasos, sin flujos adicionales.

1. **Resolver el QR y validar la deuda.** El QR identifica `comercio_id`, `documento_deuda_id` y `documento_hash`. La billetera valida que correspondan al mismo documento, obtiene la cuenta destino del comercio y muestra el monto de deuda en PEN desde SQLite, sin permitir editarlo. Para un nuevo pago, la deuda debe estar pendiente, el monto ser positivo y las cuentas ser distintas y activas. Si falla la validación, no se habilita el pago ni se crea una transacción. **Tablas:** `cuentas_individuales` y `documentos_deuda` (lectura). **Estados:** cuentas `ACTIVA`/`INACTIVA`; deuda `PENDIENTE`, `EN_PROCESO` o `PAGADA`, siendo solo `PENDIENTE` pagable. **Flujo:** 1.
2. **Autorizar y registrar un único intento.** Al confirmar el pagador el comercio, documento e importe, se vuelve a validar la deuda y se reserva cambiándola a `EN_PROCESO`. Se registra el pago `PENDIENTE` y su historial inicial en una sola operación atómica. Esta autorización habilita el envío al mock, pero no implica que el pago esté aprobado. Los reenvíos se resuelven según el criterio 5. **Tablas:** `cuentas_individuales` (lectura), `documentos_deuda` (lectura y actualización), `transacciones` e `historial_transacciones` (inserción). **Estados:** deuda `PENDIENTE -> EN_PROCESO`; pago `PENDIENTE`; historial `NULL -> PENDIENTE`. **Flujo:** 2.
3. **Solicitar el resultado al mock externo.** Solo el intento autorizado y persistido se envía al mock, con comercio, documento e importe tomados de SQLite. El mock simula aprobación, error o pendiente. Una respuesta o ausencia de respuesta no modifica directamente las tablas; se mantiene pendiente hasta procesar su confirmación. **Tablas:** `transacciones` y `documentos_deuda` (lectura); `historial_transacciones` sin cambios en este paso. **Estados:** pago `PENDIENTE`, deuda `EN_PROCESO`; resultados del mock `APROBADA`, `FALLIDA` o `PENDIENTE`. **Flujo:** 3.
4. **Persistir la aprobación o falla y actualizar la deuda.** Una confirmación válida actualiza conjuntamente pago, deuda e historial: si el mock aprueba, el pago queda `APROBADA` y la deuda `PAGADA`; si falla, el pago queda `FALLIDA` y la deuda vuelve a `PENDIENTE`. Si alguna escritura falla, todas se revierten. Un resultado pendiente mantiene la reserva y no genera cambios ficticios. **Tablas:** `transacciones` y `documentos_deuda` (actualización), `historial_transacciones` (inserción); `cuentas_individuales` no se modifica. **Estados:** pago `PENDIENTE -> APROBADA/FALLIDA`; deuda `EN_PROCESO -> PAGADA/PENDIENTE`. **Flujo:** 4.
5. **Impedir cobros duplicados y cambios inválidos.** Una misma clave y datos devuelve el intento existente sin reenviarlo al mock; datos distintos con la misma clave se rechazan. Otro intento no puede cobrar una deuda en proceso o pagada. Los eventos repetidos, referencias desconocidas, datos de confirmación diferentes a los registrados y cambios de un pago terminal no generan nuevas escrituras. **Tablas:** `documentos_deuda`, `transacciones` e `historial_transacciones` (consulta, controles atómicos y restricciones). **Estados:** se conserva el pago `PENDIENTE`, `APROBADA` o `FALLIDA` y la deuda correspondiente; `APROBADA` y `FALLIDA` son terminales para ese intento. **Flujo:** 5.
6. **Mostrar el estado persistido y su historial.** La web responsive muestra comercio, documento, referencia de pago, importe, moneda, fecha, estado de deuda y estado e historial del pago, solo para la identidad de prueba autorizada. `APROBADA` se presenta en verde (`#15803D`) como "Éxito: aprobada"; `FALLIDA`, en rojo (`#B91C1C`) como "Error"; `PENDIENTE`, en ámbar (`#92400E`) como "Pendiente". El color no es el único indicador y la recarga conserva el resultado leído de SQLite. **Tablas:** `cuentas_individuales`, `documentos_deuda`, `transacciones` e `historial_transacciones` (lectura). **Estados:** pago `PENDIENTE`/`APROBADA`/`FALLIDA`; deuda `PENDIENTE`/`EN_PROCESO`/`PAGADA`. **Flujo:** 6.

## Datos SQLite y alcance del MVP

### Tablas y campos principales

Se utilizarán únicamente las siguientes cuatro tablas. Los identificadores serán de tipo `TEXT`, los importes de tipo `INTEGER` en céntimos y las fechas de tipo `TEXT` en formato ISO 8601 UTC. La única moneda habilitada será `PEN`.

| Tabla | Campos principales | Interacción con esta historia |
|---|---|---|
| `cuentas_individuales` | `id` (PK), `identificador` (único), `comercio_id` (único, nulo para cuentas no comerciales), `nombre_titular`, `moneda`, `saldo_simulado_centimos`, `estado`, `creada_en` | Identifica al pagador y la cuenta destino del comercio. Se precarga una sola cuenta por comercio, sin tabla adicional de comercios. `estado` admite `ACTIVA` o `INACTIVA`; las cuentas y sus saldos no se modifican. |
| `documentos_deuda` | `id` (PK), `comercio_id` (FK a `cuentas_individuales.comercio_id`), `hash_qr` (único), `monto_centimos`, `moneda`, `estado`, `creada_en`, `actualizada_en` | Fuente del comercio, importe fijo y habilitación del pago. `comercio_id` es obligatorio y el monto debe ser positivo. Estados: `PENDIENTE`, `EN_PROCESO` y `PAGADA`. El documento se precarga y solo cambia de estado durante este flujo. |
| `transacciones` | `id` (PK), `referencia` (única), `documento_deuda_id` (FK), `cuenta_origen_id` (FK), `cuenta_destino_id` (FK), `monto_centimos`, `moneda`, `estado_actual`, `clave_idempotencia` (única), `creada_en`, `actualizada_en` | Referencia `documentos_deuda.id` y ambas cuentas en `cuentas_individuales.id`. Guarda el importe y destino obtenidos del documento, sin modificaciones posteriores. Estados: `PENDIENTE`, `APROBADA` y `FALLIDA`. |
| `historial_transacciones` | `id` (PK), `transaccion_id` (FK), `estado_anterior`, `estado_nuevo`, `motivo`, `evento_mock_id` (único cuando exista), `registrado_en` | Referencia `transacciones.id` y conserva la creación y el resultado confirmado con el identificador del evento del mock. El historial no se sobrescribe ni elimina. |

El QR de prueba se genera por documento, no como QR estático de una cuenta. Contiene `{"version":2,"comercio_id":"<id comercio>","documento_deuda_id":"<id documento>","documento_hash":"<hash SHA-256 de 64 caracteres hexadecimales>"}`. Los identificadores y el hash se generan al precargar el documento; leer el QR no crea un comercio ni una deuda. El hash se obtiene de un valor aleatorio por documento, se guarda en `documentos_deuda.hash_qr` y se valida por coincidencia exacta junto con ambos identificadores. No se revierte para recuperar información ni autoriza por sí solo el pago. El QR no contiene monto, saldo ni credenciales: el monto se obtiene de SQLite y no puede editarlo el pagador. No se implementa un administrador de QR ni de documentos.

### Flujo de datos de esta historia

1. **Criterio 1 - QR, comercio y deuda.** Para un intento nuevo se valida la versión 2 del QR y se busca `documentos_deuda` por `id`, `comercio_id` y `hash_qr` coincidentes. Se obtiene la cuenta destino por `comercio_id` y la cuenta origen de la sesión. Se verifica deuda `PENDIENTE`, cuentas distintas y `ACTIVA`, moneda PEN y monto positivo. Se muestra comercio, documento y monto fijo para confirmar; un error de validación no crea un pago. Los reenvíos de un intento existente se identifican primero mediante el control del paso 5.
2. **Criterio 2 - Autorización y reserva atómicas.** Dentro de una transacción SQLite con claves foráneas habilitadas se aplica el control de clave del paso 5, se revalidan las cuentas y el documento y se cambia condicionalmente la deuda de `PENDIENTE` a `EN_PROCESO`. Solo si se reserva una fila se inserta el pago `PENDIENTE`, copiando documento, destino, importe y moneda, y su historial `NULL -> PENDIENTE` con `motivo = CREACION` y `evento_mock_id = NULL`. Se confirma todo junto o se revierte; únicamente el intento confirmado puede enviarse al mock.
3. **Criterio 3 - Pago externo simulado.** Se envían al mock `referencia`, `comercio_id`, `documento_deuda_id`, `monto_centimos` y `moneda` del intento reservado. El mock devuelve esos mismos datos, un `evento_mock_id` único por confirmación y `resultado` (`APROBADA`, `FALLIDA` o `PENDIENTE`). El mock no escribe en SQLite. Mientras no exista confirmación definitiva persistida, el pago sigue `PENDIENTE` y la deuda `EN_PROCESO`; no se permite otro cobro de esa deuda.
4. **Criterio 4 - Confirmación persistida.** Tras los controles del paso 5 y dentro de una única transacción SQLite, se actualiza condicionalmente el pago aún `PENDIENTE` y su deuda reservada `EN_PROCESO`. Una respuesta `APROBADA` deja pago `APROBADA` y deuda `PAGADA`; una `FALLIDA` deja pago `FALLIDA` y deuda `PENDIENTE`. Se actualizan fechas y se inserta el historial del pago con estados anterior/nuevo, evento y motivo `MOCK_APROBADA` o `MOCK_FALLIDA`. Si no se cumplen las condiciones o falla una escritura, se revierte todo. Una respuesta pendiente no cambia estados ni añade historial; no se modifican saldos.
5. **Criterio 5 - Control transversal.** Antes de iniciar un nuevo pago se consulta `clave_idempotencia`: si origen, documento, destino, monto y moneda coinciden con el intento registrado, se devuelve ese intento incluso si ya terminó; si difieren, se rechaza. Una reserva condicional de la deuda y un índice único parcial en `transacciones.documento_deuda_id` para estados `PENDIENTE` o `APROBADA` impiden pagos simultáneos o un segundo pago aprobado. Solo tras `FALLIDA` se permite un nuevo intento con nueva clave y deuda pendiente. Antes del paso 4 se comparan todos los datos de respuesta con el intento y se comprueban estado y unicidad del evento. Los duplicados y confirmaciones contradictorias no cambian pagos terminales, deuda ni historial. Este control se aplica dentro de los pasos 1, 2 y 4; no es un flujo adicional.
6. **Criterio 6 - Consulta responsive.** Se consulta la transacción permitida a la identidad de prueba y se relaciona con su documento, comercio y cuentas; el historial se lee por `transaccion_id`, ordenado por fecha e identificador. Se muestran estado de deuda y estado del pago con texto y colores: verde/Éxito: aprobada, rojo/Error y ámbar/Pendiente. Recargar repite las consultas a SQLite, sin presentar la respuesta directa del mock como resultado definitivo antes de persistirla.

### Límites de esta entrega

- El alcance se limita a seis criterios y seis pasos: QR de comercio y documento, validación y reserva de deuda, pago mock, persistencia, protección de duplicados y consulta responsive.
- Se precargan cuentas, identificadores de comercio, documentos con sus hashes QR e identidades de prueba. No se implementan altas o edición de comercios y deudas, pagos parciales, registro de usuarios, inicio de sesión nuevo ni pantallas de administración del mock.
- La validación de la deuda autoriza iniciar el pago; solo la confirmación `APROBADA` del mock persistida aprueba la transacción y marca la deuda como `PAGADA`. Son momentos distintos y ninguno dispara un extorno.
- Se adopta la opción 1: extornos y devoluciones fuera del MVP, tanto antes como después de la aprobación. No se incorporan reclamos, quejas ni disputas.
- Los saldos son informativos. No se incluyen validación de fondos, débitos o abonos, libro de movimientos, comisiones, conversión de moneda ni conciliación bancaria.
- Una confirmación pendiente o ausente mantiene la deuda reservada; no se libera por tiempo ni se asume un fallo. No se implementan expiración automática, cancelación ni recuperación automática del envío al mock en esta entrega.
- Esta sección documenta el comportamiento mínimo y las cuatro tablas; no crea archivos SQLite, migraciones ni código de implementación.

## Contexto

- Épica: [MVP de wallet en Perú](../epic/peru-wallet-mvp.sp.md).
- El mock externo es un servicio controlado que simula al proveedor sin conectarse a una empresa real ni mover dinero real.
- Un estado confiable coincide con el resultado validado del mock, el estado de la deuda y el historial persistidos de forma consistente en SQLite; no representa una confirmación financiera real.
- No se administra la caja del comercio ni se decide la entrega de productos o servicios.

## Supuestos

- El mock reproduce escenarios de aprobación, falla y pendiente y conserva la misma referencia para cada intento.
- El documento es una deuda de pago único por su importe total en PEN; no se modifica su comercio, monto ni moneda durante los intentos de pago.
- El MVP es una web responsive para móviles, tabletas y escritorio, con etiquetas de estado además del color y sin aplicaciones nativas.

## Preguntas abiertas

- ¿Qué cuentas, comercios, documentos, hashes QR e identidades de prueba se precargarán para validar los tres escenarios del mock?
