# CLAUDE.md

Guía para Claude Code en este repositorio. Responder y redactar en **español**.

## Proyecto

**EcoStream** — proyecto final de *Estructuras de Datos y Algoritmos* (UTEC, Maestría en Ciencia de Datos e IA, 2026-2, Prof. Heider Sánchez). Grupo 3, **Escenario B: comercio electrónico**.

Pipeline de data streams de **10 etapas (E0–E9)**: cada etapa usa la estructura de datos más adecuada, **implementada a mano** (Python puro + NumPy), y su salida alimenta de forma observable a una etapa posterior. Las etapas E6 (filtrado), E7 (similitud) y E8 (grafo) son **fijas** por el enunciado.

## Estado del avance (al 2026-10-25)

Hecho (código, experimentos, informe y demo completos):

- Las 14 estructuras propias en `src/ecostream/structures/` y el módulo de grafos `src/ecostream/graph/`.
- Las 10 etapas (`src/ecostream/stages/s0_…s9_`), el orquestador `pipeline.py` y el panel `dashboard.py`.
- Generador sintético con semilla fija (`data/generator.py`) y cargador del dataset real REES46 (`data/rees46_loader.py`).
- 14 pruebas de corrección en `tests/test_structures.py` (contra dict, heapq, bisect, fuerza bruta y networkx).
- Experimentos 1K–10M (`experiments/bench_*.py`), resultados en `experiments/results/*.csv`, figuras en `experiments/figures/`, tablas LaTeX generadas con `experiments/make_tables.py`.
- Notebooks ejecutados (`notebooks/01_demo_pipeline.ipynb`, `02_resultados_experimentales.ipynb`, regenerables con `build_notebooks.py`).
- Demo `scripts/demo.py` con salida guardada en `docs/demo_output.txt`, `docs/demo_resumen.json`, `docs/dashboard.html`.
- Informe LaTeX en `docs/informe/` compilado como `Proyecto_Final_Grupo3.pdf`; diagrama en `docs/diagrama_pipeline.*`; guion de la demo en `docs/presentacion/guion_demo.md`.

Historial de git (el enunciado evalúa contribuciones por integrante):

- `9a810da` Estructura del repositorio.
- `0132df2` E0–E2 (Integrante 1): FIFO, Merge Sort + búsqueda binaria, heap y generador.
- **Todo lo demás está sin commitear.** Los commits de E3–E5, E6–E7 y E8–E9/orquestador/docs deben hacerlos los integrantes 2, 3 y 4 **desde sus propias cuentas**, siguiendo la sección "Cómo subir el repositorio…" del `README.md`. No commitear esas partes en nombre de otro integrante salvo que se pida explícitamente.

Pendiente:

