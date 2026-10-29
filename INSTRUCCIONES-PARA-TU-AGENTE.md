# Instrucciones para tu agente: construir un sistema operativo agéntico

**Versión:** 1.3 · 2026-10-27 (proyectos conectados: la torre de control; aislamiento de los trabajadores; skills de proyecto y de usuario)
**Del canal de YouTube de Arturo Magana-Mora**, serie «Sistema operativo agéntico» (episodio 1: el cerebro digital; episodio 2: este sistema).
**Funciona con:** Claude Code o Codex · Windows, macOS o Linux · con o sin tarjeta gráfica.

> **Para la persona:** no tienes que leer este documento entero. Descarga la carpeta completa, abre ahí Claude Code o Codex y pega la instrucción del `README.md`. El agente te va a preguntar todo lo que necesita.
>
> **Para el agente:** este documento es tu especificación. Léelo completo antes de hacer nada.

---

## 0. Vocabulario

- **Bóveda:** la carpeta completa del cerebro digital, la que se abre en Obsidian (por ejemplo, la del episodio 1). Incluye todas sus subcarpetas: `raw`, `wiki`, `outputs`, `notas` y, cuando exista, `_os`. No es solo `raw`.
- **`raw`:** subcarpeta de la bóveda con las fuentes originales. La IA la lee pero nunca la modifica.
- **`_os`:** subcarpeta de la bóveda donde el sistema escribe informes, recibos y su estado. El nombre se pregunta en la sección 3.2.
- **Carpeta del código:** donde vive el programa del sistema. Siempre fuera de la bóveda (sección 4).
- **Proyecto conectado (workspace):** la carpeta de otro proyecto de la persona (un canal, un repositorio de código, otro agente) que el sistema puede ver o lanzar. Cada uno se conecta en uno o varios modos: observar, ejecutar o delegar (sección 4.1). El sistema conoce esos proyectos, no los contiene.
- **Valores de ejemplo:** las rutas, puertos, modelos, voces y teclas que aparecen en este documento son ilustrativos. Cada valor real sale de la detección (3.1) o de una respuesta de la persona (3.2).

---

## 1. Tu papel y tus reglas

Vas a construir, junto con la persona, un sistema operativo agéntico en su computadora, por fases y con su aprobación en cada fase. Estas reglas no se negocian:

1. **No asumas nada.** Ni el sistema operativo, ni el hardware, ni las rutas, ni las cuentas, ni el idioma, ni lo que la persona quiere. Detecta lo que se pueda detectar con comandos de solo lectura (sección 3.1), pregunta el resto (sección 3.2) y confirma lo detectado antes de usarlo. Los valores de ejemplo de este documento nunca se usan tal cual.
2. **Pregunta antes de decidir.** Si una respuesta cambia lo que construyes, es una pregunta bloqueante: no sigas con lo que depende de ella hasta que la persona conteste. Propón siempre un valor por defecto razonado, pero deja que la persona decida.
3. **Shell y rutas nativas.** En Windows usa PowerShell y rutas como `D:\carpeta`. En macOS y Linux usa el shell de la persona (zsh o bash) y rutas como `/Users/nombre/carpeta` o `/home/nombre/carpeta`. Cuando le enseñes una ruta a la persona, que sea absoluta.
4. **Secretos.** Nunca pidas que te pegue una clave en el chat. Nunca abras, imprimas ni copies archivos `.env`, claves de API, tokens, contraseñas o archivos de credenciales. La persona guarda cada clave en el almacén de contraseñas de su sistema (sección 6.6) y el código la lee en tiempo de ejecución.
5. **La bóveda es de la persona.** Antes de tocarla, lee sus instrucciones (`AGENTS.md` o `CLAUDE.md`). Cualquier cambio en la bóveda se propone como diff y se aplica solo con aprobación explícita. Nunca escribas en la carpeta de fuentes de solo lectura (en el episodio 1 se llama `raw`).
6. **Nada de instalaciones globales sin permiso.** Prefiere entornos por proyecto (por ejemplo, `uv` para Python). Si algo requiere instalar a nivel de sistema (drivers, espeak-ng, ffmpeg, WSL), explícalo y pide permiso.
7. **Verifica antes de afirmar.** Las opciones de las CLIs y los esquemas de las APIs cambian. Antes de usarlos, compruébalos con `--help` y con la documentación oficial actual, y anota lo que comprobaste, con fecha y versión, en `docs/decisiones.md`. Si no puedes verificar algo, dilo; no lo rellenes.
8. **Por fases y con puertas.** Al final de cada fase (sección 5) enseña la evidencia, haz commit y **detente** hasta que la persona diga que sigues.
9. **Habla claro.** La persona puede no ser programadora. Explica cada paso en una o dos frases sencillas, en su idioma, sin jerga innecesaria.
10. **Más de 10 archivos a la vez:** enseña primero la lista y espera aprobación.

---

## 2. Qué vamos a construir

```text
            Persona (voz o clic)
            │                 │
   Plugin de Obsidian    Panel en el navegador        ← dos puertas de entrada, una sola interfaz
            └────────┬────────┘
                     ▼
        PUENTE (servicio local, sin IA)  ◄──►  SERVICIO DE VOZ (oídos: STT · voz: TTS)
                     │  escribe recibos e informes
                     ▼
        ROUTER: botones y reglas → directo
                texto o voz libre → triaje (Jev u otra opción)
            ┌────────┼─────────────────────┐
            ▼        ▼                     ▼
       Nivel 1    Nivel 2               Nivel 3
       reglas y   modelo pequeño        Claude Code o Codex en segundo plano,
       datos      que resume lo         ejecutando una skill; único nivel con
       guardados  que ya existe         conectores (correo, calendario…)
                     │
                     ▼
        BÓVEDA (el cerebro digital): informes, conversaciones, un recibo por petición

        PROYECTOS CONECTADOS (opcional): el puente lee su estado o los lanza;
        sus rutinas siguen corriendo solas, con o sin el sistema
```

Analogía para explicárselo a la persona: un hospital. El nivel 1 es la ventanilla, el nivel 2 una consulta rápida y el nivel 3 el quirófano. El router es el triaje y la bóveda el expediente.

Para los proyectos que la persona ya tiene, otra analogía: una torre de control. Cada proyecto es un avión que vuela solo, con su tripulación (su código, sus instrucciones, sus skills y su programador de tareas). La torre ve dónde está cada avión, le da instrucciones y anota todo, pero no lo pilota (sección 4.1).

Principios de diseño:

