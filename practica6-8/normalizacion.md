# Ejercicio 1. Pautas de diseño

Revisando cada tabla contra las cuatro pautas informales: **1) Semántica clara**, **2) Sin anomalías**, **3) Pocos nulos** y **4) Sin tuplas espurias**.

| Tabla | Descripcion: Cada fila es… | ¿Qué pauta falla y por qué? |
| :--- | :--- | :--- |
| **`cliente`** | Cada fila es un cliente empresarial (nacional o de exportación) con sus datos de contacto y su información fiscal. | **Falla Pauta 3 (pocos nulos)** y parcialmente **Pauta 1 (semántica)**: al meter los dos subtipos en una sola tabla con `tipoCliente`, los 60 clientes nacionales dejan `idFiscalGlobal` y `paisReceptor` en `NULL`, y los 20 de exportación dejan `RFC`, `regimenFiscal` y `estadoOrigen` en `NULL` (180 celdas vacías). |
| **`transportista`** | Cada fila es una empresa transportista con su contacto, si opera nacional y/o internacionalmente, y sus permisos fiscales o aduanales. | **Falla Pauta 3 (pocos nulos)**: al juntar las modalidades nacional e internacional en una tabla, los 20 transportistas solo nacionales tienen `codigoAduanal` y `agenciaAduanal` en `NULL`, y los 6 solo internacionales tienen `RFC` en `NULL`. |
| **`producto`** | Cada fila es una presentación de aceite industrial del catálogo, con su modelo, línea comercial, costo, precio de lista y proveedor. | **Falla Pautas 1, 2 y 3**:<br>• **Pauta 1 y 2 (semántica y anomalías):** mezcla la presentación (`claveProducto`) con la regla de que cada modelo pertenece a una línea (`modeloProducto -> tipoProducto`). Como hay 4 presentaciones por modelo, `tipoProducto` se repite 4 veces: si cambia la línea de un modelo hay que editar 4 filas (*actualización*), no se puede registrar un modelo nuevo sin inventar una presentación (*inserción*) y si se borran sus 4 presentaciones se pierde a qué línea pertenecía el modelo (*borrado*).<br>• **Pauta 3 (pocos nulos):** de las 3 columnas de proveedor, siempre 2 quedan en `NULL` en cada fila (80 nulos en 40 filas). |
| **`sucursal`** | Cada fila es una planta o bodega receptora de un cliente, con el teléfono de su encargado y su dirección completa. | **Falla Pautas 1 y 2 (semántica y anomalías)**: repite en las 100 sucursales que una ciudad pertenece a un estado y a un país (`ciudad -> estado -> pais`, habiendo 51 ciudades y 31 estados) y que el país depende del cliente (`numeroCliente -> pais`). Si se corrige un estado o país hay que actualizar varias filas (*actualización*), no se puede dar de alta una ciudad sin sucursal (*inserción*) y al borrar la única sucursal de una ciudad se pierde su ubicación (*borrado*). |
| **`pedido`** | Cada fila es una orden de compra levantada por un cliente en una fecha dada, con su folio, estatus y si sigue activa. | **Cumple las 4 pautas**: habla solo del encabezado del pedido, no repite datos del cliente, no tiene ningún `NULL` y su PK `folio` evita tuplas espurias. |
| **`detalle`** | Cada fila es un renglón dentro de un pedido que indica qué producto se vendió, cuántas unidades y a qué costo y precio se pactó esa venta. | **Cumple las 4 pautas**: `costoProduccion` y `precioAplicado` no son redundantes con `producto` porque guardan el precio y costo *históricos* de esa venta específica (con el descuento del cliente), no el precio actual del catálogo. No tiene nulos. |
| **`envio`** | Cada fila es el despacho físico de un pedido mediante un transportista hacia una sucursal del cliente, con su guía, fecha y costo de flete. | **Falla ligeramente Pauta 2 (redundancia entre tablas)**: repite `numeroCliente` (que ya está en `pedido`) para completar la llave foránea compuesta hacia `sucursal(numeroCliente, claveSucursal)`. Dentro de `envio` no se repite en varias filas porque `folio` es `UNIQUE`. Cumple Pautas 1, 3 y 4. |

----


# Ejercicio 1.2. Las dependencias funcionales

### 1. Conjunto F obtenido de las reglas del negocio 
* **`cliente`**
  * $df_1$: `numeroCliente -> razonSocial, correoCliente, telefonoCliente, tipoCliente, RFC, regimenFiscal, estadoOrigen, idFiscalGlobal, paisReceptor` *(Cada número de cliente identifica a una única empresa cliente y sus datos fiscales/contacto).*
  * $df_2$: `correoCliente -> numeroCliente, razonSocial, telefonoCliente, tipoCliente` *(No se permite que dos clientes compartan el mismo correo de contacto principal).*
