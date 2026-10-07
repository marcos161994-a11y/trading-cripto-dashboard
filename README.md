# Dashboard · trading cripto (cuenta de papel)

Sitio estático (`index.html` + `data.json`, Chart.js por CDN) con el estado de la cuenta **de papel** de Kraken
(workspace `global`, inicio $1,000 USD el 2026-10-06, comisión 0.26%) que opera BTCUSD, ETHUSD y SOLUSD cada 2 h.

Render no puede ver la caja ni el MCP, así que el flujo es:

```
revisión (agente en la caja) → MCP Kraken → mcp_snapshot.json → exportar_dashboard.py → data.json + historia_equity.json
→ git commit + push → Render (sitio estático) redepliega solo
```

## Archivos
| Archivo | Qué es |
|---|---|
| `index.html` | El dashboard (español, tema oscuro, apto para celular). |
| `data.json` | Foto actual generada por el script. **No editar a mano.** |
| `historia_equity.json` | Curva de equity persistente: un punto por revisión. **No borrar** (de aquí salen la gráfica y el drawdown máximo). |
| `render.yaml` | Blueprint de Render (static site, publish `.`, auto-deploy en cada commit). |

El script vive fuera del repo: `/workspace/trading-cripto/exportar_dashboard.py`. Solo **lee** `reglas.md`, `posiciones.json`,
`diario.md` y `velas/evaluacion_*.json`; nunca los modifica.

## Procedimiento por revisión (al final de cada chequeo de 2 h, después de operar y actualizar diario/posiciones)

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
   Opciones: `--no-push` (solo commit local), `--snapshot RUTA`, `--repo RUTA`.

## Qué calcula
- Equity = `current_value` del MCP (se compara con efectivo + cripto × último precio; avisa si difieren > $0.50).
- P&L realizado: costo promedio por par a partir de los fills, neto de comisiones de compra y venta. Una "operación" es un ciclo
  completo de 0 → posición → 0 en un par. Win rate, ganancia/pérdida media y factor de beneficio salen de esas operaciones.
- P&L no realizado: valor actual − costo (con comisión de entrada) de lo que queda abierto.
- Stop y objetivo de cada posición vienen de `posiciones.json` (fuente de verdad del stop).
- Drawdown máximo: sobre `historia_equity.json`.
- Freno diario (−3% desde las 00:00 PR) y freno de $900: se muestran como alerta.
- Todas las horas se muestran en hora de Puerto Rico (UTC-4).
