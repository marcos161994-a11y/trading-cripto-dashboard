# Dashboard · trading cripto (cuenta de papel)

Sitio estático (`index.html` + `data.json`, Chart.js por CDN) con el estado de **dos cuentas de papel** de Kraken (comisión 0.26%, BTCUSD/ETHUSD/SOLUSD):

- **Estrategia 1**: workspace `global`, inicio $1,000 el 2026-10-06 22:12 PR, intradía, revisión cada 2 h.
  Reglas: `/workspace/trading-cripto/reglas.md`.
- **Estrategia 2**: workspace `tendencia`, inicio $1,000 el 2026-10-07 00:09 PR, tendencia diaria SMA200, revisión diaria ~20:15 PR.
  Reglas: `/workspace/trading-cripto/estrategia2/reglas.md`.

Arriba hay una comparación de las dos estrategias y de comprar y mantener BTC: $1,000 comprados al inicio de E1 a $83,894, comisión incluida.
Debajo, pestañas con el detalle de cada estrategia (`#e1` / `#e2` en la URL).

Render no puede ver la caja ni el MCP, así que el flujo es:

```
E1: revisión 2 h → MCP Kraken (global) → mcp_snapshot.json ┐
E2: el script lee 'tendencia' con el CLI (solo lectura) ──────┴→ exportar_dashboard.py → data.json + historia_*.json
→ git commit + push → Render (sitio estático) redepliega solo
```

## Archivos
| Archivo | Qué es |
|---|---|
| `index.html` | El dashboard (español, tema oscuro, apto para celular). |
| `data.json` | Foto actual generada por el script. **No editar a mano.** La raíz es la estrategia 1 (mismo formato de siempre). `estrategia2` tiene el mismo formato para `tendencia`, y `comparacion` trae las series E1, E2 y B&H BTC. |
| `historia_equity.json` | Curva de equity de E1 (`global`): un punto por revisión. **No borrar** (de aquí salen la gráfica y el drawdown máximo). |
| `historia_tendencia.json` | Lo mismo para E2 (`tendencia`). Cada punto lleva también el precio de BTC (`btc`), que se usa para la curva B&H. **No borrar.** |
| `render.yaml` | Blueprint de Render (static site, publish `.`, auto-deploy en cada commit). |

El script vive fuera del repo: `/workspace/trading-cripto/exportar_dashboard.py`. Solo **lee** `reglas.md`, `posiciones.json`,
`diario.md` y `velas/evaluacion_*.json` de cada estrategia; nunca los modifica.
Para E2 ejecuta comandos de **solo lectura** del CLI, siempre con `--workspace tendencia`:
`kraken paper status|balance|history|orders --workspace tendencia -o json` y `kraken ticker`.
Guarda esa foto en `/workspace/trading-cripto/estrategia2/snapshot_tendencia.json`.
Si eso falla, conserva el bloque anterior de E2 y la exportación de E1 sigue normal.

## Procedimiento por revisión de la ESTRATEGIA 1 (al final de cada chequeo de 2 h, después de operar y actualizar diario/posiciones)

Es igual que antes. El mismo comando ahora refresca **las dos** estrategias: E2 se lee sola con el CLI, y no hay que hacer llamadas extra al MCP para ella.

1. Hacer estas 5 llamadas al MCP (namespace `user-Kraken`, con `CallDynamicTool`):
   - `kraken_paper_status` → `{"name": "global"}`
   - `kraken_paper_balance` → `{"name": "global"}`
   - `kraken_paper_history` → `{}`
   - `kraken_paper_orders` → `{}`
   - `kraken_ticker` → `{"pairs": ["BTCUSD", "ETHUSD", "SOLUSD"]}`
2. Guardar las respuestas **crudas, sin modificar** en `/workspace/trading-cripto/mcp_snapshot.json` con esta forma:
   ```json
   {"status": <respuesta status>, "balance": <respuesta balance>, "history": <respuesta history>,
    "orders": <respuesta orders>, "ticker": <respuesta ticker>}
   ```
3. Ejecutar:
   ```bash
   python3 /workspace/trading-cripto/exportar_dashboard.py
   ```
   El script rechaza un snapshot con más de 30 min de antigüedad, añade un punto a `historia_equity.json`
   (si se repite en menos de 10 min reemplaza el último punto), escribe `data.json` y hace commit + `git push origin HEAD` **solo si algo cambió**.
   Sale con código 2 si el push falla (el commit queda local y se sube en la próxima corrida).
   Opciones: `--no-push` (solo commit local), `--snapshot RUTA`, `--repo RUTA`, `--sin-tendencia` (no refresca E2).
   El script rechaza un `mcp_snapshot.json` cuyo `status.workspace` no sea `global`.

## Procedimiento de la ESTRATEGIA 2 (revisión diaria)
Después de `python3 /workspace/trading-cripto/estrategia2/ejecutar_diario.py --ejecutar`:
```bash
python3 /workspace/trading-cripto/exportar_dashboard.py --solo-tendencia
```
Refresca solo E2, la comparación y su punto en `historia_tendencia.json`. No necesita `mcp_snapshot.json` fresco y
deja el bloque de E1 como estaba en su última revisión de 2 h.

## Qué calcula
- Equity = `current_value` del MCP (se compara con efectivo + cripto × último precio; avisa si difieren > $0.50).
- P&L realizado: costo promedio por par a partir de los fills, neto de comisiones de compra y venta. Una "operación" es un ciclo
  completo de 0 → posición → 0 en un par. Win rate, ganancia/pérdida media y factor de beneficio salen de esas operaciones.
- P&L no realizado: valor actual − costo (con comisión de entrada) de lo que queda abierto.
- Stop y objetivo de cada posición vienen de `posiciones.json` (fuente de verdad del stop).
- Drawdown máximo: sobre `historia_equity.json`.
- E1: freno diario (−3% desde las 00:00 PR) y freno de $900; se muestran como alerta.
- E2: freno de $750 (`freno_total_activo`). E2 no tiene stop, objetivo ni freno diario; la tabla de posiciones muestra el peso actual/objetivo y la condición de salida.
- Todas las horas se muestran en hora de Puerto Rico (UTC-4).
