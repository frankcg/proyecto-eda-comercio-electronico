# Guion de la demo (3-4 minutos, repartida)

La demo no se muestra en un solo bloque: cada expositor muestra, al final de su parte, las secciones de `docs/demo_output.txt` que corresponden a sus etapas. El guion completo de la exposición está en [`guion_exposicion.md`](guion_exposicion.md).

Antes de la clase (tarda ~2.5–4 min; con la semilla 42 los resultados son siempre los mismos):

```powershell
.\.venv\Scripts\Activate.ps1
python scripts/demo.py | Out-File -Encoding utf8 docs/demo_output.txt
```

| Momento | Quién | Qué mostrar | Qué decir |
|---|---|---|---|
| 3:10–4:00 | Frank Laura Borja | Sección 1: **compra express** (líneas E0–E2), **registro sucio** y **duplicado**; sección 2: espera por clase | La compra entra por la FIFO, E1 la valida y verifica por búsqueda binaria que no es duplicada, E2 le da prioridad 2 y sale sin esperar; el precio negativo se rechaza en E1; el duplicado se elimina por Merge Sort + adyacentes. En los picos solo esperan las vistas. |
| 6:30–7:00 | Josué Campos Quiroz | Sección 1, compra express: líneas E3, E4 y E5; final de la sección 4: KD-Tree verificado | Lookup O(1) vía LRU (52 % de aciertos), orden indexada en el B+Tree de páginas de 4 KB, 5 vecinos del KD-Tree, verificado contra fuerza bruta. |
| 10:10–11:20 | Luis Dávila Tello | Sección 1: **pago fraudulento**; sección 3; sección 4 | El CMS estima 18 intentos en 5 min (> 15) y bloquea; el positivo del Bloom se verificó en la tabla hash (383 de 383, 0 bloqueos erróneos). LSH y HNSW dan el mismo resultado que el método exacto con menos trabajo. |
| 13:00–14:10 | Mauricio Vallejos Garcia | Secciones 5 y 6; `docs/dashboard.html` | Grafo construido con un range scan del B+Tree (7 días, solo aprobadas); Louvain Q 0.816 frente a 0.632 de LPA; autocompletado con Trie, recomendaciones Top-K con su fuente, rango de precios en el B+Tree y el panel. |

Plan B: si la ejecución falla, los archivos ya versionados en `docs/` (`demo_output.txt`, `dashboard.html`) son de la misma ejecución.
