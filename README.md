# PoleShift — Simulador de desplazamiento del eje terrestre

> «Si movieras el Polo Norte…»

Una PWA de un solo archivo que responde a una pregunta contrafactual: **¿qué pasaría con el planeta si el Polo Norte estuviera en otro sitio?**

Eliges un punto (tocando el mapa o con los selectores) y la app recalcula la geografía desde cero: dónde quedaría el ecuador, a qué distancia estaría cada ciudad de los polos, y cómo se vería el mundo reordenado sobre el nuevo eje. La app no simula física ni equilibrio del eje — es un ejercicio geométrico, no un modelo de dinámica.

## Qué hace

- **Mapa 1 (plano, arriba):** el mundo actual, sin transformar. Tocar un punto lo convierte en el nuevo Polo Norte.
- **Globo 3D (abajo):** la Tierra ya reordenada respecto al eje elegido, con el ecuador, los polos y las costas recalculados. Gira solo; se puede arrastrar para rotarlo a mano.
- **Tabla de ciudades:** para 14 ciudades, muestra la latitud actual y la latitud en el mundo reordenado, ordenadas de norte a sur. Marca en color cuál queda más cerca del nuevo Polo Norte, del nuevo Polo Sur y del nuevo ecuador.
- **Panel lateral:** 51 lugares predefinidos para fijar el nuevo Polo Norte, el nuevo Polo Sur, o un punto que deba quedar sobre el nuevo ecuador. Los tres son el mismo cálculo en direcciones distintas: al elegir uno, los otros dos se resincronizan solos.

## Cómo funciona

No hay ninguna librería de mapas. Todo son coordenadas cartesianas 3D sobre la esfera y trigonometría.

### 1. El eje es un vector

Un punto geográfico se convierte a vector unitario (`toXYZ`, index.html:172):

```
x = cos(lat)·cos(lon)   y = cos(lat)·sin(lon)   z = sin(lat)
```

El "nuevo eje" es simplemente el vector unitario `P` que apunta al lugar elegido como Polo Norte. El Polo Sur es `-P`; el ecuador es el plano perpendicular a `P`.

### 2. Reconstruir el mapa (`frameV` / `reproject`, index.html:188-201)

Con el eje `P` fijado se arma una base ortonormal del plano ecuatorial nuevo: `u` y `v` son dos vectores perpendiculares a `P` (Gram-Schmidt contra el eje Z original, con respaldo `[1,0,0]` si el caso es degenerado, p. ej. elegir un polo).

Cualquier punto del planeta se proyecta sobre esa base y se lee de vuelta como latitud/longitud nuevas:

```
lat' = 90 − acos(P · X)              (ángulo al nuevo eje)
lon' = atan2(X · v, X · u)           (ángulo dentro del plano ecuatorial)
```

`dot(X, P)` es directamente la **latitud geocéntrica** en el mundo nuevo. Cuando `P = [0,0,1]` el cálculo es la identidad y el mapa sale igual que el real.

El mapa plano es una proyección equirectangular simple (`proj`, index.html:202): `x = (lon+180)/360·W`, `y = (90−lat)/180·H`. Como la reproyección puede "dar la vuelta" al meridiano, cada trazo de polígono y de línea de grilla se corta (`moveTo`) cuando el salto horizontal entre dos puntos consecutivos supera el 60 % del ancho — si no, un polígono cruzaría el mapa entero de un tirón.

### 3. El globo 3D (`ringToNewXYZ` / `drawGlobe`, index.html:299-376)

Para el globo no se reproyecta a 2D: los puntos se **giran directamente al espacio de la nueva base** (`[X·u, X·v, X·P]`) y se aplica una **proyección ortográfica** con el ángulo de giro `theta`:

```
sx = CX + R·(x·sinθ + y·cosθ)
sy = CY − R·z
z' = x·cosθ − y·sinθ            (profundidad → visible u oculto)
```

`z' < 0` es la mitad trasera del planeta: ahí los trazos se interrumpen y los polígonos no se rellenan, que es lo que produce el efecto de esfera. El hemisferio visible está recortado con `clip()` a un círculo de radio `R`. Los anillos de costa se cachean por frame (`getTransformedRings`) para no recalcular 286 polígonos en cada frame de la animación.

