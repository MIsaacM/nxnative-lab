# CLAUDE.md — nxnative-lab

Proyecto de pruebas y optimización basado en **nxnative** (https://github.com/seedds/nxnative, licencia MIT).
nxnative convierte juegos de Switch que el usuario posee (NSP/XCI + `prod.keys` de su propia consola) en apps nativas para Mac, iPhone y iPad. La Switch y Apple Silicon comparten ARM64, así que el código del juego se ejecuta directamente. Se reemplaza solo lo específico de Nintendo:

- **Reescritor de NSO** (`dm/tools/patch_nso.py`, `DM_RULESET=native`): parchea las instrucciones que dependen del sistema de la Switch.
- **Host** (`dm/host`): sustituye al kernel Horizon y a los servicios del sistema (hilos, sincronización, memoria, FS, HID, audio, vídeo, teclado, ciclo de vida).
- **Gráficos** (`dm/gfx`): NVN sobre Metal. Los shaders Maxwell se traducen offline: Ryujinx → SPIR-V → SPIRV-Cross → MSL.
- **Perfiles por juego** (`dm/games/<titleID>/`): `game.tsv` más las tablas de shaders.
- **App iOS** (`dm/ios`) y empaquetado (`scripts/build_ipa_from_nsp.py`, `dm/ios/tools/build_ipa.sh`).

El idioma de trabajo con el usuario es **español**. Código, commits y comentarios van en inglés para poder enviar mejoras upstream (`AGENTS.md` exige inglés en todo el repo). Este CLAUDE.md es local del lab: nunca debe entrar en una rama `pr/*`.

---

## Objetivo

1. Reproducir nxnative en el Mac del usuario con un juego soportado que él posea.
2. Medir de forma reproducible: fps, tiempo de frame, tirones por compilación de shaders y memoria.
3. Optimizar donde haya margen y medir antes y después de cada cambio.
4. Enviar upstream (PR a seedds/nxnative) lo que sea genérico y esté probado.

El alcance es **solo Switch**. Otras consolas irán en otro proyecto.

---

## Reglas innegociables

- **Nunca** incluir en el repo, en logs compartidos ni en commits datos de juegos, NSP, XCI, RomFS, ExeFS, módulos reescritos, shaders extraídos ni `prod.keys`.
- **Nunca** leer, imprimir ni copiar el contenido de `prod.keys`. Solo se pasa su ruta a los scripts.
- Las carpetas de trabajo de los juegos (`<NSP folder>/<titleID>-work` y `<NSP folder>/<titleID>_build`) viven **fuera del repo** y fuera de la carpeta personal (regla de `AGENTS.md`). Los patrones `*.nsp`, `*.nsz`, `*.xci`, `*.xcz`, `*.keys`, `*-work/`, `*_build/`, `*_survey/`, `*.ipa`, `pso_user.tsv` y `save/` están en `.git/info/exclude`, que es local, para no tocar el `.gitignore` de upstream. Antes de cada commit, comprobar que siguen ahí (`git check-ignore -v a.nsp`).
- No usar ni descargar juegos ni claves de terceros. Si falta algo, pedírselo al usuario.
- No regenerar el traductor de shaders con un `dotnet publish` nuevo. Se usa el binario precompilado `dm/gfx/tools/dmsh_xlate_bin`, que coincide con `dmsh_chain.lock`, porque una compilación nueva cambia las huellas. Si hay que tocar el traductor, se hace en una rama aparte y se avisa al usuario.
- No hacer cambios masivos de estilo ni refactors sin pedirlo. Los diffs deben ser pequeños y fáciles de revisar, para que upstream los acepte.

---

## Fuentes de verdad (leer antes de tocar nada)

En este orden: `AGENTS.md` (reglas de upstream para agentes) → `README.md` → `docs/development.md` → `docs/agents/README.md` y sus capítulos (bring-up, gráficos, recetas de depuración, `06-dev-loop.md` antes de un PR) → `dm/games/README.md` → la nota del juego, `dm/games/<id>/README.md`.
Si este CLAUDE.md contradice al repo, **manda el repo**. En ese caso, actualiza este archivo y avisa al usuario.

---

## Configuración del repo

```bash
# Fork de seedds/nxnative en la cuenta del usuario, luego:
git clone <fork> nxnative-lab && cd nxnative-lab
git remote add upstream https://github.com/seedds/nxnative.git
sudo xcode-select -s /Applications/Xcode.app   # Xcode 27.1 exacto: dmsh_chain.lock fija su compilador Metal
xcodebuild -downloadComponent MetalToolchain
brew install xcodegen zstd glslang python@3.14   # Python >= 3.12, arm64
brew install --cask dotnet-sdk
python3 -m pip install lz4 capstone fonttools nsz "backports.zstd; python_version < '3.14'"
```

En el Mac del usuario: el Python de Homebrew no admite `pip install` global, así que los módulos están en el venv `/Users/Shared/Development/venvs/nxnative` (Python 3.14, arm64). Antes de cualquier script: `source /Users/Shared/Development/venvs/nxnative/bin/activate`. Homebrew tiene taps no fiables heredados de otra cuenta (mongodb, php). Si `brew install` se niega por eso, usar `HOMEBREW_NO_INSTALLED_DEPENDENTS_CHECK=1` en lugar de confiar en esos taps.

No instalar `spirv-cross` de Homebrew: el repo usa el suyo (`dm/gfx/tools/spirv_cross_bin`), cuya huella fija `dmsh_chain.lock`. Una versión de Homebrew no coincide (issue #242 de upstream).

Ramas:
- `main`: idéntica a `upstream/main`, sin commits propios.
- `lab`: `main` más los archivos del lab (este CLAUDE.md, `docs/lab/`, `bench/results/`). Se actualiza fusionando `main` tras sincronizar con upstream.
- `exp/<tema>`: experimentos, que salen de `lab`.
- `pr/<tema>`: lo candidato a PR, que sale de `main` (nunca de `lab`) y no lleva archivos del lab.

Sincronizar con upstream antes de empezar cada experimento. El fork (`origin`) es público: no subir logs completos (rutas locales, hashes de módulos), solo resúmenes. `bench/results/*.log` y `*.frames.tsv` están en `.git/info/exclude`.

---

## Primera sesión (checklist)

1. Comprobar las herramientas: `xcodebuild -version` (27.1), `xcrun metal --version` (debe coincidir con `dmsh_chain.lock`), `dotnet --version`, `glslang --version`, `xcodegen --version`, `python3 --version` (>= 3.12) y los módulos de Python. El propio script comprueba todo esto al empezar (`check_prereqs`).
2. Leer las fuentes de verdad y hacer al usuario un resumen breve de lo que ha cambiado respecto a este archivo.
3. Preguntar al usuario: qué juego soportado tiene, la ruta de la carpeta de NSP y la ruta de `prod.keys`. No suponer rutas. (Mac del usuario: M1 Pro, 16 GB.)
4. Pasar los tests que no necesitan juego: `for f in dm/tools/tests/test_*.py; do python3 -m unittest "$f"; done` (como en `.github/workflows/tests.yml`) y `make -C dm/host/tests GAME=<id> run`.
5. Construir para desarrollo: `python3 scripts/build_ipa_from_nsp.py <dir> -k <keys> --platform mac`. Compila solo lo que necesita el Mac (shaders de Mac y `<id>_build/tools/dmhost`), conserva lo extraído y su última línea es el comando exacto para lanzar el juego. `--platform mac-app` solo hace falta para tener una `.app` de doble clic.
6. Primer arranque con el comando que imprime el paso 5 (ya incluye `DM_WINDOW=1 DM_AUDIO=1`). `DM_SHOW_FPS` solo funciona en iOS. En Mac, los fps y tiempos de frame se sacan de `DM_NVN_TSV` o de `scripts/bench.py`. Guardar el log en `bench/results/<fecha>-<juego>-baseline.log`.
7. Añadir una ruta de benchmark para el juego (sección siguiente) y registrar la línea base.

---

## Benchmark reproducible

Upstream ya tiene el arnés: `scripts/bench.py <NSP folder>/<id>_build` (ver "A benchmark on the Mac" en `docs/development.md`). Usarlo en lugar de crear uno propio:

- Las rutas se definen en `dm/games/<id>/bench.tsv`, una por línea: nombre, `DM_KEYS` (segundos desde el arranque), inicio y duración de la ventana medida, y descripción. Hoy solo la tiene `01007EF00011E000`. Añadir la del juego del usuario es un PR pequeño y útil para upstream.
- Cada ejecución parte de una copia de la misma partida (`--save`, por defecto `<id>_build/run/save`).
- `--variant 'label:KEY=VALUE,…'` compara switches, otro host (`host=`), otros shaders (`msl=` con `shmap=`) u otro archivo de pipelines (`pso=`). `--runs N` repite cada variante (por defecto 2; usar 3) y el resultado son medianas con su rango. Para comparar con y sin prewarm: `--variant 'noprewarm:DM_PSO_MANIFEST=off'`.
- Da p50/p95/p99/máx del tiempo de frame y del tiempo en pantalla, el tiempo de GPU, los frames perdidos y las líneas `[RED]`. Escribe `results.md`, `results.json` y capturas en `<id>_build/bench/<fecha-hora>/`. Copiar `results.md` a `bench/results/` del lab.
- Comparar solo variantes de una misma invocación, porque la carga del Mac mueve los números.
- No mide memoria pico ni compilaciones de pipeline en caliente. Para eso: `DM_PSO_WAITLOG=1` (cada draw que esperó más de 20 ms por su pipeline) e Instruments (Allocations). Si hace falta un script propio, que sea un complemento pequeño en `bench/` del lab, no un sustituto de `bench.py`.

Variables útiles del host en Mac: `DM_ROOT`, `DM_ROMFS`, `DM_SAVE`, `DM_TEMP`, `DM_MSL_DIR`, `DM_FONT_DIR`, `DM_NVN_SHMAP`, `DM_WINDOW`, `DM_AUDIO=1` (audio en Mac), `DM_KEYS`, `DM_KBD`, `DM_NVN_TSV` (filas de tiempos por frame), `DM_FPS_TARGET`, `DM_OPMODE` (0 = portátil), `DM_PSO_MANIFEST`, `DM_PSO_RECORD`, `DM_PSO_USER` y `DM_PSO_WAITLOG`. La lista completa y documentada está en `docs/env_switches.tsv`.

Herramientas de perfilado: Instruments (Time Profiler, Metal System Trace, Allocations), captura de frame GPU de Xcode y `MTL_HUD_ENABLED=1` para el HUD de Metal.

**Regla:** ningún cambio de rendimiento se da por bueno sin números de antes y después en `bench/results/` y una línea en `docs/lab/CHANGELOG-lab.md`.

---

## Líneas de optimización (por prioridad)

1. **Tirones por shaders/PSO:** sembrar `dm/games/<id>/pso_manifest.tsv` grabando en Mac (`DM_PSO_MANIFEST=off DM_PSO_RECORD=<file>`) los escenarios iniciales, como describe `docs/development.md` ("Prewarm lists"). Unas 45 de las 97 listas ya tienen entradas (comprobar con `grep -vc '^#' dm/games/<id>/pso_manifest.tsv`). Si la del juego del usuario está vacía o solo cubre el arranque, es la mejora más barata y fácil de enviar.
2. **Huecos conocidos de gráficos** (según `docs/development.md`): conditional render siempre dibujado, mapeos sparse del virtual pool, `DrawElementsIndirect`, optimizaciones TBDR desactivadas por los bindless textures, y en Mac el texto de menús de Mario Wonder sale en blanco. Antes de tocar nada, leer el capítulo de gráficos de `docs/agents`.
3. **CPU del host:** perfilar sincronización (address arbiter, condvars), envío de comandos a Metal y audio. Buscar contención de locks y copias innecesarias.
4. **Memoria:** reparto del bloque guest y del application pool por juego. Medir con Allocations, porque en iPhone el límite es de unos 3,4 GB sin el entitlement de memoria ampliada.
5. **Calidad de vida en Mac:** entrada con mando nativo (ver el PR upstream #296 de Pro Controller/gyro antes de duplicar trabajo), ventana y escalado.

Antes de empezar cualquier línea, revisar los issues y PRs abiertos upstream para no duplicar trabajo.

---

## Forma de trabajar

- Ciclo: **hipótesis → medición base → cambio mínimo → medición → decisión**. Si no mejora, se revierte y se anota por qué en `docs/lab/notes.md`.
- Un cambio por commit, con un mensaje que incluya el efecto medido (p. ej. `gfx: cache sampler descriptors (+6% avg fps, MarioWonder/menu)`).
- Los cambios específicos de un juego van en su perfil o se protegen por build ID, como hace upstream. Nunca deben romper a otros juegos.
- Después de tocar `dm/host` o `dm/gfx`, recompilar (`make -C dm/host GAME=<id> dmhost`, o volver a lanzar el script con `--platform mac`, que reutiliza lo que no ha cambiado) y pasar los tests sin juego.
- Antes de un PR que toque la cadena de shaders, el host o las herramientas: `scripts/regress.py check --games` en los juegos locales a los que más pueda afectar (`scripts/regress.py impact` ayuda a elegirlos; upstream pide unos cinco) y poner el informe en el PR (`docs/agents/06-dev-loop.md`). Con un solo juego local, se pasa ese y se dice en el PR.
- Si un paso necesita al usuario (mando, mirar la pantalla, confirmar algo visual), parar y pedírselo de forma concreta.
- Al final de cada sesión, resumir al usuario en español: qué se probó, los números obtenidos y el siguiente paso propuesto.

---

## Estructura añadida por este proyecto

```
bench/results/         # resultados de scripts/bench.py y logs de línea base (sin datos del juego)
docs/lab/notes.md      # hipótesis, hallazgos y callejones sin salida
docs/lab/CHANGELOG-lab.md  # cambios aceptados con su medición
```

Todo lo demás es upstream. Mantenerlo lo más parecido posible para que los merges sean triviales.