* **`transportista`**
  * $df_3$: `claveTransportista -> nombreEmpresa, telefonoAtencion, email, esNacional, esInternacional, RFC, codigoAduanal, agenciaAduanal` *(La clave del transportista identifica de forma única a la empresa y sus credenciales).*
* **`producto`**
  * $df_4$: `claveProducto -> modeloProducto, costoUnitarioUSD, precioUnitarioUSD, tipoProducto, provMaquinaria, provTransformador, provTransporte` *(Cada código de producto identifica una presentación única con su modelo, precios y proveedor).*
  * $df_5$: `modeloProducto -> tipoProducto` *(Cada modelo comercial pertenece a una sola línea: `OMNIGOLD`, `HYVOLT` o `HYPRENE`).*
  * $df_6$: `claveProducto, modeloProducto -> precioUnitarioUSD, tipoProducto` *(La clave de presentación junto con su modelo determinan su precio de lista y línea).*
* **`sucursal`**
  * $df_7$: `numeroCliente, claveSucursal -> nombrePlanta, telefonoEncargado, calle, numero, ciudad, estado, pais, CP` *(La clave de sucursal solo es única dentro de cada cliente; juntas identifican la planta y su dirección).*
  * $df_8$: `numeroCliente -> pais` *(Un cliente nacional solo tiene plantas en México y uno de exportación en su único país receptor).*
  * $df_9$: `ciudad -> estado, pais` *(En el padrón logístico, cada ciudad registrada pertenece a un estado y país).*
  * $df_{10}$: `estado -> pais` *(Cada estado o provincia pertenece a un único país).*
* **`pedido`**
  * $df_{11}$: `folio -> numeroCliente, estatus, dia, mes, anio, activo` *(El folio identifica cada orden de compra, quién la pidió, su fecha y estatus).*
* **`detalle`**
  * $df_{12}$: `folio, claveParcial -> claveProducto, costoProduccion, precioAplicado, cantidad` *(El folio del pedido más el número de renglón identifican la partida vendida y sus montos pactados).*
* **`envio`**
  * $df_{13}$: `numeroGuia -> folio, claveTransportista, numeroCliente, claveSucursal, costoFlete, dia, mes, anio` *(Cada número de guía identifica un envío y todos sus datos).*
  * $df_{14}$: `folio -> numeroGuia, claveTransportista, numeroCliente, claveSucursal, costoFlete, dia, mes, anio` *(Cada pedido tiene como máximo un envío, por lo que el folio también determina todos los datos del envío).*
  * $df_{15}$: `folio, numeroCliente -> claveSucursal, costoFlete` *(El folio y cliente de un envío determinan a qué sucursal llegó y cuánto costó el flete).*

### 2. Cierre de atributos para claves candidatas

### 3. Reducir F a un conjunto mínimo

### 4. Intercambiar conjunto F con otro equipo
---

# Ejercicio 1.3. Las formas normales

### 1. Forma normal actual de cada tabla

Todas las tablas están al menos en **1FN** porque no hay celdas con listas ni atributos compuestos sin separar.

* **`cliente` — BCNF:** Sus claves candidatas son `{numeroCliente}` y `{correoCliente}` (1 solo atributo, cumple 2FN). Los únicos determinantes en $F_{\text{cliente}}$ son esos dos mismos atributos (ambos superclaves), por lo que cumple 3FN y BCNF.
* **`transportista` — BCNF:** Su única clave candidata es `{claveTransportista}` y es el único determinante de la tabla.
* **`pedido` — BCNF:** Su única clave candidata es `{folio}` y determina a todas las demás columnas.
* **`detalle` — BCNF:** Su clave candidata es `{folio, claveParcial}`. Ninguna columna depende solo de `folio` ni solo de `claveParcial` (cumple 2FN), ni de `claveProducto` (porque el costo y precio guardados aquí son los montos históricos de esa partida, no los del catálogo). El único determinante es la clave completa.
* **`envio` — BCNF:** Sus dos determinantes (`numeroGuia` y `folio`) son claves candidatas de la tabla, así que todo determinante es superclave.
* **`producto` — 2FN (No cumple 3FN):** Su clave candidata `{claveProducto}` es de una sola columna, por lo que está en **2FN** (no puede tener dependencias parciales). Falla **3FN** por la dependencia transitiva `claveProducto -> modeloProducto -> tipoProducto`, donde `modeloProducto` no es superclave y `tipoProducto` no es atributo primo.
* **`sucursal` — 1FN (No cumple 2FN ni 3FN):** Su clave candidata es compuesta `{numeroCliente, claveSucursal}`. Falla **2FN** por la dependencia parcial `numeroCliente -> pais` (`pais` depende solo de una parte de la clave). Además, falla **3FN** por las dependencias transitivas `ciudad -> estado` y `estado -> pais`.