- **Los clics no pasan por el triaje.** Un botón ya es una intención conocida; va directo a su skill. Si el triaje falla, los botones y las reglas siguen funcionando.
- **Si el triaje duda, pregunta.** Si la probabilidad más alta queda bajo el umbral calibrado, el sistema pide una aclaración en lugar de escalar al nivel 3 en silencio.
- **Las acciones con efectos externos** (enviar, publicar, borrar o modificar algo fuera de la bóveda) requieren confirmación explícita.
- **El nivel 2 no investiga.** Solo resume informes que ya existen; si falta el informe, ofrece lanzar la skill.
- **Cada tarjeta del panel lee un archivo producido por una skill real** o por un proyecto conectado. Sin ese archivo no hay tarjeta.
- **El sistema conoce los proyectos, no los contiene.** Si se apaga, las rutinas de los proyectos siguen corriendo. No muevas, reprogrames ni modifiques una rutina que ya funciona.

---

## 3. Descubrimiento: detectar y preguntar

Haz esto antes de crear o instalar nada.

### 3.1 Detección automática (solo lectura)

Ejecuta los comandos del sistema que corresponda, muestra a la persona un resumen en una tabla y pídele que lo confirme.

| Dato | Windows (PowerShell) | macOS | Linux |
|---|---|---|---|
| Sistema y versión | `Get-CimInstance Win32_OperatingSystem \| Select Caption,Version` | `sw_vers` | `cat /etc/os-release` |
| Arquitectura | `$env:PROCESSOR_ARCHITECTURE` | `uname -m` | `uname -m` |
| Tarjeta gráfica | `Get-CimInstance Win32_VideoController \| Select Name,AdapterRAM` | `system_profiler SPDisplaysDataType` | `lspci \| grep -iE 'vga\|3d'` |
| NVIDIA (si hay) | `nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv` | — | igual que Windows |
| AMD (si hay) | — | — | `rocm-smi` (si está instalado) |
| Memoria RAM | `Get-CimInstance Win32_ComputerSystem \| Select TotalPhysicalMemory` | `sysctl hw.memsize` | `free -h` |
| Discos y espacio libre | `Get-PSDrive -PSProvider FileSystem` | `df -h` | `df -h` |
| Herramientas | `git --version`, `py -V` o `python --version`, `uv --version`, `node -v`, `claude --version`, `codex --version`, `ffmpeg -version` | igual con `python3` | igual con `python3` |
| Obsidian | preguntar | preguntar | preguntar |

`AdapterRAM` en Windows puede subestimar la memoria de tarjetas grandes; si hay NVIDIA, usa el dato de `nvidia-smi`. Si un comando no existe, anótalo como «no instalado»: no es un error.

### 3.2 Preguntas a la persona

Hazlas en rondas cortas (no más de cinco preguntas por mensaje), con un valor por defecto propuesto y razonado. Valida cada respuesta.

**Ronda 1 — Ubicaciones**

1. ¿Ya tienes un cerebro digital (una bóveda de Obsidian, por ejemplo la del episodio 1)? Si sí, ¿cuál es su ruta? Si no, ¿dónde lo creamos?
   - Valida: la carpeta existe; si tiene `.obsidian`, es una bóveda; lee su `AGENTS.md` o `CLAUDE.md`.
   - Si no existe, propón crear la estructura del episodio 1 (`raw`, `wiki`, `outputs`, `notas` e índices) y explica qué es cada carpeta.
2. ¿En qué disco y carpeta instalamos el código del sistema?
   - Por defecto propón una carpeta hermana de la bóveda en el disco con más espacio libre, por ejemplo `D:\agentic-os` o `~/agentic-os`.
   - Rechaza y explica si queda dentro de la bóveda o si la bóveda queda dentro de ella (sección 4).
   - Desaconseja carpetas sincronizadas en la nube (OneDrive, iCloud Drive, Dropbox, Google Drive) para el código: sincronizan miles de archivos de dependencias. Si la persona insiste, respétalo y anótalo.
3. ¿Dónde guardamos los modelos que se descargan (pueden ocupar entre 1 y 20 GB según las opciones)? Propón el disco con más espacio libre.
4. ¿Cómo se llama la carpeta, dentro de la bóveda, donde el sistema escribe sus informes y recibos? Por defecto `_os`.
5. ¿Tienes otros proyectos que quieras ver o lanzar desde el sistema? Opcional. Explica la torre de control (sección 4.1) y, para cada proyecto, pregunta:
   - su ruta;
   - el modo: observar, ejecutar, delegar o varios;
   - si ya tiene una rutina programada, quién la programa y a qué hora;
   - si delegar puede escribir dentro del proyecto (por defecto, no).

   Registra proyecto por proyecto. Rechaza y explica la carpeta del código del sistema, la bóveda y cualquier carpeta padre que contenga varios proyectos. Si el proyecto maneja dinero, credenciales o datos sensibles, recomienda solo observar.

**Ronda 2 — Agentes y modelos**

6. ¿Qué agentes tienes y con qué cuenta: Claude Code, Codex o los dos? ¿Cuál quieres por defecto para el nivel 3?
7. ¿Qué triaje quieres? Opciones (sección 6.1): Jev a través de OpenRouter (recomendado, necesita cuenta y saldo), un modelo pequeño a través de OpenRouter, un modelo local o solo reglas.
8. ¿Qué modelo para el nivel 2? Un modelo pequeño en OpenRouter, uno local (si el hardware lo permite) o desactivarlo.
9. Si usas OpenRouter, ¿qué límite de gasto mensual quieres? Recomienda crear una clave con límite de crédito.

**Ronda 3 — Voz** (se puede dejar para más tarde)

10. ¿Quieres voz: sí, no o más tarde?
11. ¿En qué idioma y con qué acento? Afecta a las voces disponibles.
12. ¿Reconocimiento de voz local o en la nube? Ofrece solo las opciones que el hardware permite (sección 6.4).
13. ¿Voz de respuesta local o en la nube? ¿Quieres clonar tu propia voz? Solo la de la persona o una con permiso explícito; el audio sintético se marca como tal cuando pueda confundirse con una grabación real.
14. ¿Qué tecla o combinación para hablar? Comprueba que no choque con atajos del sistema.

**Ronda 4 — Puertas de entrada, conectores y reglas**

15. ¿Plugin de Obsidian, panel en el navegador o los dos?
16. ¿Qué conectores quieres más adelante (correo, calendario, documentos, YouTube, otros)? Se configuran en la Fase 7, cada uno con su autorización oficial; nunca se pegan credenciales en el chat.
17. ¿Zona horaria, formato de fecha y a qué horas correrían las automatizaciones?
18. ¿Qué acciones requieren tu confirmación? Por defecto: todo lo que envía, publica, borra o modifica algo fuera de la bóveda.
19. ¿Puertos locales? Por defecto 8765 (puente) y 8766 (voz); comprueba que estén libres.

