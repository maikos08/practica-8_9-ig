# Visualización 3D de Datos SityCleta con Shaders

## Autores y asignatura
- **Autor**: Miguel Ángel Rodríguez Ruano
- **Universidad**: ULPGC - GII (Grado en Ingeniería Informática)
- **Asignatura**: Informática Gráfica


## Enlace al código en codesandbox.io
[Clica aquí](https://codesandbox.io/p/sandbox/ig2526-s8-forked-qz2hzl)


## Video de la demo
Aquí se muestra un video demostrativo de la demo (haz clic sobre el para ver el video)
[![Ver demo](https://img.youtube.com/vi/HjtTUbw-B9E/0.jpg)](https://youtu.be/HjtTUbw-B9E)

Peuqeño GIF

- **Vista 3D**: Usa el mouse para orbitar, zoom y panear con los controles OrbitControls.

- **Control de velocidad**:
  - Flecha Arriba (↑): Aumentar velocidad de simulación (máximo 5x)
  - Flecha Abajo (↓): Reducir velocidad de simulación (mínimo 0.1x)
  - Barra Espaciadora: Pausar/Reanudar simulación
  
- **Visualización**:
  - El ciclo día/noche cambia el cielo de forma dinámica (madrugada, amanecer, mediodía, atardecer, noche)
  - Sol visible entre 6:00 AM - 6:00 PM
  - Luna visible entre 6:00 PM - 6:00 AM
  - Estrellas aparecen durante la noche


## Explicación del código
Este proyecto es una visualización 3D interactiva de datos del sistema de bicicletas compartidas SityCleta de Las Palmas de Gran Canaria, desarrollado con Three.js y GLSL shaders personalizados. La aplicación representa geográficamente las estaciones de bicicletas, visualiza alquileres en tiempo real mediante bicis 3D que vuelan entre estaciones, aplica un mapa de calor temporal para mostrar zonas de mayor actividad, y simula un ciclo completo día/noche con sol, luna y skybox dinámico. El código está escrito en JavaScript ES6 modular y hace uso intensivo de shaders GLSL para efectos visuales avanzados.

### 1. Importaciones y Configuración Inicial
El código importa las dependencias necesarias:
- `* as THREE from "three"`: Biblioteca principal de Three.js para renderizado 3D.
- `{ OrbitControls }`: Controles de cámara para navegación intuitiva.
- `{ GLTFLoader }`: Cargador de modelos 3D en formato GLB (usado para el modelo de bicicleta).

Se definen elementos del DOM:
- `reloj`: Div superior que muestra la fecha/hora actual de la simulación.
- `stats`: Panel inferior izquierdo con estadísticas en tiempo real (bicis volando, estaciones activas, alquileres del día, velocidad).
- `instrucciones`: Panel superior derecho con controles del teclado.

Variables globales clave incluyen:
- `scene`, `renderer`, `camera`, `camcontrols`: Componentes básicos de Three.js.
- `mapa`, `mapsx`, `mapsy`: Plano del mapa de Las Palmas y sus dimensiones.
- `minlon`, `maxlon`, `minlat`, `maxlat`: Coordenadas geográficas del área (lon: -15.47 a -15.39, lat: 28.08 a 28.18).
- `datosEstaciones`, `datosSitycleta`: Arrays con datos cargados de los CSV.
- `bicisEnAire`: Array con bicis actualmente en tránsito.
- `ondasPool`, `particulasPool`: Pools de objetos reutilizables para ondas expansivas y partículas.
- `velocidadSimulacion`: Factor multiplicador del tiempo (0 = pausa, 0.5 = normal, hasta 5x).
- `fechaInicio`, `fechaActual`, `totalMinutos`: Control temporal de la simulación.

### 2. Shaders GLSL Personalizados

#### Shader de Estela para Bicis (vertexShaderBici, fragmentShaderBici)
Crea un efecto de estela luminosa en las bicis 3D:
- **Vertex Shader**: Pasa UV coordinates, posición del vértice y normal al fragment shader.
- **Fragment Shader**:
  - `u_colorStart`, `u_colorEnd`: Colores cyan (0x00ffff) y magenta (0xff00ff) para degradado.
  - `u_progress`: Progreso del trayecto (0 a 1) para ajustar intensidad.
  - `u_time`: Tiempo global para animación de pulsación con `sin()`.
  - Efecto Fresnel: `pow(1.0 - abs(dot(viewDir, vNormal)), 2.0)` para brillo en bordes.
  - Mezcla colores según UV.y (vertical), aplica pulsación sinusoidal y varía intensidad según progreso.
  - Resultado: Bicis con brillo pulsante y degradado dinámico.

#### Shader de Heatmap Temporal (vertexShaderMapa, fragmentShaderMapa)
Visualiza densidad de actividad en el mapa mediante un grid 8×8:
- **Uniform `u_densityData[64]`**: Array con conteo de alquileres activos en cada celda del grid.
- **Uniform `u_maxDensity`**: Valor máximo para normalización.
- **Función `heatColor()`**: Mapea densidad normalizada (0-1) a gradiente de color:
  - 0-0.33: Azul oscuro → Verde (sin/baja actividad)
  - 0.33-0.66: Verde → Amarillo (media actividad)
  - 0.66-1: Amarillo → Rojo (alta actividad)
- Calcula índice de celda desde UV coordinates (`gridX = floor(vUv.x * 8.0)`, `gridY = floor(vUv.y * 8.0)`).
- Aplica pulsación con `sin(u_time * 2.0) * density` para zonas activas.
- Mezcla textura del mapa base con color de calor usando `mixFactor = 0.4 + density * 0.4`.

#### Shader de Skybox Dinámico (vertexShaderSky, fragmentShaderSky)
Crea un cielo que evoluciona con la hora del día:
- **Uniform `u_horaDelDia`**: Hora actual (0-24) para determinar colores.
- **Función `getSkyColor(hora)`**: Define paletas de colores para cada período:
  - Madrugada (0-6h): Púrpura muy oscuro → Púrpura oscuro
  - Amanecer (6-8h): Morado rosado → Naranja brillante
  - Mañana (8-12h): Azul cielo → Azul claro
  - Mediodía (12-15h): Azul brillante → Casi blanco azulado
  - Tarde (15-18h): Naranja dorado → Naranja rojizo
  - Atardecer (18-20h): Rosa rojizo → Púrpura profundo
  - Noche (20-24h): Azul muy oscuro → Azul medianoche
- Usa `smoothstep()` para transiciones suaves entre períodos.
- Degradado vertical: `pow(vUv.y, 1.5)` para gradiente más natural.
- Bandas atmosféricas: `sin(vUv.y * PI * 3.0) * 0.03` para estratos sutiles.
- **Estrellas procedurales**: 
  - Genera campo de estrellas con función `random()` sobre grid 200×200.
  - `step(0.9995, random(...))` para estrellas pequeñas y numerosas.
  - Brillo pulsante con `sin(u_time * 2.0 + random(...) * 6.28)`.
  - Visibilidad adaptativa: Solo visibles de noche (20-6h), con fade-in/out en amaneceres/atardeceres.

### 3. Función `init()`
Inicializa el entorno 3D completo:
- Configura `scene`, `renderer` con antialiasing y `camera` (FOV 75).
- Crea `OrbitControls` con damping para navegación suave.
- **Skybox**: Esfera de radio 100 con material shader, lado interior (`BackSide`).
- **Sol y Luna**: Esferas emisivas creadas con `crearSolLuna()`:
  - Sol: Radio 0.3, color 0xffdd00, visible 6-18h.
  - Luna: Radio 0.25, color 0xccccff, visible 18-6h.
  - Posicionados dinámicamente en arco según hora (distancia 8 unidades, altura máx 6).
- **Iluminación**: `AmbientLight` (0xffffff, 1) y `DirectionalLight` (0xffffff, 1) en posición (5,10,7).
- **Arrays de actividad**: Inicializa `actividadMapa` con 64 ceros (grid 8×8).
- **Pools de efectos**: 
  - `crearPoolOndas()`: 30 anillos (`RingGeometry(0.02, 0.04)`) con material transparente, `depthTest: false` para renderizado sobre el mapa.
  - `crearPoolParticulas()`: 200 esferas pequeñas (radio 0.004) con blending aditivo para estelas brillantes.
- **Carga de modelo 3D**: `GLTFLoader` carga `bicicleta.glb`, escala a 0.00008 (ajuste fino para tamaño correcto).
- **Carga de texturas**: `TextureLoader` carga `mapaLPGC.png`, calcula ratio para dimensiones del plano, llama a `PlanoConHeatmap()`.
- **Carga de datos CSV**: `cargarCSV()` mediante `Promise.all` fetch a dos archivos:
  - `Geolocalización estaciones sitycleta.csv`: Coordenadas y nombres de estaciones.
  - `SITYCLETA-2018.csv`: Datos de alquileres con timestamps de inicio/fin.
- **Controles de teclado**: `configurarControles()` añade listeners para flechas y espacio.

### 4. Procesamiento de Datos CSV

#### `procesarCSVEstaciones(c)`
Parsea CSV de estaciones (separador `;`):
- Extrae índices de columnas `nombre`, `latitud`, `altitud` del header.
- Limita a 100 líneas para rendimiento.
- Para cada línea válida:
  - Crea objeto `{ nombre, lat, lon }`.
  - Convierte coordenadas geográficas a posiciones 3D con `Map2Range()`:
    - `x = Map2Range(lon, minlon, maxlon, -mapsx/2, mapsx/2)`
    - `z = Map2Range(lat, minlat, maxlat, mapsy/2, -mapsy/2)` (invertido para orientación correcta)
  - Crea esfera (`Esfera()`) en posición calculada, radio 0.015, color verde (0x00ff88).
  - Añade a `datosEstaciones` y `objetos`.

#### `procesarCSVAlquileres(c)`
Parsea CSV de alquileres:
- Extrae índices de `Start`, `End`, `Rental place`, `Return place`.
- Limita a 10000 líneas para rendimiento.
- Usa `convertirFecha()` para parsear timestamps formato "DD/MM/YYYY HH:MM".
- Crea objetos `{ t_inicio: Date, t_fin: Date, p_inicio: string, p_fin: string }`.
- Añade a `datosSitycleta` si todos los campos son válidos.

#### `convertirFecha(str)`
Convierte string "DD/MM/YYYY HH:MM" a objeto Date:
- Valida formato con split y comprobaciones `isNaN`.
- Retorna `new Date(año, mes-1, día, hora, minuto)` o `null` si inválido.

### 5. Función `Map2Range(val, vmin, vmax, dmin, dmax)`
Mapea valor de rango [vmin, vmax] a [dmin, dmax] con interpolación lineal:
```
const t = 1 - (vmax - val) / (vmax - vmin);
return dmin + t * (dmax - dmin);
```
Usada para transformar coordenadas geográficas (lon/lat) a posiciones en el plano 3D.

### 6. Sistema de Lanzamiento de Bicis

#### `lanzarBici(o, d, tiempoInicio, tiempoFin)`
Crea una bici que vuela de origen `o` a destino `d`:
- Lanza onda expansiva cyan en origen (`lanzarOnda(o, 0x00ffff)`).
- Si `modeloBiciOriginal` existe:
  - Clona el modelo 3D con `clone()`.
  - Crea `ShaderMaterial` con shader de estela.
  - Itera sobre hijos del modelo (`traverse`) y aplica material shader clonado a cada mesh.
  - Posiciona en origen, añade a escena.
  - Añade a `bicisEnAire` con datos: `{ mesh, start, end, t: 0, tiempoInicio, tiempoFin, esBici3D: true }`.
- Fallback: Si no hay modelo, llama a `lanzarBiciCilindro()` que usa geometría cilíndrica simple.

#### Trayectoria y Timing
En `animate()`, para cada bici en `bicisEnAire`:
- Calcula progreso real desde el CSV:
  ```
  const duracionTotal = v.tiempoFin.getTime() - v.tiempoInicio.getTime();
  const tiempoTranscurrido = fechaActual.getTime() - v.tiempoInicio.getTime();
  v.t = Math.max(0, Math.min(1, tiempoTranscurrido / duracionTotal));
  ```
- Si `v.t >= 1`: Lanza onda magenta en destino (`lanzarOnda(v.end, 0xff00ff)`), elimina mesh, retorna `false`.
- Interpola posición con `lerp()`: `p = start.clone().lerp(end, t)`.
- Trayectoria parabólica: `p.y = 0.05 + 0.35 * Math.sin(t * Math.PI)` (altura máxima 0.4 a mitad del trayecto).
- Orienta bici hacia destino con `lookAt(end)`.
- Genera partículas trail si no está pausado (`velocidadSimulacion > 0`):
  - 40% probabilidad cada frame.
  - Copia posición de bici, desplaza ligeramente hacia abajo (-0.02 en Y).
  - Color según tipo (cyan para 3D, magenta para cilindro).

### 7. Sistema de Efectos Visuales

#### Ondas Expansivas
Pool de 30 anillos reutilizables:
- Geometría: `RingGeometry(0.02, 0.04, 32)` para anillo fino.
- Material: Transparente, `side: DoubleSide`, `depthTest: false` para renderizado correcto.
- Animación en `animate()`:
  - `vida` decrementa 0.02 por frame.
  - Escala crece: `escala = 1 + (1 - vida) * velocidad * 4`.
  - Opacidad fade-out: `opacity = vida * 0.95`.
  - Color según evento (cyan al despegar, magenta al aterrizar).

#### Partículas Trail
Pool de 200 esferas pequeñas:
- Geometría: `SphereGeometry(0.004, 6, 6)` de baja resolución.
- Material: Transparente, `AdditiveBlending` para efecto luminoso acumulativo.
- Animación:
  - `vida` decrementa 0.04 por frame (desaparecen más rápido que ondas).
  - Opacidad: `vida * 0.7`.
  - Escala crece: `vida * 2`.
  - Caída lenta: `position.y -= 0.01` por frame.

#### Sol y Luna
Actualizados cada frame según hora:
- Ángulo calculado desde hora: `angulo = ((hora - 6) / 12) * PI` para sol (6 AM a 6 PM = 0° a 180°).
- Posición en arco:
  ```javascript
  sol.position.set(
    Math.cos(angulo) * 8,  // X: movimiento horizontal
    Math.sin(angulo) * 6 + 3,  // Y: arco vertical con offset +3
    -3  // Z: profundidad fija
  );
  ```
- Visibilidad: `sol.visible = hora >= 6 && hora <= 18`.
- Luna usa ángulo desplazado `+6` horas (visible de noche).

### 8. Función `actualizarDensidadMapa()`
Calcula densidad de actividad en tiempo real:
- Resetea `actividadMapa` a ceros.
- Itera sobre `datosSitycleta` (límite 1000 para rendimiento):
  - Si alquiler está activo (`t_inicio <= fechaActual && t_fin >= fechaActual`):
    - Busca estaciones origen y destino en `datosEstaciones`.
    - Para cada estación:
      - Normaliza coordenadas: `normX = (lon - minlon) / (maxlon - minlon)`.
      - Calcula índice de celda: `gridX = floor(normX * 8)`, `gridY = floor(normY * 8)`.
      - Incrementa `actividadMapa[gridY * 8 + gridX]`.
- Calcula `maxDens = max(...actividadMapa)` y actualiza uniform del shader.

### 9. Función `animate()` - Bucle Principal
Ejecuta cada frame con `requestAnimationFrame`:
- Actualiza tiempo: `totalMinutos += velocidadSimulacion`.
- Calcula `fechaActual = new Date(fechaInicio + totalMinutos * 60000)`.
- Actualiza reloj DOM con `toLocaleString("es-ES", {...})`.
- **Skybox**: Actualiza uniforms `u_time` y `u_horaDelDia`.
- **Sol/Luna**: Calcula ángulos, actualiza posiciones y visibilidad.
- **Heatmap**: Llama a `actualizarDensidadMapa()`, actualiza uniforms del shader del mapa.
- **Lanzamiento de bicis**: 
  - Busca alquileres que inician en el minuto actual.
  - Limita a 50 bicis por minuto para rendimiento.
  - Convierte coordenadas de estaciones origen/destino a vectores 3D.
  - Llama a `lanzarBici(A, B, t_inicio, t_fin)`.
- **Actualización de bicis**: Filtra `bicisEnAire`, actualiza posiciones, genera partículas, actualiza uniforms de shaders.
- **Ondas y partículas**: Actualiza vida, escalas, opacidades; elimina invisibles.
- **Estaciones**: 
  - Crea set `act` con nombres de estaciones activas.
  - Para cada esfera de estación:
    - Si activa: Escala 2.5, color rosa (0xff0055), pulsación con `sin()`.
    - Si inactiva: Escala 1, color verde (0x00ff88).
- **Panel de stats**: 
  - Cuenta `bicisEnAire.length`.
  - Cuenta `act.size` (estaciones únicas activas).
  - Filtra alquileres del día actual.
  - Actualiza `stats.innerHTML` con HTML formateado.
- **Renderizado**: Actualiza `controls.update()`, renderiza escena con `renderer.render(scene, camera)`.

### 10. Consideraciones Técnicas

#### Rendimiento
- **Pools de objetos**: Reutilización de ondas y partículas evita creación/destrucción constante (garbage collection).
- **Límites**: 100 estaciones, 10000 alquileres, 50 bicis/minuto, 1000 búsquedas en heatmap.
- **Geometrías low-poly**: Esferas con segmentos reducidos (16×16), partículas con 6 segmentos.
- **Shader optimization**: Cálculos en GPU, uniforms en lugar de atributos cuando posible.

#### Precisión Temporal
- Sistema basado en timestamps reales del CSV.
- `tiempoTranscurrido / duracionTotal` garantiza sincronización exacta.
- Pausa congela `fechaActual`, bicis quedan suspendidas en posición actual.

#### Shaders Avanzados
- **Blending modes**: `AdditiveBlending` para partículas (efecto luminoso acumulativo), `NormalBlending` para ondas.
- **Depth control**: `depthTest: false` en ondas para renderizado correcto sobre el mapa.
- **Uniforms dinámicos**: Actualizados cada frame para animaciones fluidas.
- **Varying variables**: Pasan datos de vertex a fragment shader (UV, posición, normal).

#### Extensibilidad
- Fácil añadir más efectos: Nuevos pools, shaders adicionales.
- Modular: Funciones independientes para cada sistema.
- Datos externos: CSV permite cambiar dataset sin modificar código.

### Consideraciones Generales
- **Ejecución**: Requiere servidor local por CORS en carga de archivos (CSV, texturas, modelos GLB). Usar `npx serve` o similar.
- **Compatibilidad**: Shaders GLSL compatibles con WebGL 1.0, funciona en navegadores modernos.
- **Educativo**: Ideal para aprendizaje de shaders, sistemas de partículas, visualización de datos geoespaciales y temporales.
- **Realismo visual**: Ciclo día/noche, iluminación dinámica, físicas de trayectorias parabólicas.

Este proyecto combina renderizado 3D avanzado, programación de shaders GLSL, procesamiento de datos CSV y diseño de interfaces interactivas, demostrando técnicas fundamentales de informática gráfica aplicadas a visualización de datos reales.
