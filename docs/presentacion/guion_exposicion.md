# Guion de la exposición (15 min + 10 de preguntas)

Cada integrante expone **una sola vez**, en este orden, y muestra en la demo **solo la parte de sus etapas**, al final de su bloque. Las diapositivas (15) tienen este mismo texto en las notas del orador.

| # | Expositor | Diapositivas | Tiempo | Incluye demo |
|---|---|---|---|---|
| 1 | Frank Laura Borja | 1–4: portada, contexto y datos, arquitectura, E0–E2 | 0:00–4:00 | 0:50 (secciones 1–2) |
| 2 | Josué Campos Quiroz | 5–7: E3–E4, E5, metodología experimental | 4:00–7:00 | 0:30 (sección 1, líneas E3–E5) |
| 3 | Luis Dávila Tello | 8–11: E6 diseño, E6 resultados, E7 LSH, E7 HNSW | 7:00–11:20 | 1:10 (secciones 3–4) |
| 4 | Mauricio Vallejos Garcia | 12–15: E8, E9 y escala, conclusiones, preguntas | 11:20–15:00 | 1:10 (secciones 5–6 y panel) |

Demo total: ~3:40 min, repartida. Quien termina su bloque dice la frase de enlace y **no vuelve a exponer**.

## Preparación de la demo (antes de la clase)

La demo tarda ~2.5–4 min en ejecutarse (la construcción del HNSW toma ~100 s), así que **se ejecuta antes** y en la clase se muestra su salida. Con la semilla 42 los resultados son siempre los mismos; solo cambian los tiempos.

```powershell
.\.venv\Scripts\Activate.ps1
python -m pytest -q tests                                              # 14 pruebas deben pasar
python scripts/demo.py | Out-File -Encoding utf8 docs/demo_output.txt  # regenera salida, panel y resumen
```

Dejar abiertos antes de empezar:

1. `docs/demo_output.txt` en VS Code (con la búsqueda lista para saltar entre secciones: `1. Recorrido`, `3. Filtrado`, etc.).
2. `docs/dashboard.html` en el navegador (Mauricio).
3. Las diapositivas en modo presentación.

Plan B: si la ejecución falla o no hay tiempo, los archivos ya versionados en `docs/` son de la misma ejecución. Si el profesor pide verlo correr, ejecutar `python scripts/demo.py --out pruebas/vivo` (no sobrescribe los archivos de la presentación) y explicar que la inicialización tarda ~100 s.

---

## 1. Frank Laura Borja (0:00–4:00)

**Diapositiva 1, portada (0:00–0:20).** “Somos el Grupo 3 y presentamos EcoStream, un pipeline de procesamiento de streams para una tienda en línea, el escenario B. Expondremos en orden: yo, el contexto, la arquitectura y E0–E2; Josué, almacenamiento, índices y metodología; Luis, filtrado antifraude y similitud; y Mauricio, el grafo, el servicio y las conclusiones. Cada uno muestra la parte de la demo de sus etapas.”

**Diapositiva 2, contexto y datos (0:20–1:00).** Las cinco necesidades del negocio con baja latencia: priorizar pagos y órdenes urgentes, consultar por clave y por rango, detectar fraude antes de aprobar, recomendar y encontrar comunidades de co-compra. Ninguna estructura resuelve todo: el valor está en orquestarlas. Datos: esquema REES46 (Kaggle) como base; stream sintético con semilla 42, 10M eventos en ~20 s, 20 000 productos en 40 categorías, 2 % duplicados, 0.5 % registros sucios y ráfagas de fraude a propósito.

**Diapositiva 3, arquitectura (1:00–2:00).** Las 10 etapas; todas las estructuras son implementación propia. Flecha continua = camino del evento. Flechas punteadas = una etapa consume la salida de otra: la lista negra exacta de E3 verifica el Bloom de E6; E8 lee del B+Tree de E4 solo las órdenes que E6 aprobó. Flecha roja = las alertas de fraude vuelven al heap de E2 con prioridad 0. Micro-lotes como Spark Structured Streaming; E7 y E8 se recalculan cada 20 lotes.