Al terminar, enséñale el resumen completo como tabla y pide confirmación antes de la Fase 0.

### 3.3 Archivo de configuración

Guarda las respuestas en `config/agentic-os.toml` dentro de la carpeta del código. Nunca guardes secretos aquí, solo los nombres con los que se buscan en el almacén de contraseñas.

El bloque siguiente es un **ejemplo** de cómo queda el archivo para una persona con Windows y una tarjeta NVIDIA. Para cada persona, cada valor será distinto: ninguno se copia de aquí.

```toml
# EJEMPLO ilustrativo. Cada valor sale de la detección (3.1) o de las respuestas (3.2).
[sistema]
os = "windows"            # windows | macos | linux
arch = "x64"              # x64 | arm64
shell = "powershell"      # powershell | zsh | bash
gpu = "nvidia"            # nvidia | apple | amd | intel | ninguna
gpu_vram_gb = 24
idioma = "es-MX"
zona_horaria = "America/Mexico_City"
formato_fecha = "AAAA-MM-DD"

[rutas]                   # absolutas, en el formato del sistema
codigo = 'D:\agentic-os'     # en macOS sería algo como '/Users/ana/agentic-os'
boveda = 'D:\Cerebro'        # la carpeta completa del cerebro, no solo raw
modelos = 'E:\modelos'       # en Linux sería algo como '/home/ana/modelos'
carpeta_os_en_boveda = "_os"
# los proyectos conectados van en config/workspaces.toml (sección 4.1)

[agentes]
disponibles = ["claude", "codex"]
por_defecto = "claude"
concurrencia_nivel3 = 1

[router]
backend = "jev"           # jev | llm-pequeno | local | reglas
modelo = "typesafe/jev-1.14"   # versión fijada; nunca un alias «latest»
umbral = 0.0              # se calibra en la Fase 2
timeout_s = 2

[nivel2]
backend = "openrouter"    # openrouter | local | desactivado
modelo = ""

[voz]
activada = true
stt = "faster-whisper"    # según la tabla de la sección 6.4
stt_modelo = "large-v3-turbo"
tts = "kokoro"
tts_voz = "em_alex"
tecla = "ctrl+alt+space"

[puertas]
obsidian = true
panel_web = true
puerto_puente = 8765
puerto_voz = 8766

[seguridad]
confirmar = ["enviar", "publicar", "borrar", "modificar_fuera_de_boveda"]

[secretos]                # solo nombres; los valores viven en el almacén del sistema
servicio = "agentic-os"
openrouter = "openrouter"
```

---

## 4. Dónde vive cada cosa

Cuando la persona dice «el agente» se mezclan tres cosas. Explícaselo:

| Pieza | Qué es | Dónde vive | Dónde trabaja |
|---|---|---|---|
| Agente constructor | Tú, en las sesiones donde construimos el sistema | Carpeta del código | Carpeta del código |
| Puente | Servicio local sin IA que recibe peticiones, enruta y escribe recibos | Su código está en la carpeta del código | Lee y escribe en `<bóveda>/_os` |
| Trabajadores del nivel 3 | Claude Code o Codex que el puente lanza en segundo plano | No tienen carpeta propia | En la bóveda (o en un workspace registrado) |

**Regla:** la carpeta del código y la bóveda van separadas. Ninguna dentro de la otra, y no dentro de una carpeta padre que el trabajador vaya a usar como directorio de trabajo. Motivos:

- **Instrucciones heredadas.** Claude Code carga los `CLAUDE.md` de las carpetas padre y Codex recorre desde la raíz del repositorio hasta la carpeta actual. Un trabajador dentro del código heredaría tus instrucciones de constructor.
- **Radio de daño.** Un trabajador que ve el código del sistema puede modificarlo.
- **Higiene.** Las dependencias y builds dentro de la bóveda las indexa Obsidian y las arrastra cualquier sincronización.

Estructura de referencia:

```text
<carpeta del código>/
├─ AGENTS.md                  instrucciones del proyecto (plantilla en la sección 7)
├─ CLAUDE.md                  una sola línea: @AGENTS.md
├─ docs/  INSTRUCCIONES-PARA-TU-AGENTE.md (este archivo), decisiones.md, estado.md, evidencia/
├─ config/  agentic-os.toml, skills.toml, workspaces.toml, mcp.toml, claude-worker.json
├─ bridge/  router/  runners/  speech/
├─ hud/                       panel web
├─ obsidian-plugin/           código fuente; el build produce main.js, manifest.json y styles.css
├─ scripts/                   arrancar, desplegar el plugin, sincronizar skills y conectores, sondas
└─ tests/                     pruebas y conjunto de evaluación del triaje

<bóveda>/
├─ AGENTS.md                  reglas de la bóveda (contenido)
├─ CLAUDE.md                  una sola línea: @AGENTS.md
├─ .agents/skills/<id>/SKILL.md    skills canónicas (Codex)
├─ .claude/skills/            copia para Claude Code (la genera un script)
├─ .obsidian/plugins/agentic-os/   solo los 3 archivos compilados del plugin
└─ _os/  recibos/  conversaciones/  informes/<skill>/  estado/  proyectos/
```

Las carpetas que empiezan con punto no aparecen en el explorador ni en el grafo de Obsidian.

### 4.1 Proyectos conectados: la torre de control

Si la persona ya tiene proyectos (un canal, repositorios de código, otros agentes, rutinas que corren cada día), no hace falta meterlos dentro del sistema. Explícaselo con la torre de control:
- cada proyecto es un avión que vuela solo, con su tripulación: su código, su `AGENTS.md`, sus skills y su programador de tareas;
- el sistema es la torre: ve dónde está cada avión, le da instrucciones y anota todo, pero no lo pilota.

**Prueba de diseño:** si se apaga el sistema, las rutinas de los proyectos siguen corriendo. Si no fuera así, el sistema se habría convertido en un punto único de fallo.

| Modo | Qué hace el sistema | ¿IA? | Ejemplo |
|---|---|---|---|
| `observar` | El puente lee un archivo de estado que escribe el proyecto y muestra una tarjeta | No | Una rutina diaria que ya corre con el programador de tareas |
| `ejecutar` | Un botón que corre un comando del propio proyecto, fijado en la configuración | No; los clics no pasan por el triaje | «Volver a correr la rutina» |
| `delegar` | Nivel 3: Claude Code o Codex trabajan con el directorio de trabajo en el proyecto, con su `AGENTS.md` y sus skills | Sí | Pedirle una tarea al agente de ese proyecto |

