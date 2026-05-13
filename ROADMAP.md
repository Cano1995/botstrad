# Roadmap de desarrollo — General Ares
**Estado actual:** v1 (análisis completo, sin correcciones)  
**Objetivo:** Bot rentable con lógica correcta, señales filtradas y gestión de riesgo sólida

---

## FASE 1 — Corrección de bugs críticos
> Prioridad: **BLOQUEANTE**. Son errores que rompen la lógica del bot aunque parezca funcionar.
> Versión objetivo: `general_ares_v2.txt`

| # | Tarea | Archivo/Función | Descripción |
|---|---|---|---|
| 1.1 | Fix cierre BUY inalcanzable | `CheckFloatingProfit()` | Cambiar `current_rsi >= 8000` por un valor real (ej. `>= 70`) para que las posiciones BUY puedan cerrarse en la zona ±PuntosCierre |
| 1.2 | Fix cierres SELL con comentarios de COMPRA | `CheckFloatingProfit()` | Los `StringFind(comentario, "COMPRA2/3")` dentro del bloque SELL deben buscar `"VENTA2/3"` según corresponda |
| 1.3 | Fix pausa en OnTradeTransaction | `OnTradeTransaction()` | La pausa de 900s debe activarse solo si `deal_profit < 0`, no en todo cierre |
| 1.4 | Fix trailing retrocede en 950–1050 pts | `CheckFloatingProfit()` | Corregir `price_open + 550 * _Point` por `price_open + 750 * _Point` en ese rango |

**Criterio de aceptación:** El bot abre y cierra posiciones BUY y SELL de forma simétrica y correcta. La pausa post-ganancia ya no existe.

---

## FASE 2 — Gestión de riesgo base
> Prioridad: **ALTA**. Sin esto el bot es vulnerable ante slippage, desconexiones y mercados volátiles.  
> Versión objetivo: `general_ares_v3.txt`

| # | Tarea | Archivo/Función | Descripción |
|---|---|---|---|
| 2.1 | SL físico en apertura | `OpenBuy()` / `OpenSell()` | Pasar un SL real en la llamada a `trade.Buy/Sell` (ej. 150–200 puntos desde entrada). Hoy es `0`. |
| 2.2 | Verificación de spread máximo | `OnTick()` antes de señales | Antes de cualquier entrada: `if (spread > max_spread) return;`. Agregar `input int MaxSpread = 30;` |
| 2.3 | Rellenar gaps del trailing stop | `CheckFloatingProfit()` | Cubrir los rangos sin protección: 200–249 pts, 1550–1999 pts, y 3000+ pts |
| 2.4 | Convertir `apagar` en input | Variables globales | `input bool Apagar_Bot = false;` — permite frenar el bot desde el panel sin recompilarlo |
| 2.5 | Eliminar pausa al mover SL | `CheckFloatingProfit()` | Quitar `pausa_hasta = TimeCurrent() + 300` tras `PositionModify`. Innecesario con posición activa. |

**Criterio de aceptación:** Toda posición abierta tiene SL físico desde el momento 0. El bot no opera si el spread es excesivo. El trailing no tiene zonas sin cobertura.

---

## FASE 3 — Calidad de señales
> Prioridad: **ALTA**. Es la razón principal por la que el bot genera señales falsas.  
> Versión objetivo: `general_ares_v4.txt`

| # | Tarea | Archivo/Función | Descripción |
|---|---|---|---|
| 3.1 | Activar filtro de tendencia MA | `OnTick()` — condiciones de entrada | Aplicar `tendencia_alcista` y `tendencia_bajista` (ya calculadas) como condición obligatoria en cada señal de compra/venta |
| 3.2 | Implementar confirmación en dos ticks | `OnTick()` — flags `compra_ready` | El flag se activa en el tick N. La ejecución ocurre solo si el flag sigue activo en el tick N+1 con condiciones vigentes. Hoy se activa y ejecuta en el mismo tick. |
| 3.3 | Evaluar períodos RSI y ADX | `OnInit()` — handles | Testear RSI(7) y ADX(14) como alternativa a RSI(2)/ADX(5). Medir tasa de señales falsas antes y después. |
| 3.4 | Corregir asimetría compra/venta | `OnTick()` — condiciones | Revisar que los umbrales de VENTA sean equivalentes (en extremidad) a los de COMPRA, o documentar la razón del sesgo. |
| 3.5 | Limpiar señales duplicadas o redundantes | `OnTick()` | VENTA3 y VENTA5 tienen condiciones solapadas (ambas RSI>=90). Definir si son necesarias o fusionarlas. |