`theta` se incrementa solo a 0.006 rad/frame mientras no se arrastre; al hacer `pointerdown` la rotación automática se detiene y el `pointermove` mueve el globo en proporción al desplazamiento horizontal.

### 4. Los tres selectores son el mismo cálculo invertido

Todo pasa por `applyPole(P)` (index.html:469): pinta el mapa, dibuja el punto elegido, recalcula el globo y la tabla, y resincroniza los selectores. Los tres `change` solo calculan `P` distinto:

- **Polo Norte** = `P` del lugar elegido.
- **Polo Sur** = `-P` del lugar elegido (el nuevo norte es su antípoda).
- **Ecuador** = el componente de `P` perpendicular al lugar elegido: `P = normalize(N - C·(N·C))`. Es decir, se proyecta el eje actual sobre el plano perpendicular a `C`, de modo que `C` queda exactamente sobre el ecuador nuevo.

`nearestIndex` / `nearestEquatorIndex` hacen el camino de vuelta: eligen, de los 51 lugares, el más cercano al nuevo polo (mayor `dot`) o al ecuador (menor `|dot|`).

### 5. Los datos de la Tierra

- **Costas:** 286 polígonos anidados como texto plano en `RINGS_RAW` (index.html:121) — un contorno de costas mundial muy simplificado, parseado al arrancar con `split(';')` + `split(',')` en pares `[lon, lat]`.
- **Ciudades (tabla):** 14, hardcodeadas en `cities` (index.html:129).
- **Lugares (selectores):** 51, en `places` (index.html:137) con el formato `[país, ciudad, lat, lon]`.

Los tres arrays están al principio del `<script>`: editar la geografía es editar esas listas, no tocar la lógica.

## Archivos

| Archivo | Qué es |
|---|---|
| `index.html` | Toda la app: HTML, CSS, lógica, datos y los tres canvas de dibujo. 513 líneas, cero dependencias. |
| `manifest.json` | Manifiesto PWA: nombre, icono, pantalla fija en retrato, colores. |
| `sw.js` | Service worker: cachea los 5 assets para uso sin conexión. |
| `icon-{180,192,512}.png` | Iconos de la app. |

## Ejecutar

No hay build, ni `package.json`, ni bundler. Doble clic en `index.html` y funciona.

Para la instalación como PWA (y por tanto para el service worker, que los navegadores exigen sobre `https://` o `localhost`), hay que servirlo por HTTP:

```powershell
python -m http.server 8000
# → http://localhost:8000
```

Probado en Chrome/Edge. En móvil: «Añadir a pantalla de inicio» y arranca a pantalla completa, con soporte de `safe-area-inset` para no quedar bajo el notch.

## Offline y actualización

`sw.js` usa dos estrategias:

- **`network-first` para el HTML:** siempre intenta la red para que un cambio publicado se vea al recargar; si no hay red, sirve la copia cacheada.
- **`cache-first` para el resto** (íconos, manifiesto): de la caché, y en paralelo refresca la copia.

La versión de la cache es el string `CACHE = 'eje-terrestre-v1'` — **al publicar cambios hay que subirlo**, o los navegadores seguirán sirviendo la versión vieja. El registro llama a `reg.update()` cada hora y hace `location.reload()` en el primer `controllerchange` de la sesión, de modo que la recarga no suele ser necesaria a mano.

## Limitaciones conocidas

- **Contornos, no datos geográficos.** Las costas son un polígono simplificado: suficiente para reconocer continentes, insuficiente para medir. No hay relieve, bathimetría, fronteras ni topónimos.
- **Modelo geométrico, no físico.** No hay nada de balance de modos, precesión, deriva continental ni timescales. Es una rotación rígida: las distancias entre ciudades no cambian, solo la latitud que se les atribuye.
- **Proyecciones distintas entre las dos vistas.** El mapa plano y el globo son vistas diferentes del mismo mundo reordenado; las formas no coinciden exactamente entre una y otra.
- **La funciona «Reiniciar»** solo restaura el eje original; no borra nada más porque no hay estado que borrar.
- **Tema claro a medias:** el CSS define la paleta clara, pero el dibujo de los canvas usa colores oscuros fijos, así que los mapas se ven igual de oscuros.