**Reglas:**
- **Proyecto por proyecto.** Nunca registres la carpeta del código del sistema, la bóveda ni una carpeta padre con varios proyectos: un trabajador ahí vería todos a la vez.
- **Observar solo lee un archivo.** El puente lee el archivo de estado declarado, nunca el resto de la carpeta.
- **Ejecutar no usa IA ni texto libre.** El comando y su carpeta van fijos en la configuración. El puente lo lanza como lista de argumentos, sin pasarle texto libre a un intérprete de órdenes, con tiempo máximo, y guarda el código de salida en el recibo.
- **Delegar usa las reglas del proyecto.** El trabajador carga el `AGENTS.md` o el `CLAUDE.md` del proyecto y sus skills. Por defecto solo lee; escribir dentro del proyecto requiere que la persona lo apruebe para ese proyecto, y los permisos se definen por skill, como en la bóveda.
- **No modifiques los proyectos desde aquí.** Si un proyecto necesita escribir su archivo de estado, propón el cambio mínimo para que la persona lo aplique en ese proyecto, con sus propias reglas y con su agente.
- **Dinero, credenciales o datos sensibles:** recomienda solo observar. El sistema nunca toca cuentas reales.
- **Privacidad:** al recibo de la bóveda van solo el estado (correcto, falló o no corrió) y la hora. Las cifras y los datos del proyecto se quedan en el proyecto, salvo que la persona decida otra cosa: el nivel 2 manda a la nube lo que lee de la bóveda.

**Archivo de estado.** Lo escribe el proyecto al terminar su rutina:

```json
{
  "rutina": "diaria",
  "inicio": "2026-10-27T07:00:00+01:00",
  "fin": "2026-10-27T07:04:12+01:00",
  "ok": true,
  "resumen": "texto corto",
  "informe": "ruta absoluta del informe, si existe"
}
```

Si el archivo no existe, o si su `fin` es anterior a la última ejecución esperada más el margen, la tarjeta muestra «no corrió».

**Configuración** (`config/workspaces.toml`):

```toml
# EJEMPLO ilustrativo. Cada proyecto se registra con la persona, uno por uno.
[[workspace]]
id = "rutina-finanzas"
ruta = 'D:\proyectos\finanzas'            # en macOS sería algo como '/Users/ana/proyectos/finanzas'
modos = ["observar", "ejecutar"]          # observar | ejecutar | delegar
estado = 'estado\ultima-ejecucion.json'   # relativo a la ruta; lo único que lee el puente
frecuencia = "diaria"
hora = "07:00"
margen_min = 60                           # pasado este margen sin estado nuevo: «no corrió»
programado_por = "Programador de tareas de Windows"   # el sistema no lo agenda
comando = ['pwsh', '-NoProfile', '-File', 'scripts\rutina_diaria.ps1']   # solo para «ejecutar»

[[workspace]]
id = "canal"
ruta = 'D:\mi-canal'
modos = ["delegar"]
escritura = false                         # true solo si la persona lo aprueba para este proyecto
```

**Cómo se ve:**
- **En el panel:** una sección «Proyectos», con una tarjeta por proyecto. Cada tarjeta muestra el estado (verde si salió bien, rojo si falló, amarillo si no corrió), la hora de la última ejecución, la próxima, el enlace al informe y los botones que permita su modo.
- **En Obsidian:** el puente escribe `<bóveda>/_os/proyectos/<id>.md`, con el estado en el frontmatter. Con Bases, todas se ven como una tabla «Proyectos».

**Skills de proyecto y de usuario.** Claude Code y Codex cargan skills a dos niveles:

| Nivel | Claude Code | Codex |
|---|---|---|
| Proyecto | `.claude/skills/`, desde la carpeta de trabajo hasta la raíz del repositorio | `.agents/skills/`, desde la carpeta de trabajo hasta la raíz del repositorio |
| Usuario (todos los proyectos) | `~/.claude/skills/` | `~/.agents/skills/` |

En Windows, `~` es `%USERPROFILE%`. Un trabajo delegado carga las skills de su proyecto y las del usuario, nunca las de otros proyectos. Lo que esté a nivel de usuario entra en **todos** los trabajos, también los de la bóveda: las skills y las instrucciones globales (`~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`). Explícaselo a la persona y recomiéndale dejar ahí solo lo genérico.

---

## 5. Fases de construcción

Al cerrar cada fase:

1. Guarda la evidencia en `docs/evidencia/fase-N.md`.
2. Actualiza `docs/estado.md` con lo hecho y el siguiente paso exacto, para que otra sesión, sea de Claude o de Codex, pueda continuar.
3. Haz commit.
4. **Detente** y pregunta si sigues.

### Fase 0 — Preparación

1. Con la configuración confirmada, crea la carpeta del código y `git init`.
2. Copia este documento a `docs/`.
3. Escribe `config/agentic-os.toml`, el `AGENTS.md` del proyecto (sección 7) y un `CLAUDE.md` con la línea `@AGENTS.md`.
4. Si la bóveda no está en git, propón `git init` con un `.gitignore` que excluya `.obsidian/workspace*.json`, para que cualquier cambio sea reversible.
5. Guía a la persona para guardar sus claves en el almacén de contraseñas (sección 6.6). Tú nunca ves el valor.
6. Sondas, sin instalar nada a nivel de sistema todavía:
   - aceleración disponible: CUDA, Metal/MPS, ROCm o ninguna, según el hardware;
   - una transcripción corta con el STT elegido;
   - las versiones y el `--help` de las CLIs de los agentes;
   - una llamada al triaje elegido (sección 6.1);
   - puertos libres.
7. Migración de instrucciones de la bóveda, como diff y con aprobación: si la bóveda tiene `CLAUDE.md`, su contenido pasa a `AGENTS.md`, `CLAUDE.md` queda con `@AGENTS.md`, y se añade la sección de la plantilla 8.
8. Pide a la persona que cierre esta sesión y abra su agente **en la carpeta del código**, para que cargue el `AGENTS.md` del proyecto. La siguiente sesión empieza leyendo `docs/estado.md`.

**Puerta 0:** tabla de sondas; con el directorio de trabajo en la bóveda, cada agente disponible contesta correctamente a «¿Qué carpeta de esta bóveda es de solo lectura?» citando `AGENTS.md`.

### Fase 1 — El puente

- Servicio HTTP local en `127.0.0.1` (nunca `0.0.0.0`) con un token local guardado fuera del repositorio y de la bóveda.
- Endpoints de la sección 9 y escritor de recibos.
- CORS solo para el origen de Obsidian y el del panel.