- Integrantes (en orden de rol 1–4): Frank Laura Borja (E0–E2), Josué Campos Quiroz (E3–E5), Luis Dávila Tello (E6–E7), Mauricio Vallejos Garcia (E8–E9). Ya están en los `.tex` y `.md`, pero **falta recompilar** `docs/informe/main.tex` → `Proyecto_Final_Grupo3.pdf` (no hay LaTeX instalado en la PC; usar Overleaf o MiKTeX).
- Commits de los integrantes 2–4 (ver arriba).
- Diapositivas (15) hechas como artefacto de Claude (https://claude.ai/artifact/NNJsE9QpkGCkT296P8dqxz), ordenadas por expositor (Frank 1–4, Josué 5–7, Luis 8–11, Mauricio 12–15), guion en las notas y en `docs/presentacion/guion_exposicion.md`. Falta exportarlas a .pptx/PDF y guardarlas en `docs/presentacion/`.
- Si se re-ejecutan experimentos, actualizar las cifras del informe para que coincidan con los CSV (`experiments/make_tables.py`, `experiments/results/resumen_cifras.txt`).

## Comandos

Entorno: Windows + PowerShell 5.1, Python 3.11+, venv en `.venv/`. Ejecutar todo desde la raíz.

```powershell
.\.venv\Scripts\Activate.ps1
python -m pytest -q tests                                        # 14 pruebas, ~15 s
python scripts/demo.py | Out-File -Encoding utf8 docs/demo_output.txt   # ~3-4 min
python experiments/bench_base.py --min=1e3 --max=1e3             # también bench_prob, bench_similarity, bench_graph, bench_pipeline
python experiments/plots.py                                      # figuras log-log y pendientes
python notebooks/build_notebooks.py                              # regenera y ejecuta notebooks
```

- Parámetros de la demo: `--events 300000 --products 20000 --seed 42 --batch 5000 --out docs`. Para pruebas usar otro `--out` y no sobrescribir los archivos de la presentación.
- Benchmarks: `--min/--max` en `1e3…1e7`, `--only=...`; los CSV se **acumulan** (solo se reemplazan las filas del tamaño corrido). `bench_pipeline` empieza en 10K. HNSW se omite >100K salvo `$env:HNSW_MAX="1000000"`. Presupuesto por medición: `$env:BENCH_BUDGET` (60 s por defecto). `experiments/run_all.sh` corre todo en bash (~3–4 h).
- En PowerShell 5.1 no usar `tee`/`Tee-Object` (escriben UTF-16); usar `Out-File -Encoding utf8`.

## Arquitectura

| Etapa | Archivo | Estructura | Alimenta a |
|---|---|---|---|
| E0 Ingesta | `s0_ingesta.py` | Cola FIFO con lista enlazada (`linked_queue`), backpressure | micro-lotes → E1 |
| E1 Preprocesamiento | `s1_preprocesamiento.py` | Merge Sort estable + búsqueda binaria (`sorting`) para validar/deduplicar | lote limpio → E2 |
| E2 Prioridades | `s2_prioridades.py` | Min-heap binario (`heap`), 9 clases (alerta > pago > express/premium > … > vista) | → E3 |
| E3 Almacenamiento | `s3_almacenamiento.py` | Hash con sondeo lineal, LRU O(1), matriz dispersa DOK→CSR | → E4, E6 (lista negra exacta), E7 |
| E4 Indexación | `s4_indexacion.py` | B+Tree en disco (páginas 4 KB + buffer pool LRU; archivos `*.idx`, ignorados) | range scan → E8; precios → E9 |
| E5 Multidimensional | `s5_multidimensional.py` | KD-Tree 4D + KNN fuerza bruta como línea base | → E9 |
| E6 Filtrado [FIJA] | `s6_filtrado.py` | Bloom (positivos verificados contra hash exacta) + Count-Min Sketch con ventana | aprobado/bloqueado → E7, E8; alertas → E2 (prioridad 0) |
| E7 Similitud [FIJA] | `s7_similitud.py` | MinHash+LSH (b=32, r=2) y HNSW (32D) | → E9 |
| E8 Grafo [FIJA] | `s8_grafo.py` + `graph/` | Grafo de co-compra, PageRank/PPR, Louvain vs Label Propagation | → E9 |
| E9 Servicio | `s9_servicio.py` | Trie con top-K por nodo, Top-K con heap, panel HTML | salida final |

- Filtrado: clave = `payment_token` y `device_id`; ventana de 5 min (5 buckets de 60 s); umbral > 15 intentos.
- E7 y E8 se recalculan cada `refresh_every` micro-lotes (estilo micro-batch).
- `stages/base.py`: `StageMetrics` (métricas por etapa), `Tracer` (trazas de eventos marcados para la demo), `timed` (context manager de tiempo).
- `experiments/harness.py`: arnés común (semilla 42, reps = min(100, presupuesto/t), memoria con `tracemalloc`).

## Convenciones

- Las estructuras del pipeline son **implementaciones propias**; no reemplazarlas por librerías. `networkx`/`scipy` se usan **solo en tests** como referencia; `dict`, `heapq`, `bisect`, `deque` solo como líneas base en tests/benchmarks.
- Scripts, tests y benchmarks agregan `src/` a `sys.path` (no hay paquete instalado); mantener ese patrón.
- Scripts que imprimen a consola reconfiguran `stdout` a UTF-8 (Windows usa cp1252).
- Código, comentarios, nombres de etapas y documentación en español; reproducibilidad con semilla fija (42).
- Toda afirmación numérica del informe/README debe salir de `experiments/results/` o `docs/demo_resumen.json`.

## Resultados clave (para no recalcular)

- Pipeline con 10M eventos de punta a punta: ~7 674 eventos/s; 348 074 órdenes aprobadas, 1 405 bloqueadas, 200 alertas.
- Bloom: FPR 1.008 % (teórica 1.004 %), 52× menos memoria que `set` (10M). CMS: 104 KB constantes, nunca subestima.
- MinHash+LSH ≈9 200× más rápido que fuerza bruta (10M), recall 1.0 para J ≥ 0.5. HNSW 54× más rápido que KNN exacto (1M), recall@10 = 0.989.
- Louvain vs LPA: NMI 0.999 vs 0.945; modularidad 0.755 vs 0.630.
- Detalle: `docs/resumen_resultados.md`, `experiments/results/resumen_cifras.txt`.