**Diapositiva 4, E0–E2 (2:00–3:10).**
- E0: cola FIFO con lista enlazada y capacidad máxima (backpressure); O(1). `list.pop(0)` es O(n): pendiente 2.01 frente a 1.08.
- E1: Merge Sort (estable, sin peor caso cuadrático); los duplicados quedan juntos y se eliminan comparando vecinos; entre lotes, búsqueda binaria sobre 300 000 ids recientes. Rechaza precios negativos y normaliza formatos.
- E2: min-heap con 9 clases, clave (prioridad, tiempo) para evitar inanición. Se atienden 5 000 eventos por tick: en los picos esperan las vistas, no los pagos.

**Demo (3:10–4:00), `docs/demo_output.txt`:**
- Sección 1, `compra_express`: líneas E0–E2 (FIFO, única por búsqueda binaria, prioridad 2, espera 0 ticks).
- `registro_sucio`: rechazado en E1 por precio negativo.
- `duplicado`: dos encolados, una copia eliminada por Merge Sort + adyacentes.
- Sección 2: espera por clase; solo `vista/busqueda` con 0.507.

**Enlace:** “El evento ya está limpio y priorizado; Josué explica dónde se guarda y cómo se busca.”

## 2. Josué Campos Quiroz (4:00–7:00)

**Diapositiva 5, E3–E4 (4:00–5:00).**
- E3: tabla hash con sondeo lineal, carga ≤ 0.7, 1.4 sondeos por operación con 10M claves (256 MB frente a 320 MB del `dict`). LRU (hash + lista doble) O(1); con popularidad Zipf acierta el 52 %. Matriz usuario × producto dispersa: densa costaría 8 GB. Alimenta a E6 (lista negra) y E7 (conjuntos de compradores).
- E4: B+Tree en disco real, páginas de 4 KB en un archivo, buffer pool LRU. Fan-out ≈ 200: altura 4 con 10M claves (un AVL tendría ~24 niveles); 1.7 lecturas de disco y 23 µs por búsqueda. Hojas enlazadas: range scan para E8 (7 días de órdenes) y E9 (rangos de precio).

**Diapositiva 6, E5 (5:00–5:40).** Productos como puntos 4D (precio, rating, popularidad, descuento); corte por la mediana de la dimensión más dispersa; poda de ramas. KD-Tree y no R-Tree porque son puntos. Contra KNN fuerza bruta con 10M: 0.19 ms frente a 211 ms (1 100×), 94 nodos visitados, mismo resultado. Límite: en 32D visita el 82 % de los nodos y pierde contra la fuerza bruta → por eso E7 usa HNSW.

**Diapositiva 7, metodología (5:40–6:30).** Tamaños 10³ a 10⁷; repeticiones = min(100, lo que cabe en 60 s) → menos en 1M y 10M, y cada fila del CSV dice cuántas. Media, desviación, memoria actual y pico con tracemalloc (RSS en 10M), plataforma y semilla 42. Log-log y pendiente b de t ∝ nᵇ: FIFO 1.08 frente a `list.pop(0)` 2.01; B+Tree bulk load 1.09; KD-Tree 0.09 (≈ constante).

**Demo (6:30–7:00):** en la sección 1, `compra_express`, señalar E3 (lookup vía LRU, 52 %), E4 (orden indexada, altura 2, 51 páginas de 4 KB) y E5 (5 vecinos). Al final de la sección 4: “KD-Tree verificado contra fuerza bruta: True”.

**Enlace:** “El evento ya está guardado e indexado; Luis explica cómo decidimos si un pago es fraude.”

## 3. Luis Dávila Tello (7:00–11:20)

**Diapositiva 8, E6 diseño (7:00–8:00).**
- Bloom Filter (Bloom, 1970): bits + k hashes; sin falsos negativos, con falsos positivos. 50 000 elementos, p = 0.1 %, 88 KB, k = 10. **Un positivo no bloquea**: se verifica en la lista negra exacta (tabla hash de E3). Demo: 383 positivos, 383 verificados, 0 bloqueos erróneos.
- Count-Min Sketch (Cormode y Muthukrishnan, 2005): matriz d × w; estimación = mínimo de las filas; sobreestima, nunca subestima. Detecta card testing con tarjetas nuevas.
- Definiciones exigidas: clave `payment_token` y `device_id`; ventana deslizante de 5 min (5 sub-sketches de 60 s); umbral > 15 intentos (un cliente normal hace 1–3). Al superarlo: alerta a E2 con prioridad 0, al Bloom y la lista negra, y al panel.