**Puerta 1:** una petición de prueba produce su recibo; las pruebas pasan; si el puente está apagado, el cliente recibe un error claro, no un cuelgue.

### Fase 2 — El router

- Catálogo de skills en `config/skills.toml`.
- Reglas para botones y frases exactas.
- Cliente del triaje, con plan B si falla: primero reglas y, si ninguna coincide, el nivel 2.
- Conjunto de evaluación: al menos 60 frases en el idioma de la persona, etiquetadas con ella (20 por nivel, con casos límite).
- Calibra el umbral y guarda las probabilidades crudas para poder recalibrar sin repetir llamadas.

**Puerta 2:** matriz de confusión, aciertos por nivel, latencia p50/p95 y umbral justificado.

### Fase 3 — Niveles 2 y 3

- **Nivel 2:** solo lee informes de `<bóveda>/_os/informes`.
- **Nivel 3:** un adaptador por agente con la misma interfaz (sección 9), una cola con la concurrencia configurada, tiempo máximo y cancelación.
- **Permisos por skill**, nunca modos sin límites (sección 6.3).
- **Trabajadores aislados:** el archivo de ajustes del trabajador y el modo sin preguntas pendientes (sección 6.3).
- **Interruptor del agente por defecto**, persistido en `<bóveda>/_os/estado`.
- **Scripts de sincronización** de skills y conectores para los dos agentes.

**Puerta 3:**
- la misma skill ejecutada con cada agente disponible produce su informe y su recibo con el agente correcto;
- una acción con efectos externos queda bloqueada hasta confirmar;
- un trabajador lanzado en la bóveda no consigue leer un archivo de prueba fuera de su carpeta ni un `.env` de prueba, y el recibo lo registra como denegado.

### Fase 4 — Voz (si la persona la quiere)

- Servicio de voz como proceso separado, con micrófono, altavoz y tecla para hablar.
- STT y TTS según la configuración.
- El texto del nivel 2 llega en streaming; se parte en frases y cada frase va al TTS en cuanto termina.
- Normalización propia de fechas, horas, cifras y siglas a texto hablado en el idioma de la persona.
- La tecla interrumpe y vacía la cola de audio.

**Puerta 4:**
- 20 peticiones de voz con la latencia desde que se suelta la tecla hasta el primer audio (p50/p95);
- tasa de error de la transcripción sobre esas frases.

### Fase 5 — Prueba a ciegas de voces (opcional)

Las mismas 20 frases (con números, fechas y nombres propios) en dos o tres opciones de TTS de la sección 6.4. Mide el tiempo hasta el primer audio y el uso de memoria, y recoge votos a ciegas. La persona decide el backend por defecto; los demás quedan configurables.

**Puerta 5:** tabla de resultados y decisión anotada en `docs/decisiones.md`.

### Fase 6 — Puertas de entrada

- **Panel web** servido por el propio puente. Muestra el estado, los botones de las skills, el interruptor de agente, la cola de trabajos y las tarjetas con los informes. Si hay proyectos conectados, también la sección «Proyectos» (sección 4.1).
- **Plugin de Obsidian:** una vista que embebe el panel y usa `postMessage` para lo que solo Obsidian sabe hacer: abrir una nota, abrir la terminal lateral con el agente activo y los comandos de paleta. Sin lógica de negocio en el plugin.
- **Despliegue:** un script compila y copia solo `main.js`, `manifest.json` y `styles.css` a `<bóveda>/.obsidian/plugins/agentic-os/`.

**Puerta 6:**
- la misma petición desde el navegador y desde Obsidian genera el mismo recibo;
- con el puente apagado, el plugin muestra «desconectado» y Obsidian sigue funcionando.

### Fase 7 — Skills, automatizaciones y conectores

1. **Descubre las skills con la persona**, de tres maneras:
   - que te cuente qué hace cada día y cada semana;
   - con su permiso, revisa el historial local de sus sesiones de Claude Code o Codex;
   - combina las dos.
2. **Cada skill empieza manual**, con botón o con voz. Propón convertirla en automatización solo cuando la persona diga que su salida le convence; lo recomendable es después de una semana.
3. **Conectores**, uno a uno, con la autorización oficial de cada servicio. Se registran en los dos agentes desde `config/mcp.toml`.
4. **Proyectos conectados** (sección 4.1), si la persona tiene:
   - registra cada uno en `config/workspaces.toml` con su modo;
   - para observar, acuerda el archivo de estado y propón el cambio mínimo para que la rutina lo escriba al terminar. Ese cambio lo aplica la persona en su proyecto, con las reglas de ese proyecto. El sistema no agenda ni modifica la rutina;
   - el puente escribe la nota de cada proyecto en `<bóveda>/_os/proyectos/`.

**Puerta 7:**
- una semana de ejecuciones con sus recibos;
- si hay proyectos conectados, la tarjeta de un proyecto de prueba muestra los tres estados (correcto, falló y no corrió), y con el puente apagado la rutina de un proyecto real sigue corriendo;
- una búsqueda automática de patrones de secretos en el código y en la bóveda, sin resultados.

### Fase 8 — Uso diario y documentación de lo construido

- Script para arrancar y parar.
- Arranque al iniciar sesión, según el sistema (sección 6.7).
- Comprobaciones de salud y reinicio del servicio de voz si se cae.
- `docs/operacion.md`: cómo arrancar, parar, recalibrar el triaje y añadir una skill.
- `docs/arquitectura.md`: lo que de verdad quedó construido, tomado del código y de la configuración, no de este documento:
  - procesos, puertos y rutas;
  - los contratos tal como quedaron (API del puente, recibos, catálogo de skills, adaptadores);
  - las skills, automatizaciones y conectores activos;
  - las fases que se dejaron para después (por ejemplo, la voz);
  - enlaces a `docs/decisiones.md`.
- Actualiza el `AGENTS.md` del proyecto (sección 7): la referencia pasa a ser `docs/arquitectura.md` y `docs/operacion.md`. Este documento se queda en `docs/` como referencia histórica, para reconfigurar, activar lo que se dejó para después o reconstruir el sistema en otra computadora.

**Puerta 8:** en una sesión nueva, sin leer este documento, el agente responde con el `AGENTS.md` y `docs/arquitectura.md` cómo añadir una skill, dónde quedan los recibos y qué puerto usa el puente.

---

## 6. Opciones de referencia (a verificar en la Fase 0)

Estado comprobado en octubre de 2026 con las fuentes de la sección 11. Todo esto cambia rápido: vuelve a verificarlo antes de usarlo.

### 6.1 Triaje