**Criterio de aceptación:** Reducción medible de operaciones abiertas en contra de tendencia. El bot no abre dos veces en el mismo tick por la misma señal.

---

## FASE 4 — Arquitectura y limpieza de código
> Prioridad: **MEDIA**. No afecta rentabilidad directamente pero es necesario para mantener el bot a largo plazo.  
> Versión objetivo: `general_ares_v5.txt`

| # | Tarea | Archivo/Función | Descripción |
|---|---|---|---|
| 4.1 | Eliminar variables y flags sin uso | Global | Eliminar `compra_ready`, `compra_ready2`, `venta_ready`, `venta_ready1`, `venta_ready3`, `autorizado`. O implementarlos correctamente en Fase 3. |
| 4.2 | Eliminar bloque de verificación vacío | `OnTick()` | El bloque `if (TimeCurrent() - ultima_verificacion > ...)` no hace nada. Completarlo o eliminarlo. |
| 4.3 | Convertir constantes en inputs | Variables globales | `RSI_Period`, `ADX_Period`, `Lots`, `PuntosCierre`, `MaxSpread` deben ser `input` para poder ajustarlos sin recompilar. |
| 4.4 | Reactivar y corregir integración API | `OnTick()` — bloque comentado | Descomentar y testear el bloque de `ConsultarAPI`. Validar que el JSON parser funcione correctamente. |
| 4.5 | Centralizar lógica de cierre | Nueva función `CheckCloseConditions()` | Extraer la lógica de cierre condicional de `CheckFloatingProfit()` a una función separada. Mejora legibilidad. |
| 4.6 | Corregir `PuntosCierre` o eliminarlo | `CheckFloatingProfit()` | El cierre en +5000 pts siempre ocurre antes que el de ±20000. Sincronizar ambos valores o eliminar la redundancia. |

**Criterio de aceptación:** El código no tiene variables muertas ni bloques vacíos. Todos los parámetros clave son configurables desde el panel de MT5.

---

## FASE 5 — Backtesting y optimización
> Prioridad: **ALTA** (en paralelo con Fases 3 y 4 idealmente).  
> Herramienta: MT5 Strategy Tester

| # | Tarea | Descripción |
|---|---|---|
| 5.1 | Backtest v1 (baseline) | Correr el bot original para tener una línea base de performance (drawdown, win rate, profit factor) |
| 5.2 | Backtest v2 (bugs corregidos) | Medir el impacto puro de los fixes de Fase 1+2 |
| 5.3 | Backtest v3 (señales mejoradas) | Medir el impacto de Fase 3 — especialmente el filtro MA y la confirmación en dos ticks |
| 5.4 | Optimización de parámetros | Usar el optimizador de MT5 para: `RSI_Period`, `ADX_Period`, `ADX_Level`, `SL_Inicial`, `MaxSpread` |
| 5.5 | Forward test en demo | Correr la versión optimizada en cuenta demo mínimo 2 semanas antes de producción |

**Criterio de aceptación:** Profit Factor > 1.3, Drawdown máximo < 15%, Win Rate > 45% en período de 6 meses histórico.

---

## FASE 6 — Features avanzados (post-estabilización)
> Solo iniciar cuando Fases 1–5 estén completas y el bot sea rentable en demo.

| # | Feature | Descripción |
|---|---|---|
| 6.1 | Gestión dinámica de lote | Ajustar `Lots` según % del capital (Kelly fraction o fixed fraction) en lugar de lote fijo |
| 6.2 | Filtro horario | No operar en horarios de bajo volumen o alta volatilidad no estructurada (ej. apertura NY, noticias) |
| 6.3 | Multi-timeframe | Confirmar señal M1 con tendencia en M5 o M15 antes de entrar |
| 6.4 | Dashboard en pantalla | Panel visual con estado del bot, PnL del día, trades abiertos, pausa activa |
| 6.5 | Integración API completa | Señales externas desde el sistema Oracle Cloud ya construido |

---

## Resumen de versiones

| Versión | Fase | Estado |
|---|---|---|
| `general_ares_v1.txt` | Original analizado | ✅ Guardado |
| `general_ares_v2.txt` | Fase 1 — Bugs críticos corregidos | ✅ Completo |
| `general_ares_v3.txt` | Fase 2 — Riesgo base | ⏳ Pendiente |
| `general_ares_v4.txt` | Fase 3 — Calidad de señales | ⏳ Pendiente |
| `general_ares_v5.txt` | Fase 4 — Arquitectura limpia | ⏳ Pendiente |
| `general_ares_v6.txt` | Fase 5 — Versión optimizada | ⏳ Pendiente |

---

*Plan generado con Claude Code — 2026-05-13*