**Diapositiva 9, E6 resultados (8:00–8:40).** Con 10M claves: Bloom 11.4 MB frente a 594 MB del `set` (52×); FPR 1.008 % frente a 1.004 % teórica; 0 falsos negativos. **El Bloom no es más rápido que el `set` (C nativo): su ventaja es la memoria.** CMS: 107 KB fijos frente a 72 MB del `Counter`; nunca subestimó; 100 % dentro de εN; la ventana de 5 min mantiene N pequeño.

**Diapositiva 10, E7 MinHash + LSH (8:40–9:30).** Producto = conjunto de compradores → Jaccard → MinHash+LSH (no SimHash, que es para coseno). P[mismo mínimo] = J. 64 hashes, 32 bandas × 2 filas, umbral ≈ (1/b)^(1/r) ≈ 0.18, porque en co-compra la Jaccard es baja: 16 × 4 pierde el 21 % de los vecinos con J ≥ 0.5 (y en el pipeline recuperaba menos del 10 %). 10M conjuntos: 0.22 ms frente a 2.06 s (≈ 9 200×), recall 1.0 para J ≥ 0.5; costo: índice de 2.4 GB.

**Diapositiva 11, E7 HNSW (9:30–10:10).** Vectores 32D; grafo de mundo pequeño jerárquico (Malkov y Yashunin, 2020), M = 16, ef = 50, heurística de diversidad. Frente a IVF: inserciones sin reentrenar centroides. 1M vectores: 0.84 ms frente a 45 ms del KNN exacto (54×), recall@10 = 0.989; ef ≥ 50 da recall 1.0 en 100K. Límite: construir 1M toma 55 min en Python; 10M (~10 h) no se midió.

**Demo (10:10–11:20):**
- Sección 1, `pago_fraudulento`: prioridad 1, 0 ticks; E6 estima 18 intentos en la ventana > 15 → BLOQUEADO; además su token estaba en la lista negra (positivo del Bloom verificado).
- Sección 3: Bloom 383 positivos / 383 verificados / 0 falsos positivos; 6 alertas de velocidad.
- Sección 4: la licuadora; LSH 1.69 ms frente a 4.83 ms de la fuerza bruta, encuentra los 4 vecinos con J ≥ 0.2; HNSW 10 de 10 frente al KNN exacto.

**Enlace:** “Todo lo que E6 aprueba alimenta el grafo; Mauricio lo explica.”

## 4. Mauricio Vallejos Garcia (11:20–15:00)

**Diapositiva 12, E8 (11:20–12:20).** Nodos = productos; arista ponderada = comprados en la misma orden; solo órdenes aprobadas de 7 días leídas del B+Tree (range scan); se recalcula cada 20 lotes. PageRank (Brin y Page, 1998) por iteración de potencias, ~38 iteraciones, 8.6 s con 8.6M aristas; frente al grado: Spearman 0.985 → el valor está en el **PageRank personalizado** desde el carrito. Louvain (Blondel et al., 2008) frente a Label Propagation: grafo real Q 0.755 frente a 0.630 y 221 frente a 848 comunidades; 10M: NMI 0.999 frente a 0.945; LPA 2× más rápido (78 s frente a 152 s).

**Diapositiva 13, E9 y escala (12:20–13:00).** Trie con top-K por nodo: 1.8 µs frente a 963 ms del escaneo lineal (1M nombres). Recomendación = 0.45 PPR + 0.30 LSH + 0.15 HNSW + 0.03 KD + 0.07 PageRank; excluye lo comprado; Top-K con min-heap de tamaño K; cada recomendación dice su fuente. Panel HTML. Escala: 10M eventos de punta a punta a 7 674 eventos/s; 348 074 aprobadas, 1 405 bloqueadas, 200 alertas.

**Demo (13:00–14:10):**
- Sección 5: ventana de 7 días (2 432 órdenes aprobadas, 5 047 aristas); 8/10 en común entre top PageRank y top grado; Louvain Q = 0.816 (190 comunidades) frente a LPA Q = 0.632 (496); la mayor comunidad junta 91 laptops, 43 mouse, 42 teclados y 26 monitores.
- Sección 6: autocompletar `smart`, `lap`, `zapa`; usuario 0 (compró jean y casacas) recibe casacas y jeans con fuente `[LSH+PPR]`; laptops de S/ 2000–3000 por range scan del B+Tree.
- Abrir `docs/dashboard.html`: ventas, alertas y métricas por etapa.

