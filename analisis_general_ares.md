# Análisis — Expert Advisor "General Ares"
**Autor del bot:** Cristhian Cano  
**Fecha de análisis:** 2026-05-13  
**Versión analizada:** v1 (código inicial)

---

## Resumen ejecutivo

El bot opera con RSI(2) y ADX(5) en temporalidad M1. Tiene múltiples señales de entrada (COMPRA2, COMPRA3, VENTA3, VENTA4, VENTA5), un sistema de trailing stop escalonado y una infraestructura para señales externas vía API (actualmente comentada). Presenta bugs críticos que anulan condiciones de cierre enteras y decisiones de diseño que reducen drásticamente su rentabilidad.

---

## Puntos Fuertes

| # | Descripción |
|---|---|
| 1 | **Trailing stop escalonado** — concepto correcto para proteger ganancias progresivamente |
| 2 | **Pausa post-pérdida** — frena el bot tras una pérdida para evitar revenge trading |
| 3 | **Comentario en posición** — registra el tipo de señal en el comment para trazabilidad |
| 4 | **`OnTradeTransaction`** — usa el callback correcto de MT5 en lugar de polling en OnTick |
| 5 | **Stop duro en -500 puntos** — limita la pérdida máxima por operación |
| 6 | **Infraestructura API** — arquitectura preparada para recibir señales externas |

---

## Bugs Críticos (código roto)

### BUG 1 — Condición de cierre de BUY es inalcanzable
```mql5
if (current_rsi >= 8000) {  // RSI nunca llega a 8000. Máximo: 100.
    trade.PositionClose(ticket);
```
**Impacto:** Las posiciones BUY en la zona `±PuntosCierre` **nunca se cierran** por esta condición. Es código muerto.  
**Fix:** Cambiar a un valor real, ej. `current_rsi >= 70`.

---

### BUG 2 — El trailing stop retrocede entre 950 y 1050 puntos
```mql5
// 850–950 pts  → SL en price_open + 650
// 950–1050 pts → SL en price_open + 550  ← retrocede 100 puntos
```
**Impacto:** La lógica intenta mover el SL hacia atrás. La guarda `nuevo_sl > sl_actual + 10` lo bloquea, pero la lógica es incorrecta y dejará de proteger si se llega a ese rango desde abajo.  
**Fix:** Cambiar `550` por `650` en ese bloque (o `750` para mantener la progresión).

---

### BUG 3 — Cierres de SELL referencian comentarios de COMPRA
```mql5
// Dentro del bloque SELL (is_sell = true):
if (current_rsi >= 80 && StringFind(comentario, "COMPRA2") >= 0) { ... }
if (current_rsi >= 80 && StringFind(comentario, "COMPRA3") >= 0) { ... }
if (current_adx >= 90 && StringFind(comentario, "COMPRA3") >= 0) { ... }
```
**Impacto:** Una posición SELL nunca tendrá `"COMPRA2"` o `"COMPRA3"` en su comentario. Estas tres condiciones de cierre son **código muerto**. Las posiciones VENTA3, VENTA4, VENTA5 no tienen salida funcional vía condición RSI/ADX.  
**Fix:** Cambiar los `StringFind` para que busquen los comentarios correctos (`"VENTA4"`, `"VENTA3"`, `"VENTA5"`).

---

### BUG 4 — La pausa de 15 min se activa tras CUALQUIER cierre, incluyendo ganancias
```mql5
if(deal_reason == codigo_sl_posible_1 || deal_reason == codigo_sl_posible_2) {
    pausa_hasta = TimeCurrent() + 900;
    return;
}
// Sin else ni return previo — esto corre siempre:
pausa_hasta = TimeCurrent() + 900;
PrintFormat("⏸️ Cierre con érdida detectado...");
return;
```
**Impacto:** Después de cada operación ganadora, el bot se pausa 15 minutos. Reduce entre 50–80% de las oportunidades.  
**Fix:** Envolver el segundo bloque en `if(deal_profit < 0)`.

---

### BUG 5 — Filtro de tendencia MA calculado pero nunca aplicado
```mql5
bool tendencia_alcista = precio_actual > ma_value[0];
bool tendencia_bajista = precio_actual < ma_value[0];
// Estas variables NO aparecen en ninguna condición de entrada
```
**Impacto:** El filtro de tendencia es completamente inefectivo. El bot entra indiscriminadamente contra la tendencia.  
**Fix:** Agregar `&& tendencia_alcista` en las condiciones de compra y `&& tendencia_bajista` en las de venta.

---

### BUG 6 — `compra_ready` se activa pero nunca se verifica antes de operar
```mql5
compra_ready3 = "S";
// En el mismo tick, sin verificar compra_ready3:
if (... && !buy_opened) {
    OpenBuy("COMPRA3");  // abre igual, con o sin ready
}
```
**Impacto:** El sistema de confirmación en dos pasos nunca funciona. Las entradas se ejecutan en el mismo tick que la señal, sin filtro adicional.  
**Fix:** Separar la detección de la ejecución: activar el flag en el tick actual y ejecutar solo si el flag ya estaba activo del tick anterior.

---

## Puntos Débiles (diseño)