---

### 2. Paso a 3FN y comprobación de BCNF

#### A) Tabla `producto`
* **Paso único (eliminar `modeloProducto -> tipoProducto`):**
  * Creamos **`modelo_producto(modeloProducto [PK], tipoProducto)`**.
  * Quitamos `tipoProducto` de `producto`, quedando **`producto(claveProducto [PK], modeloProducto [FK], costoUnitarioUSD, precioUnitarioUSD, provMaquinaria, provTransformador, provTransporte)`**.
* **Comprobación de BCNF:** En `modelo_producto` el único determinante es `modeloProducto` (su PK) y en `producto` el único determinante es `claveProducto` (su PK). Ambas quedan en **3FN y BCNF**.

#### B) Tabla `sucursal`
* **Paso 1 (llevar a 2FN y eliminar `estado -> pais` y `numeroCliente -> pais`):**
  * Creamos **`estado_ubicacion(estado [PK], pais)`** y quitamos `pais` de `sucursal` (quedando temporalmente `sucursal_sin_pais` con `estado` como FK). Al salir `pais` de `sucursal`, desaparece la dependencia parcial `numeroCliente -> pais` y la tabla sube a **2FN**.
* **Paso 2 (llevar de 2FN a 3FN eliminando `ciudad -> estado`):**
  * Creamos **`ciudad_ubicacion(ciudad [PK], estado [FK])`** y quitamos `estado` de `sucursal`, dejando `ciudad` como FK:
  * **`sucursal(numeroCliente [PK, FK], claveSucursal [PK], nombrePlanta, telefonoEncargado, calle, numero, ciudad [FK], CP)`**.
* **Comprobación de BCNF:** En `estado_ubicacion` el único determinante es `estado` (PK); en `ciudad_ubicacion` es `ciudad` (PK); y en `sucursal` es `(numeroCliente, claveSucursal)` (PK). Las tres quedan en **3FN y BCNF**.



> **Nota sobre cuándo 3FN no llega a BCNF (qué dependencia se perdería):** Nuestras tablas llegaron a BCNF sin perder dependencias. Si en una tabla de direcciones tuviéramos las reglas `(ciudad, calle) -> CP` y `CP -> ciudad`, estaría en **3FN** (porque `ciudad` es parte de la clave candidata `{ciudad, calle}`) pero no en **BCNF** (porque `CP` no es superclave). Si la partiéramos a BCNF en `(CP, ciudad)` y `(CP, calle)`, **se perdería la dependencia `(ciudad, calle) -> CP`**. En ese caso nos quedaríamos en 3FN para no perder esa regla.

---

### 3. Comprobación de reunión sin pérdida y preservación de dependencias

Aplicamos la **prueba de los atributos comunes** ($R_1 \cap R_2$ debe ser superclave de $R_1$ o de $R_2$):

1. **Descomposición de `producto` en $R_1(\text{modelo\_producto})$ y $R_2(\text{producto})$:**
   * Atributo común: $R_1 \cap R_2 = \{modeloProducto\}$.
   * Cierre: $\{modeloProducto\}^+ = \{modeloProducto, tipoProducto\} = R_1$. Como determina toda $R_1$, **tiene reunión sin pérdida**.
   * **Preserva dependencias:** `modeloProducto -> tipoProducto` vive en $R_1$ y las dependencias de `claveProducto` viven en $R_2$.

2. **Descomposición de `sucursal` en $S_1(\text{estado\_ubicacion})$, $S_2(\text{ciudad\_ubicacion})$ y $S_3(\text{sucursal})$:**
   * Paso 1 ($S_1$ y $S_{\text{temp}}$): $S_1 \cap S_{\text{temp}} = \{estado\}$, y $\{estado\}^+ = \{estado, pais\} = S_1 \implies$ **reunión sin pérdida**.
   * Paso 2 ($S_2$ y $S_3$): $S_2 \cap S_3 = \{ciudad\}$, y $\{ciudad\}^+ = \{ciudad, estado\} = S_2 \implies$ **reunión sin pérdida**.
   * **Preserva dependencias:** `estado -> pais` queda en $S_1$, `ciudad -> estado` queda en $S_2$ y `(numeroCliente, claveSucursal) -> ...` queda en $S_3$ (mientras que el país del cliente ya está guardado en `cliente` y se recupera por transitividad $ciudad \to estado \to pais$).