**Diapositiva 14, conclusiones (14:10–14:50).** Integración verificable; complejidades confirmadas; probabilístico = memoria con verificación; aproximado = velocidad con parámetros ajustados al dominio; Louvain sobre LPA. Límites y trabajo futuro: HNSW en C/hnswlib, Leiden, Cuckoo filters, CSV completo de REES46.

**Diapositiva 15 (14:50–15:00).** “Gracias, quedamos atentos a sus preguntas.”

---

## Preguntas probables (todos deben poder responderlas)

| Pregunta | Respuesta corta |
|---|---|
| ¿Por qué Merge Sort y no Quicksort? | Estable (mantiene el orden de llegada entre iguales) y O(n log n) garantizado; Quicksort tiene peor caso O(n²). |
| ¿Por qué un heap y no una lista ordenada? | Insertar en lista ordenada es O(n); en el heap, O(log n). Medido: pendiente 1.60 de `insort` frente a 1.15 del heap. |
| ¿Cómo evitan la inanición en el heap? | La clave es (prioridad, tiempo): dentro de una clase sale primero el más antiguo; las vistas esperan solo ~0.5 ticks. |
| ¿Por qué sondeo lineal? | Buena localidad de caché y simple; con carga ≤ 0.7 hace ~1.4 sondeos por operación. |
| ¿Por qué un B+Tree y no un árbol binario? | Fan-out ≈ 200 → altura 4 con 10M claves (pocas lecturas de disco); hojas enlazadas → range scans. |
| ¿Por qué KD-Tree en E5 y HNSW en E7? | KD-Tree es exacto y excelente en 4D; en 32D visita el 82 % de los nodos (maldición de la dimensionalidad). |
| ¿Qué pasa con un falso positivo del Bloom? | No se bloquea: se verifica en la tabla hash exacta; si no está, se cuenta como falso positivo y la orden sigue. |
| ¿Por qué el Bloom no es más rápido que el set? | El `set` es C nativo; ambos son O(1) por operación. La ventaja del Bloom es memoria (52×). |
| ¿Clave, ventana y umbral del CMS? | `payment_token` y `device_id`; 5 min deslizantes en 5 buckets de 60 s; > 15 intentos. |
| ¿Por qué el CMS nunca subestima? | Cada contador acumula la clave más las que colisionan; el mínimo de las filas es ≥ la frecuencia real. |
| ¿Por qué b = 32, r = 2 en LSH? | Umbral ≈ 0.18, adecuado a la Jaccard baja de co-compra; 16 × 4 (umbral 0.5) perdía vecinos reales. |
| ¿Por qué MinHash y no SimHash? | La señal es un conjunto (compradores) → Jaccard; SimHash aproxima coseno sobre vectores ponderados. |
| ¿Por qué HNSW y no IVF? | El catálogo crece con inserciones; IVF requiere reentrenar centroides. |
| ¿Por qué no hay HNSW con 10M? | Construir 1M toma 55 min en Python puro; 10M serían ~10 h. Está justificado en el informe, como permite el enunciado. |
| ¿PageRank aporta algo sobre el grado? | Poco en el global (Spearman 0.985); el valor está en el PageRank personalizado desde el carrito, que el grado no da. |
| ¿Por qué Louvain y no Label Propagation? | Mayor modularidad en el grafo real (0.755 frente a 0.630), menos fragmentación y más estable en el sintético; LPA es 2× más rápido. |
| ¿Cuántas repeticiones y por qué menos en 10M? | Hasta 100; reps = min(100, 60 s / t). En 10M una corrida dura minutos en Python con 2 vCPU; cada CSV indica sus repeticiones. |
| ¿Cómo midieron la memoria? | tracemalloc (actual y pico); en 10M, RSS del proceso, porque tracemalloc multiplica el tiempo 3–5×. |
| ¿Usaron datos reales? | Esquema REES46 como base y stream sintético (como recomienda el enunciado); hay cargador para el CSV real (`rees46_loader.py`). |