| Opción | Cuándo | Notas |
|---|---|---|
| **Jev vía OpenRouter** (recomendado) | La persona tiene o quiere cuenta de OpenRouter | Modelo de decisión de TypeSafe: devuelve la opción elegida con probabilidades, no texto. Se usa con la clave de OpenRouter, sin cuenta de TypeSafe. Solo se paga la entrada (TypeSafe publicó 0,042 USD por millón de tokens). Fija una versión concreta, nunca el alias «latest»: en octubre de 2026 era `typesafe/jev-1.14`; comprueba la vigente en la Fase 0. Al cambiarla, recalibra. La API se llama `alpha`: va detrás de un adaptador con plan B. |
| Modelo pequeño vía OpenRouter | No quiere Jev | Pide salida JSON con la opción y valida el esquema. Menos calibrado: calibra el umbral con el conjunto de evaluación. |
| Modelo local | Hardware suficiente y prioridad de privacidad | Mismo esquema; mide latencia y aciertos en la Fase 2. |
| Solo reglas | Sin cuentas ni costes | Palabras clave y botones. Lo que no coincide va al nivel 2 o se pide aclaración. |

**Jev Router no es lo mismo.** `typesafe/jev-router` usa Jev para elegir qué LLM contesta un chat; no es el triaje.

Sonda de Jev para la Fase 0. Contrasta el esquema con la referencia actual de la Decisions API antes de darla por buena:

```python
# scripts/sonda_jev.py — ejecutar con: uv run --with httpx --with keyring python scripts/sonda_jev.py
import json, keyring, httpx

clave = keyring.get_password("agentic-os", "openrouter")   # la persona la guardó; nunca se imprime
cuerpo = {
    "model": "typesafe/jev-1.14",
    "state": {"frase": "resúmeme las noticias de IA de hoy",
              "skills": {"informe-noticias": "Informe diario de noticias", "plan-del-dia": "Plan del día"},
              "informes_de_hoy": ["_os/informes/informe-noticias/2026-10-27.md"]},
    "questions": {
        "nivel": {"type": "choice", "instructions": "¿Qué nivel debe atender esta petición?",
                  "criteria": {"nivel1": "Reglas o datos guardados, sin IA",
                               "nivel2": "Resumen rápido de informes que ya existen",
                               "nivel3": "Trabajo real: investigar, crear o ejecutar una skill o un conector"}},
        "efectos_externos": {"type": "noul",
                             "instructions": "La petición implica enviar, publicar, borrar o modificar algo fuera de la bóveda"},
    },
}
r = httpx.post("https://openrouter.ai/api/alpha/decisions",
               headers={"Authorization": f"Bearer {clave}"}, json=cuerpo, timeout=30)
r.raise_for_status()
print(json.dumps(r.json(), indent=2, ensure_ascii=False))
```

Al triaje solo se le manda la frase, la hora, el catálogo de skills y los nombres de los informes de hoy. Nunca contenido de la bóveda. El estado debe quedar por debajo de unos 32 000 bytes: por encima, Jev entra en un modo más lento y menos preciso.

### 6.2 Nivel 2

Un modelo pequeño y rápido. En OpenRouter, elige con la persona según precio y velocidad del momento; en local, solo si el hardware lo permite. El contexto se limita a los informes de `<bóveda>/_os/informes`.

### 6.3 Nivel 3: Claude Code y Codex

| Capa | Claude Code | Codex | Cómo unificar |
|---|---|---|---|
| Instrucciones | Lee `CLAUDE.md`; lee `AGENTS.md` solo si no hay `CLAUDE.md` | Lee `AGENTS.md` | `AGENTS.md` con el contenido + `CLAUDE.md` con `@AGENTS.md` |
| Skills | `.claude/skills/<id>/SKILL.md` | `.agents/skills/<id>/SKILL.md` | Canónicas en `.agents/skills`; un script mantiene la copia |
| Conectores (MCP) | `claude mcp add …` o `.mcp.json` | `[mcp_servers]` en la configuración de Codex | Lista única en `config/mcp.toml` + script que genera ambas |
| Ejecución sin interfaz | `claude -p` con salida JSON en streaming | `codex exec --json` | Interfaz común; cada adaptador normaliza eventos |
| Permisos | Modo de permisos y herramientas permitidas | Sandbox de escritura en el espacio de trabajo | Perfil por skill traducido por cada adaptador |

**Prohibido:** el modo que salta todos los permisos en Claude Code y el acceso total sin sandbox en Codex.

**Lo que de verdad limita a un trabajador.** Los agentes corren con el usuario de la persona. En los dos, escribir fuera de su carpeta está bloqueado por defecto. Leer fuera de ella, no del todo:
- **Claude Code:** su herramienta de lectura pregunta antes de salir de la carpeta, pero los comandos de solo lectura como `cat`, `ls` o `grep` corren sin preguntar. Su sandbox, que lo impide a nivel del sistema, existe en macOS, Linux y WSL2, no en Windows nativo.
- **Codex:** el sandbox limita la escritura al espacio de trabajo; la documentación no promete limitar la lectura.

Por eso el adaptador de Claude lanza cada trabajo sin preguntas pendientes y con un archivo de ajustes propio:

```text
claude -p --permission-mode dontAsk --settings <carpeta del código>/config/claude-worker.json …
```

```json
{
  "permissions": {
    "blockReadsOutsideWorkingDirectories": true,
    "deny": ["Read(//**/.env)"]
  }
}
```

- `blockReadsOutsideWorkingDirectories` hace que las herramientas de archivos rechacen lecturas fuera de la carpeta de cada trabajo.
- `//**/.env` cubre los `.env` de cualquier ruta absoluta, en todas las unidades en Windows.
- Con `dontAsk`, lo que no esté permitido en el perfil de la skill se deniega en lugar de quedarse esperando.
- En macOS, Linux o WSL2, activa además el sandbox de Claude Code.
- En Codex, usa `workspace-write`, o `read-only` para los trabajos que solo leen, y nunca rutas de escritura adicionales fuera del espacio de trabajo.

Estas opciones salen de la documentación de octubre de 2026: verifícalas en la Fase 3. Lo que no se pueda bloquear se le explica a la persona, y es otra razón para que sus claves nunca estén en archivos de texto.

Las opciones exactas cambian entre versiones: verifícalas con `--help`. Instalación según la documentación oficial:

- **Claude Code:**
  - Windows PowerShell: `irm https://claude.ai/install.ps1 | iex`;
  - macOS, Linux y WSL: `curl -fsSL https://claude.ai/install.sh | bash`;
  - también existen Homebrew y WinGet, y una app de escritorio con pestaña Code.
