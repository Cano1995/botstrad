# Configuración para XAUUSD (Oro) — General Ares v5
**Fecha:** 2026-05-13  
**Base:** _Point = 0.01 | Contrato estándar = 100 oz | 0.01 lotes = 1 oz

---

## Por qué el oro necesita parámetros distintos

| Característica | Forex (EURUSD) | Oro (XAUUSD) | Impacto en el bot |
|---|---|---|---|
| Spread típico | 5–15 pts | 25–50 pts | MaxSpread=30 bloquearía casi todo |
| Movimiento por vela M1 | 20–80 pts | 50–200 pts | SL_Inicial=150 se toca por ruido |
| Rango diario | 500–1500 pts | 3000–8000 pts | TP y PuntosCierre deben escalar |
| Tendencia | Oscilación frecuente | Tendencias largas y fuertes | Filtro MA más importante aún |
| RSI(2) extremos | Muy frecuentes | Menos frecuentes (más ruido espurio) | Señales más selectivas |

---

## Parámetros recomendados para XAUUSD

```
//=== INPUTS SUGERIDOS PARA ORO ===//
RSI_Period      = 2        // mantener para inicio, probar 5 en backtest
ADX_Period      = 5        // mantener, probar 10
MA_Period_Inp   = 20       // MA(5) demasiado corta para oro — subir a 20
Lots            = 0.01     // 1 oz — OK para prueba inicial
SL_Inicial      = 250      // $2.50 por trade — oro mueve $1-2 por vela M1
TP_Objetivo     = 3000     // $30 target — alcanzable en tendencias de oro
SL_Maximo       = 600      // $6 máx pérdida por trade — adaptado a volatilidad
MaxSpread       = 50       // spread de oro suele ser 25-45 pts
Pausa_Perdida   = 900      // mantener 15 min
Pausa_SL        = 900      // mantener 15 min
Apagar_Bot      = false
Usar_Filtro_MA  = true     // CRÍTICO para oro — filtra entradas contra tendencia
```

---

## Por qué cada cambio

### MA_Period_Inp: 5 → 20
El oro tiene tendencias sostenidas de horas. MA(5) en M1 cambia de dirección cada 5 velas — demasiado ruidoso para determinar tendencia. MA(20) da una dirección más estable.

### SL_Inicial: 150 → 250
El movimiento promedio por vela M1 en oro es $0.50–$2.00. Con SL_Inicial=150 pts ($1.50), un movimiento normal de mercado toca el SL antes de que la señal RSI se resuelva. 250 pts ($2.50) da más margen sin ser temerario.

### TP_Objetivo: 5000 → 3000
5000 pts = $50 de movimiento en oro. Alcanzable pero poco frecuente en M1 scalping. 3000 pts = $30 es más realista y captura más trades.

### SL_Maximo: 500 → 600
Adaptado a la mayor volatilidad del oro. Permite al precio "respirar" sin cerrar por ruido.

### MaxSpread: 30 → 50
Gold spread típico: $0.25–$0.45 = 25–45 pts. Con MaxSpread=30 el bot no abriría casi nada. 50 pts cubre el spread normal sin operar en spreads de noticias.

---

## Valores de referencia — qué significa cada punto en XAUUSD

| Puntos | USD (0.01 lotes = 1 oz) | USD (0.10 lotes = 10 oz) |
|---|---|---|
| 150 pts | $1.50 | $15.00 |
| 250 pts (SL) | $2.50 | $25.00 |
| 600 pts (SL máx) | $6.00 | $60.00 |
| 3000 pts (TP) | $30.00 | $300.00 |
| 5000 pts | $50.00 | $500.00 |

---

## Señales que más van a disparar en oro

**COMPRA3** (RSI<=6, ADX 30-60): ocurre cuando el oro tiene una caída rápida dentro de una tendencia con momentum moderado. **La más frecuente y relevante para oro.**

**VENTA4** (RSI>=80, ADX<=30): ocurre cuando el precio rebotó en zona de sobrecompra pero sin tendencia fuerte. Útil en consolidaciones.

**COMPRA2** (RSI<=10, ADX 26-35): similar a COMPRA3 pero más extremo. En oro puede tardar más en darse.

**VENTA3/VENTA5** (RSI>=90, ADX>=65/90): en oro estos extremos son raros — cuando se dan, suelen ser reversiones reales.

---

## Plan de backtest para oro

### Paso 1 — Baseline con parámetros ajustados
- Símbolo: XAUUSD
- Período: M1
- Rango histórico: 6 meses mínimo (ej. Oct 2024 – May 2025)
- Parámetros: los de esta config
- Métrica objetivo: Profit Factor > 1.3, Drawdown < 15%

### Paso 2 — Optimización del MA
| Variable | Mínimo | Máximo | Paso |
|---|---|---|---|
| MA_Period_Inp | 10 | 50 | 5 |

### Paso 3 — Optimización RSI/ADX
| Variable | Mínimo | Máximo | Paso |
|---|---|---|---|
| RSI_Period | 2 | 10 | 1 |
| ADX_Period | 5 | 20 | 5 |

### Paso 4 — Optimización SL/TP
| Variable | Mínimo | Máximo | Paso |
|---|---|---|---|
| SL_Inicial | 150 | 400 | 50 |
| TP_Objetivo | 1500 | 5000 | 500 |
| SL_Maximo | 400 | 1000 | 100 |

### Paso 5 — Forward test en demo
- Mínimo 2 semanas en XAUUSD demo antes de ir a real
- Monitorear spread real del broker durante las horas activas (London + NY)

---

## Horarios de oro recomendados

El oro tiene volumen concentrado en dos ventanas:

| Sesión | Hora Paraguay (UTC-4) | Características |
|---|---|---|
| Londres | 05:00 – 13:00 | Tendencias limpias, spread normal |
| Nueva York | 09:00 – 17:00 | Mayor volatilidad, spreads se ensanchan en noticias |
| Asia | 21:00 – 04:00 | Movimiento lento, spreads altos — **evitar** |

> Para Fase 6 se puede agregar un filtro horario que deshabilite el bot en sesión asiática.

---

*Config generada con Claude Code — v5 del bot — 2026-05-13*
