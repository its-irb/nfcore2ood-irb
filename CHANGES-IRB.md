# Cambios del IRB sobre nfcore2ood

Este repositorio es un fork de [TuftsRT/nfcore2ood](https://github.com/TuftsRT/nfcore2ood)
(versión 1.4.0, commit base `68f2230`). Este documento recoge **todo lo que hemos cambiado
respecto al original y por qué**. Las apps de Open OnDemand (OOD) que genera el IRB dependen
de estos cambios para funcionar como queremos.

- La rama **`main`** es idéntica a Tufts y no se toca nunca.
- La rama **`irb`** es `main` más estos cambios. Es la que se usa en el cluster.
- Cada cambio tiene un identificador `IRB-xx`, y el código lleva esa misma etiqueta en un
  comentario. Para verlos todos: `grep -rn "IRB-" .`

## Qué hace el generador con estos cambios

El generador (`nf2ood`) se ejecuta **a mano**, una vez por pipeline y versión, y crea una app de
OOD a partir de una pipeline de nf-core ya descargada. No hace nada por sí solo.

Cuando una persona lanza esa app desde OOD:

1. OOD valida el formulario (`submit.yml.erb`) y envía a Slurm un job de orquestación, en la
   partición elegida en el formulario.
2. El job (`template/script.sh.erb`) carga el módulo de Nextflow, lee el parser que corresponde a
   la pipeline y lanza `nextflow run` con el perfil del IRB.
3. Nextflow envía todas las tareas a esa misma partición y lee las imágenes de la biblioteca
   compartida.

## Resumen

| ID | Cambio | Ficheros |
|---|---|---|
| IRB-01 | La partición viene del dashboard de OOD (`auto_queues_cpu`) | `cpu_partition.yml`, `json2ood.py`, `submit.yml.erb`, `script.sh.erb` |
| IRB-02 | Solo Slurm: sin `executor`, `num_cores` ni `num_memory` | `json2ood.py`, `form.template.erb`, `submit.yml.erb`, `script.sh.erb` |
| IRB-03 | `workdir` obligatorio; `workdir` y `outdir` fuera del home | `form.template.erb`, `submit.yml.erb`, `script.sh.erb` |
| IRB-04 | Perfil del sitio y `-c` obligatorio al lanzar Nextflow | `script.sh.erb` |
| IRB-05 | No se define `NXF_SINGULARITY_CACHEDIR` | `script.sh.erb` |
| IRB-06 | Parser de Nextflow por pipeline (`nxf_syntax_parser`) | `script.sh.erb` |
| IRB-07 | Versiones de Nextflow: lista, valor por defecto de Lmod y error si no hay | `nf2ood` |
| IRB-08 | Categoría de la app en el menú de OOD | `nf2ood`, `manifest.yml` |
| IRB-09 | Los `select` numéricos llegan como número | `json2ood.py` |

Los commits de la rama `irb` agrupan estos cambios así: `IRB-09`, `IRB-07 + IRB-08` y
`IRB-01 a IRB-06`.

El riesgo de conflicto al integrar una versión nueva de Tufts es **alto** donde se reescribió una
zona entera (`submit.yml.erb`, `script.sh.erb`), **medio** en `nf2ood` y `json2ood.py`, y **bajo**
en `manifest.yml`.

---

## IRB-01: la partición viene del dashboard de OOD

**Qué hace.** El formulario no define la partición: la aporta el dashboard de OOD como atributo
inteligente `auto_queues_cpu`. Esa misma partición se usa para el job de lanzamiento
(`-p <partición>`) y, vía `NF_QUEUE`, para el perfil de Nextflow, de modo que las tareas van a la
misma partición.

**Por qué.** Cada persona solo debe ver las particiones a las que tiene acceso, y eso ya lo resuelve
el dashboard. Elegir una partición para el job y otra para las tareas daría errores confusos.

**Dónde.** `cpu_partition.yml` (queda solo un comentario), `json2ood.py` (`STATIC_FORM_FIELDS`
usa `auto_queues_cpu` en lugar de `partition`), `submit.yml.erb` (`-p`), `script.sh.erb`
(`NF_QUEUE` y `SBATCH_PARTITION`; falla si no hay partición).

**Cómo comprobar.** Generar una app y abrirla en OOD: el formulario muestra el selector de
partición del dashboard. Al lanzar, el job y las tareas salen en la partición elegida (`squeue`).

## IRB-02: solo Slurm

**Qué hace.** Se eliminan los campos `executor`, `num_cores` y `num_memory`, y el modo de ejecución
`local` con su `custom.config`. El job de OOD pide siempre 2 CPUs y 12 GB, porque solo orquesta
Nextflow.

**Por qué.** No ofrecemos ejecutar las pipelines dentro del job de OOD: el trabajo real va a Slurm,
con los límites de cada partición.

**Dónde.** `json2ood.py`, `form.template.erb`, `submit.yml.erb`, `script.sh.erb`.

**Cómo comprobar.** El formulario generado no tiene esos campos.

## IRB-03: `workdir` obligatorio y fuera del home

**Qué hace.** `workdir` es obligatorio y debe ser una ruta absoluta que no esté dentro del home.
Si `outdir` es una ruta absoluta, tampoco puede estar en el home. Se comprueba **dos veces**: al
pulsar Launch (`submit.yml.erb`, en Ruby) y dentro del job (`script.sh.erb`, en bash) como red de
seguridad.

**Por qué.** El home es para configuración y datos pequeños. Una pipeline escribe `work/` y
resultados muy grandes, y llenaría el home de la persona.

**Dónde.** `form.template.erb`, `submit.yml.erb`, `script.sh.erb`.

**Cómo comprobar.** Lanzar con `workdir` dentro de `/home/...` debe dar error. El bloque de bash se
probó en 7 combinaciones. **La parte Ruby de `submit.yml.erb` no se ha podido probar fuera de OOD**
(no hay `ruby` en el cluster): hay que verla funcionar al menos una vez en OOD.

## IRB-04: perfil del sitio y `-c` obligatorio

**Qué hace.** Nextflow se lanza con `-profile <perfil del sitio>,<partición>` y
`-c $NF2OOD_PROFILE_CONFIG`. El perfil de la partición va **después** del perfil del sitio para
sobrescribir sus valores por defecto.

**Por qué.** El fichero de configuración del IRB define el executor Slurm, Singularity, la
biblioteca de imágenes y los límites de CPU y memoria de cada partición.

**Dónde.** `script.sh.erb`. La variable `NF2OOD_PROFILE_CONFIG` viene de `nf2ood.env`.

**Limitación conocida.** Si `NF2OOD_PROFILE_CONFIG` no está en `nf2ood.env`, el generador **no se
queja** y la app falla al lanzarla. Pendiente: validarla en `nf2ood` al generar (propuesta IRB-10).

**Cómo comprobar.** En la salida del job, la línea `Executing: nextflow run ...` debe incluir
`-profile irb,<partición>` y `-c .../irb.config`.

## IRB-05: no se define `NXF_SINGULARITY_CACHEDIR`

**Qué hace.** Se elimina la variable `NXF_SINGULARITY_CACHEDIR` del script. Las imágenes se leen de
la biblioteca compartida (`singularity.libraryDir`, definida en el fichero de configuración del
IRB), que es de solo lectura.

**Por qué.** Esa variable indica dónde **descarga** Nextflow las imágenes que faltan. Si apuntara a
la biblioteca compartida, un usuario sin permisos fallaría y, con permisos, varios usuarios
escribiendo a la vez podrían corromper imágenes. Sin la variable, lo que falte se descarga en el
directorio de trabajo de cada persona.

**Dónde.** `script.sh.erb`.

## IRB-06: parser de Nextflow por pipeline

**Qué hace.** Antes de lanzar, el script lee el fichero `nxf_syntax_parser` que hay junto al código
de la pipeline (`<pipeline>/<versión>/nxf_syntax_parser`, con `v1` o `v2`) y fija
`NXF_SYNTAX_PARSER`. Si el fichero no existe, se deja el valor que ya hubiera en el entorno.

**Por qué.** Desde Nextflow 26 hay dos parsers de configuración y cada versión de pipeline solo
funciona con uno. Lo decide el script de descarga al instalar la pipeline.

**Dónde.** `script.sh.erb`. Requiere que el script de descarga (o `backfill_parser_files.sh`) haya
escrito el fichero; esos scripts **no** están en este repo.

## IRB-07: versiones de Nextflow

**Qué hace.** El desplegable de versión de Nextflow:
- puede venir de una lista explícita (`NF2OOD_NEXTFLOW_VERSIONS`, con `NF2OOD_NEXTFLOW_DEFAULT`);
- si no, se genera con `module avail` y **preselecciona la versión que Lmod marca como `(D)`**;
- si no encuentra ninguna versión, **da error** en vez de dejar un campo de texto vacío.

**Por qué.** Queremos que por defecto se use la versión aprobada por el equipo del cluster, y que un
fallo de `MODULEPATH` no produzca en silencio una app con el campo vacío.

**Dónde.** `nf2ood` (`discover_nextflow_versions`, `render_nextflow_version_field`).

**Cómo comprobar.** Con `module avail nextflow` visible, generar una app y mirar `form.yml`: la
versión preseleccionada es la `(D)`. Sin `module`, el generador debe parar con un mensaje claro.

## IRB-08: categoría de la app

**Qué hace.** `NF2OOD_CATEGORY` se escribe en el campo `category` de `manifest.yml`.

**Por qué.** Para agrupar las apps de nf-core en el menú de OOD.

**Dónde.** `nf2ood`, `manifest.yml`.

## IRB-09: `select` numéricos como número

**Qué hace.** Un parámetro entero con lista cerrada de opciones (por ejemplo `read_length` en
atacseq) se muestra como desplegable, pero ahora llega a Nextflow como número y no como texto.

**Por qué.** Sin esto, la validación del esquema rechaza `'50' is not a valid choice`.

**Dónde.** `json2ood.py` (`render_params_entry`).

**Nota.** Es un fallo del original, no una preferencia nuestra: es buen candidato para enviarlo a
Tufts como pull request y quitarlo de esta lista.

---

## Lo que este repo necesita y no incluye

El generador solo funciona con estas piezas, que viven en el cluster:

| Pieza | Dónde |
|---|---|
| **`nf2ood.env`** del IRB | `/apps/scripts/nf-core-pipelines/config/nf2ood.env` |
| Configuración de Nextflow del IRB (`irb.config`) | `/apps/scripts/nf-core-pipelines/config/irb.config` |
| Pipelines descargadas, con su `nxf_syntax_parser` | `/apps/scripts/nf-core-pipelines/pipelines/` |
| Biblioteca de imágenes de contenedores | `/apps/scripts/nf-core-pipelines/singularity-images/` |
| Módulo Lmod `nextflow` | `/apps/scripts/nextflow/modulefiles/` |
| Atributo `auto_queues_cpu` en el dashboard de OOD | configuración de OOD |

**El `nf2ood.env` del IRB no es el `nf2ood.env.example` que viene en este repo.** El `.example` es el
de Tufts, con sus rutas y su perfil, y solo sirve de plantilla. Los valores reales del IRB están en
el fichero compartido del cluster. Además de las variables de Tufts, el del IRB define
`NF2OOD_PROFILE_CONFIG` y `NF2OOD_CATEGORY` (IRB-04 e IRB-08), y puede definir
`NF2OOD_NEXTFLOW_VERSIONS` y `NF2OOD_NEXTFLOW_DEFAULT` (IRB-07).

Los scripts de descarga de pipelines e imágenes tampoco están aquí: aunque `download_nfcore_pipeline.sh`
de Tufts sigue en el repo, en el IRB no se usa.

## Cómo generar una app

```bash
source /apps/scripts/nf-core-pipelines/config/nf2ood.env
./nf2ood -p <pipeline> -v <versión> -o <carpeta de destino> --dry-run   # simulacro
./nf2ood -p <pipeline> -v <versión> -o <carpeta de destino>             # generar
```

Siempre con `-p` y `-v`, y **sin `--force`**, para no regenerar apps que no toca. Generar una app no
la publica en OOD.

## Cómo integrar una versión nueva de Tufts

```bash
git checkout irb
git fetch upstream
git log --oneline irb..upstream/main     # qué ha cambiado Tufts
git merge upstream/main
```

- Si Tufts tocó partes distintas, git lo junta solo.
- Si hay conflicto, el fichero queda marcado. Busca las etiquetas `IRB-xx` de la zona y mira su
  ficha en este documento para decidir qué conservar. Para abortar: `git merge --abort`.

**Después de cada merge, hay que repetir la prueba de regresión**: generar una app en una carpeta
nueva y compararla con la que sabemos que funciona.

```bash
source /apps/scripts/nf-core-pipelines/config/nf2ood.env
mkdir <carpeta nueva>
./nf2ood -p atacseq -v 2.1.2 -o <carpeta nueva>
diff -r <app que funciona>/nf-core-atacseq-2-1-2 <carpeta nueva>/nf-core-atacseq-2-1-2
```

Sin diferencias significa que el generador sigue produciendo lo mismo. Las etiquetas `IRB-xx` están
también en los ficheros de la app generada (son comentarios), así que una app generada antes de
añadirlas solo difiere en esos comentarios.

## Pendiente

- **IRB-10 (propuesta):** que `nf2ood` compruebe `NF2OOD_PROFILE_CONFIG` al arrancar (ver IRB-04).
- Partes del `README.md` describen el comportamiento de Tufts y no se han reescrito, para no
  provocar conflictos al integrar versiones nuevas (modo `local`, campo de partición, flujo con
  `download_nfcore_pipeline.sh`).
- Probar la parte Ruby de `submit.yml.erb` (IRB-03) en OOD.