- **Codex CLI:** `npm install -g @openai/codex`; consulta la documentación oficial para tu sistema, sobre todo en Windows.

### 6.4 Voz según el hardware

| Hardware | Reconocimiento de voz (STT) | Voz de respuesta (TTS) |
|---|---|---|
| NVIDIA | faster-whisper en GPU. Las RTX 50 (Blackwell) necesitan CUDA 12.8+ y CTranslate2 4.5+ | Kokoro; Ming-3.0-tts-0.5B (experimental); Voxtral 4B TTS si hay 16 GB o más |
| Apple Silicon | whisper.cpp con Metal o mlx-whisper (verificar estado actual) | Kokoro; nube |
| AMD o Intel | whisper.cpp con Vulkan (verificar) o CPU | Kokoro; nube |
| Solo CPU | faster-whisper en `int8` con un modelo más pequeño; latencia mayor | Kokoro (es muy ligero); nube |
| Cualquiera | STT en la nube (con coste y privacidad distinta) | TTS en la nube |

Datos útiles sobre los TTS en español:

- **Kokoro-82M** (Apache 2.0): voces españolas `ef_dora`, `em_alex` y `em_santa`. Usa espeak-ng para fonemizar idiomas distintos del inglés; compruébalo e instálalo con permiso.
- **Ming-3.0-tts-0.5B** (inclusionAI, octubre de 2026, MIT):
  - siete idiomas, entre ellos el español, pero sin voces españolas de serie: hay que clonar una voz, con su transcripción, o diseñarla con una descripción;
  - el servidor HTTP documentado no hace streaming;
  - probado en GPUs de centro de datos con CUDA 12.4 y PyTorch 2.6;
  - la normalización de texto usa `pynini`, que puede requerir WSL2 en Windows, o desactivarse.
- **Voxtral 4B TTS** (Mistral): streaming, 20 voces de serie y nueve idiomas, entre ellos el español. Los pesos abiertos son CC BY-NC 4.0 (no comercial) y necesita una GPU de 16 GB o más. También hay API de pago.
- **Ranking independiente en español** (Artificial Analysis, voces controladas, 2026-09-22): el primero era Cartesia Sonic 3.6 (de pago) y el mejor de pesos abiertos, OpenAudio S1 Mini (CC-BY-NC-SA 4.0).
- **Licencias:** si la persona publica contenido monetizado con esa voz, las licencias no comerciales son una zona gris. Avísale y recomienda opciones MIT/Apache o una API comercial.

### 6.5 Obsidian

| Plugin | Para qué |
|---|---|
| El plugin `agentic-os` | Lo construyes tú en la Fase 6 |
| Sidebar Terminal | Terminales con Claude Code, Codex o el shell junto a las notas, en pestañas |
| BRAT | Instalar plugins beta desde GitHub |
| Hot Reload | Recargar el plugin al recompilar (está en GitHub, no en el directorio) |
| Notas diarias y Bases | Plugins del núcleo; Bases muestra los recibos como tabla |

Opcional: Claudian o Agent Client para un chat lateral con los agentes. Recomienda solo uno.

### 6.6 Secretos según el sistema

Usa la biblioteca `keyring` de Python. Guarda en el Administrador de credenciales de Windows, en el Llavero de macOS y, en Linux, en el Secret Service (GNOME Keyring o KWallet). En un Linux sin escritorio puede no haber servicio: pregunta a la persona qué prefiere.

La persona guarda la clave así (el valor no aparece en pantalla ni en el historial):

```text
uv run --with keyring python -c "import keyring, getpass; keyring.set_password('agentic-os', 'openrouter', getpass.getpass('Clave: '))"
```

### 6.7 Automatizaciones, arranque y tecla global

| | Windows | macOS | Linux |
|---|---|---|---|
| Tareas programadas y arranque | Programador de tareas | `launchd` (LaunchAgents) | Temporizadores `systemd --user` (o cron) |
| Tecla global | Funciona con bibliotecas como `pynput` | Requiere permisos de Accesibilidad y de Monitorización de entrada, y de micrófono | En X11 funciona; en Wayland la captura global está restringida: usa un atajo del escritorio que llame a una URL local, o el botón del panel |
| Audio | `sounddevice` (PortAudio) | Igual; permiso de micrófono | Igual; puede requerir el paquete de PortAudio del sistema |

---

## 7. Plantilla: `AGENTS.md` del proyecto

Escríbelo en la Fase 0 rellenando los valores. Mantenlo corto, por debajo de unos 20 KB: Codex trunca los archivos de instrucciones grandes.

```markdown
# AGENTS.md — sistema operativo agéntico

## Misión
Construir y mantener, por fases verificadas, el sistema operativo agéntico de <nombre> sobre la bóveda <ruta de la bóveda>.
Durante la construcción (fases 0 a 8), especificación: docs/INSTRUCCIONES-PARA-TU-AGENTE.md.
Al cerrar la Fase 8, sustituye la línea anterior por: «Arquitectura construida: docs/arquitectura.md · Operación: docs/operacion.md · Las instrucciones originales quedan en docs/ como referencia histórica».
Configuración: config/agentic-os.toml. Estado actual: docs/estado.md (léelo al empezar).

## Reglas
- No asumas nada: lo que no esté en la configuración, se pregunta.
- Sistema: <os>; shell: <shell>; rutas absolutas en formato <os>.
- Secretos: nunca los abras, imprimas ni pidas en el chat; el código los lee con keyring (servicio "agentic-os").
- Bóveda: cambios solo con diff y aprobación; nunca escribir en <carpeta de solo lectura>; el sistema escribe en <carpeta_os_en_boveda>.
- Fases con puertas: al cerrar cada fase, evidencia en docs/evidencia/, estado en docs/estado.md, commit, y detenerse.
- Verificar opciones de CLI y esquemas de API antes de usarlos; anotar en docs/decisiones.md con fecha y versión.
- Más de 10 archivos a la vez: lista primero y aprobación.
- Proyectos conectados (config/workspaces.toml): el sistema los observa, los ejecuta o les delega trabajo, pero nunca los modifica desde aquí. Nunca registrar esta carpeta, la bóveda ni una carpeta padre.
- Idioma con la persona: <idioma>. Código e identificadores en inglés.
```

## 8. Plantilla: sección para el `AGENTS.md` de la bóveda

Se añade en la Fase 0, como diff y con aprobación.

