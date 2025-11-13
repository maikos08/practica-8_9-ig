# Visualización 3D de Datos SityCleta con Shaders

## Autores y asignatura
- **Autor**: Miguel Ángel Rodríguez Ruano
- **Universidad**: ULPGC - GII (Grado en Ingeniería Informática)
- **Asignatura**: Informática Gráfica


## Enlace al código en codesandbox.io
[Clica aquí](TU_ENLACE_AQUI)


## Video de la demo
Aquí se muestra un video demostrativo de la demo (haz clic sobre el para ver el video)
[![Ver demo](https://img.youtube.com/vi/Tes de la demo
- **Vista 3D**: Usa el mouse para orbitar, zoom y panear con los controles OrbitControls.
- **Control de velocidad**:
  - Flecha Arriba (↑): Aumentar velocidad de simulación (máximo 5x)
  - Flecha Abajo (↓): Reducir velocidad de simulación (mínimo 0.1x)
  - Barra Espaciadora: Pausar/Reanudar simulación
- **Visualización**:
  - El tiempo avanza automáticamente desde el 1 de mayo de 2018
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
```javascript
const t = 1 - (vmax - val) / (vmax - vmin);
return dmin + t * (dmax - dmin);