### D1 — RSI(2) y ADX(5) en M1 son generadores de ruido
RSI de período 2 alcanza extremos (`< 6` o `> 90`) decenas de veces por hora en M1. ADX(5) es igualmente inestable. El resultado es señales de alta frecuencia basadas en ruido estadístico, no en tendencia real. Cada señal falsa paga spread.

### D2 — Sin Stop Loss físico en la entrada
```mql5
trade.Buy(Lots, _Symbol, ask, 0, 0, comentarioc);  // SL = 0
```
La posición abre sin SL físico. Si hay un salto de precio brusco o el servidor cae, la posición queda desprotegida hasta el próximo tick de `CheckFloatingProfit()`.

### D3 — Sin verificación de spread antes de operar
En scalping M1, un spread de 20–50 puntos consume directamente la ganancia esperada. No existe guarda del tipo:
```mql5
double spread = SymbolInfoInteger(_Symbol, SYMBOL_SPREAD);
if (spread > max_spread_permitido) return;
```

### D4 — Condiciones de venta asimétricas respecto a compra
- Compra: RSI ≤ 6 (COMPRA3) o RSI ≤ 10 (COMPRA2)
- Venta: RSI ≥ 85 (VENTA4) o RSI ≥ 90 (VENTA3, VENTA5)

Las ventas requieren condiciones más extremas → más compras que ventas en el tiempo. Si el instrumento tiene sesgo bajista, el bot pierde sistemáticamente.

### D5 — `PuntosCierre = 20000` es irrelevante en la práctica
La condición de ganancia en +5000 puntos se ejecuta primero. El bloque `±PuntosCierre` con 20000 nunca se alcanza en condiciones normales (y para BUY está roto por BUG 1 de todas formas).

### D6 — `apagar` hardcodeado, no configurable desde MT5
```mql5
string apagar = "N";  // Nunca cambia en runtime
```
El kill switch no tiene forma de activarse desde el panel de inputs. Debería ser:
```mql5
input bool Apagar_Bot = false;
```

### D7 — Pausa post-trailing innecesaria para nuevas entradas
```mql5
if (trade.PositionModify(ticket, nuevo_sl, 0)) {
    pausa_hasta = TimeCurrent() + 300;
```
Cada avance del trailing pausa el bot 5 minutos. Innecesario si ya hay una posición abierta.

### D8 — Gaps en la cobertura del trailing stop

| Rango de puntos | Estado |
|---|---|
| 0 – 109 | Sin SL |
| 200 – 249 | Sin movimiento de SL |
| 1550 – 1999 | Sin movimiento de SL |
| 3000+ | Sin cobertura — posición en riesgo libre |

### D9 — Bloque de verificación periódica vacío
```mql5
if (TimeCurrent() - ultima_verificacion > intervalo_verificacion) {
    ultima_verificacion = TimeCurrent();
    // vacío — no hace nada
}
```

### D10 — Gestión solo de una posición simultánea
`PositionSelect(_Symbol)` solo maneja una posición por símbolo. Sin posibilidad de piramidado ni gestión multiposición.

---

## Por qué no es rentable — diagnóstico

| Causa | Impacto estimado |
|---|---|
| Pausa 15 min tras TODO cierre (incluye ganancias) | Elimina 50–80% de oportunidades de entrada |
| Filtro MA activo ignorado | Entradas contra tendencia principal |
| RSI(2)/ADX(5) en M1 | Alta tasa de señales falsas; spread consume ganancias |
| Sin SL físico en entrada | Riesgo ilimitado hasta el próximo tick |
| Cierre BUY por RSI ≥ 8000 (imposible) | Posiciones BUY sin lógica de salida funcional |
| Cierres SELL con comentarios de COMPRA | Posiciones SELL sin condiciones de salida funcionales |
| Trailing retrocede en 950–1050 pts | Lógica de protección incorrecta en ese rango |
| compra_ready nunca verificado | Confirmación de dos pasos no implementada |

---

## Plan de correcciones — prioridad

### Alta (bugs que rompen la lógica)
1. Corregir cierre BUY: `>= 8000` → `>= 70` (u otro nivel real)
2. Corregir `StringFind` en cierres de SELL para que usen comentarios de VENTA
3. Corregir `OnTradeTransaction`: pausa solo si `deal_profit < 0`
4. Corregir trailing en 950–1050 pts: `550` → `650`

### Media (diseño)
5. Activar filtro MA en condiciones de entrada
6. Agregar SL físico en `OpenBuy` y `OpenSell` (ej. 150–200 puntos)
7. Agregar verificación de spread máximo antes de entrar
8. Implementar `compra_ready` como confirmación real (tick anterior → tick actual)
9. Convertir `apagar` en `input bool`
10. Rellenar gaps en el trailing stop (200–249, 1550–1999, 3000+)

### Baja (optimización)
11. Considerar RSI(7–14) y ADX(14) para reducir ruido, o subir a M5
12. Hacer `PuntosCierre` coherente con la lógica real de cierre
13. Eliminar código comentado o variables sin uso (`compra_ready2`, bloque verificación vacío)

---

*Análisis generado con Claude Code — versión bot: v1*