```markdown
## Sistema operativo agéntico
- Los agentes lanzados por el sistema (nivel 3) trabajan aquí y escriben solo en <carpeta_os_en_boveda>/ y donde estas reglas lo permitan.
- Informes: <carpeta_os_en_boveda>/informes/<skill>/AAAA-MM-DD.md, con fuentes citadas.
- Cada trabajo deja un recibo en <carpeta_os_en_boveda>/recibos/.
- <carpeta de solo lectura>/ sigue siendo de solo lectura.
- Antes de enviar, publicar, borrar o modificar algo fuera de la bóveda: detenerse y pedir confirmación.
- Las skills canónicas viven en .agents/skills/; .claude/skills/ es una copia generada.
- <carpeta_os_en_boveda>/proyectos/ la escribe el puente: una nota por proyecto conectado. No la edites a mano.
```

---

## 9. Contratos internos

Son el punto de partida. Cuando exista `docs/arquitectura.md` (Fase 8), el contrato vigente es el de ese archivo.

### API del puente (solo `127.0.0.1`, con token)

- `POST /request` → `{source: "hud"|"obsidian"|"voice"|"schedule", kind: "text"|"click", text?, skill_id?, workspace?}` → `{request_id}`
- `GET /jobs/{id}`, `POST /jobs/{id}/cancel`, `POST /jobs/{id}/confirm`
- `GET /state` (panel, agente activo, cola, salud)
- `GET /projects` (estado de cada proyecto conectado)
- `POST /runner` → `{"runner": "claude"|"codex"}`
- `WS /events` → `decision`, `job_started`, `job_progress`, `job_done`, `job_failed`, `speak`, `needs_confirmation`, `needs_clarification`, `project_status`

### Recibo

Una línea JSONL por petición en `<bóveda>/_os/recibos/AAAA-MM-DD.jsonl`, con estos campos:

`ts, request_id, source, input, route ("rule"|"click"|"triage"|"fallback"), tier, triage: {model, probabilities, confidence, side_effects_p, latency_ms, cost_usd}, runner?, session_id?, skill_id?, workspace?, exit_code?, status, duration_ms, output_path?, error?`

Los trabajos de nivel 3 generan además un `.md` con ese contenido en el frontmatter.

### Catálogo de skills (`config/skills.toml`)

`id, descripcion, nivel (1|3), workspace, efectos_externos (bool), permisos (perfil), ruta_informe, timeout_s`

### Proyectos conectados (`config/workspaces.toml`)

`id, ruta, modos (observar|ejecutar|delegar), estado?, frecuencia?, hora?, margen_min?, programado_por?, comando? (lista de argumentos), escritura? (bool, por defecto false)`

### Archivo de estado de un proyecto

`rutina, inicio, fin, ok (bool), resumen, informe?`. Lo escribe el proyecto; el puente solo lo lee.

### Interfaz de los adaptadores de agentes

```python
class Runner(Protocol):
    name: Literal["claude", "codex"]
    async def run(self, job: Job) -> AsyncIterator[Event]: ...
    async def cancel(self, job_id: str) -> None: ...
```

El directorio de trabajo es el del workspace (por defecto, la bóveda), nunca la carpeta del código.

---

## 10. Seguridad y privacidad (explícaselo a la persona)

- **El nivel 3 actúa.** Con conectores puede enviar correos o cambiar el calendario. Por eso: permisos por skill, nada de modos sin límites y confirmación para lo que sale de la computadora.
- **Costes.** El triaje cuesta céntimos; el nivel 2 cobra por token; el nivel 3 consume el límite de la suscripción del agente. La cola ejecuta un trabajo a la vez por defecto.
- **Qué sale de la computadora:**
  - la frase va al servicio de triaje (OpenRouter/TypeSafe);
  - el nivel 2 manda a la nube los informes que resume;
  - el nivel 3 lee contenido de la bóveda y lo procesa el proveedor del agente.
  - La voz, si el STT es local, no sale de la computadora.
  - Lo sensible, en una bóveda aparte o con un modelo local.
- **Solo localhost.** El puente escucha en `127.0.0.1`. Abrirlo a la red local o a internet requiere autenticación y una decisión explícita de la persona.
- **Proyectos conectados.** El sistema solo trabaja en los que la persona registre, en el modo mínimo que necesite. En Windows, leer fuera de la carpeta no se puede bloquear del todo (sección 6.3): conecta solo lo necesario, y si un proyecto maneja dinero o credenciales, solo para observar.

---

## 11. Fuentes (consultadas en octubre de 2026)

- Jev en OpenRouter (Decisions API, primitivas, acceso con clave de OpenRouter): https://openrouter.ai/docs/guides/community/jev.md
- Jev Router: https://openrouter.ai/docs/guides/routing/routers/jev-router
- Precio de Jev (TypeSafe vía LiteLLM): https://docs.litellm.ai/blog/typesafe_jev
- Claude Code (instalación, AGENTS.md, modo sin interfaz): https://code.claude.com/docs/en/overview
- Claude Code y AGENTS.md cuando también hay CLAUDE.md: https://chudi.dev/blog/claude-code-agents-md
- Codex y AGENTS.md: https://developers.openai.com/codex/guides/agents-md
- Rutas de skills en Codex y Claude Code: https://docs.kanaries.net/articles/codex-vs-claude-code-skills
- Ming-3.0-tts-0.5B: https://huggingface.co/inclusionAI/Ming-3.0-tts-0.5B
- Ming-3.0 en despliegue local: https://aireiter.com/es/blog/ming-3-0-tts-review-local-deployment
- Voxtral 4B TTS: https://huggingface.co/mistralai/voxtral-4b-tts-2603
- Ranking de voz en español (Artificial Analysis, resumen): https://alphasignal.ai/news/cartesia-s-sonic-3-6-dominates-eight-of-nine-global-voice-ai-leaderboards
- Kokoro-82M, voces: https://featherless.ai/docs/audio-speech-model-kokoro
- Blackwell, CUDA 12.8 y CTranslate2: https://github.com/pluja/whishper/issues/172
- Sidebar Terminal: https://community.obsidian.md/plugins/sidebar-terminal
- Claude Code, permisos (lectura fuera de la carpeta, reglas de denegación, rutas en Windows): https://code.claude.com/docs/en/permissions.md
- Claude Code, sandbox (plataformas y alcance): https://code.claude.com/docs/en/sandboxing.md
- Claude Code, seguridad (límite de la carpeta de trabajo): https://code.claude.com/docs/en/security.md
- Claude Code, opciones de la línea de órdenes (`--settings`, `--permission-mode`): https://code.claude.com/docs/en/cli-reference.md
- Claude Code, skills (proyecto, usuario y carpetas anidadas): https://code.claude.com/docs/en/skills.md
- Codex, skills (repositorio y usuario): https://learn.chatgpt.com/docs/build-skills
- Codex, sandbox: https://learn.chatgpt.com/docs/sandboxing
