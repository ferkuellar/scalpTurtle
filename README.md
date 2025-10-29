> **Nota sobre “Chain-of-Thought”**: en lugar de exponer razonamiento paso a paso (que no comparto por política), te doy un **resumen táctico claro** y todo el **diseño operativo + pseudo-código** para implementarlo ya.

### Breve razonamiento (alto nivel)

* Tomo el **núcleo Turtle**: ruptura de canal (Donchian), riesgo por unidad **N = ATR**, y **SL/Trailing en múltiplos de N**.
* Lo adapto a **marcos 1m/5m** (mucho ruido) con **filtros de calidad**: 1) **Volumen** > media reciente (para evitar rupturas vacías), 2) **RSI** en régimen direccional (≥50 para largos, ≤50 para cortos), 3) **distancia mínima** del breakout (≥0.25–0.5N) para evitar “mechazos”.
* **Dimensiono por volatilidad**: el tamaño de la posición cae cuando sube el ATR, manteniendo **riesgo fijo** (1–2% por trade).
* **Salida híbrida**: SL inicial en N-multiplicador, **TP con R:R ≥ 1.5–2.0** y **trailing por ATR/Donchian corto**; opcional **scale-out** a 1R y breakeven.

---

# SECCIÓN 1: Parámetros del Bot y Gestión de Riesgo (ATR)

1. **Marco Temporal**

* Principal: **1m** (ultra-rápido) o **5m** (más estable).
* Recomendación: 5m para empezar; 1m cuando ya afinas latencia y microestructura.

2. **Unidad de Riesgo (N)**

* **N = ATR(p_atr, fuente=high,low,close)** en el **timeframe de ejecución**.
* Valores típicos: `p_atr = 14` (rápido) o `21` (más suave).
* **Volatility clamp** (filtro): habilitar entradas solo si
  [
  0.6 \le \frac{ATR_{p}}{SMA(ATR_{p},,p_{atr_ma})} \le 2.2
  ]
  con `p_atr_ma = 20`. (Evita chop extremo o sobre-volatilidad).

3. **Dimensionamiento de Posición** (riesgo fijo 1–2% del capital)

* Riesgo monetario por trade:
  [
  R$ = capital \times riesgo_{por_op} \quad (\text{ej. } 0.01 \text{ o } 0.02)
  ]
* **Stop técnico inicial** (ver Sección 3): (\text{SL_dist} = k_{SL}\cdot N) (ej. (k_{SL}=1.5)).
* **Tamaño** (Spot/Perp USDT-margined):
  [
  qty = \frac{R$}{SL_dist \times \text{valor_por_unidad}}
  ]
  donde (\text{valor_por_unidad} \approx precio) en contratos lineales.
* **Cap**: respetar **tamaño máximo** por liquidez y **riesgo de cola** (ej. `qty_max_usd = 5% del capital`).

> *Tip Turtle opcional (piramidación):* hasta **4 unidades**, añadiendo 0.5–1.0N a favor; cada add con su propio SL de cartera. En 1m/5m úsalo con prudencia.

---

# SECCIÓN 2: Lógica de Entrada (Buy & Short Sell)

**Definiciones base**

* **Canal Donchian (ruptura)**:

  * `B` = lookback de ruptura (ej. 1m: **60** velas; 5m: **24**).
  * `X` = lookback de salida/canal corto (ej. 1m: **20**; 5m: **10**).
  * **Resistencia (High_B)** = máximo de las últimas `B` velas.
  * **Soporte (Low_B)** = mínimo de las últimas `B` velas.
* **Volumen filtro**: `Vol > SMA(Vol, 20) * α`, con **α = 1.2–1.5** (ajustable por par).
* **RSI filtro**: `RSI(p_rsi=14)`:

  * Régimen **alcista**: `RSI >= 52–55` (mejor si RSI sube vs. su SMA(9)).
  * Régimen **bajista**: `RSI <= 48–45` (mejor si RSI baja vs. su SMA(9)).
* **Distancia de ruptura**: cierre por encima/debajo del nivel **≥ δ·N** (con **δ = 0.25–0.50**) para evitar falsos quiebres microscópicos.

### Largos (Buy)

1. **Ruptura**: `close_t >= High_B` **y** `(close_t - High_B) >= δ·N`.
2. **Volumen**: `Vol_t >= SMA(Vol,20) * α`.
3. **RSI**: `RSI_t >= 52–55` **y** `RSI_t > SMA(RSI,9)` (momento real).
4. **ATR clamp** dentro de rango permitido (ver Sección 1).
5. **No operar si** spread/latencia/fees estimadas > `γ` bps (define `γ`, ej. 4–8 bps).

### Cortos (Short)

1. **Ruptura**: `close_t <= Low_B` **y** `(Low_B - close_t) >= δ·N`.
2. **Volumen**: `Vol_t >= SMA(Vol,20) * α`.
3. **RSI**: `RSI_t <= 48–45` **y** `RSI_t < SMA(RSI,9)`.
4. **ATR clamp** dentro de rango permitido.
5. **Mismo control de costos** (γ bps).

> *Rebote S/R (opcional, modo “fade con confirmación”):*
> − En **soporte**: pin bar/engulfing + Volumen > α y **RSI cruza↑ 50** ⇒ long con SL bajo el swing − k_{SL}·N.
> − En **resistencia**: análogo para short. Úsalo solo si el sistema base (breakout) está *off* por sobre-extensión.

---

# SECCIÓN 3: Lógica de Salida (Take Profit & Stop Loss)

1. **SL Inicial**

* **Long**:
  [
  SL = entry_price - k_{SL}\cdot N \quad (k_{SL}=1.5 \text{ por defecto})
  ]
* **Short**:
  [
  SL = entry_price + k_{SL}\cdot N
  ]

2. **TP Objetivo**

* **R:R mínimo** = **1:1.8** (recomendado **≥2:1** en 1m para cubrir fees/slippage).
  [
  TP = entry_price \pm R\cdot (k_{SL}\cdot N)
  ]
  (signo “+” para long, “−” para short).
* **Scale-out opcional**: cerrar **50% en 1R**, mover **SL a BE − ε** (ε = 0.1N), dejar correr hasta trailing/2R.

3. **Trailing Dinámico**

* **Chandelier-ATR**:

  * Long: `TS = max(TS, highest_high(X) - k_tr·ATR)`, con **k_tr = 2.0**.
  * Short: `TS = min(TS, lowest_low(X) + k_tr·ATR)`.
* **Canal de salida (Donchian X)**:

  * Long: salir si `close < lowest_low(X)`
  * Short: salir si `close > highest_high(X)`
* **Hard stop manda**: si **SL** o **TS** tocan primero, se ejecuta salida.

4. **Guardas operativas**

* **Daily Stop**: detener trading si `pérdida diaria >= D%` (ej. 3%).
* **Max Trades** por sesión (ej. 6–10).
* **Cooldown**: tras **3 pérdidas consecutivas**, pausar `M` minutos (ej. 30–60).

---

# SECCIÓN 4: Estructura de Ejecución (Pseudo-código para API)

```python
# ===============================
#  PSEUDO-CÓDIGO: CryptoScalperBot
# ===============================

class CryptoScalperBot:
    def __init__(self, capital, riesgo_por_op=0.01, timeframe="5m",
                 p_atr=14, p_atr_ma=20, p_rsi=14,
                 B=24, X=10, alpha_vol=1.3, delta_N=0.35,
                 k_SL=1.5, R_target=2.0, k_tr=2.0,
                 daily_stop_pct=0.03, max_trades_day=8, cooldown_min=45):
        self.capital = capital
        self.riesgo = riesgo_por_op
        self.tf = timeframe

        # Parámetros indicadores
        self.p_atr = p_atr
        self.p_atr_ma = p_atr_ma
        self.p_rsi = p_rsi

        # Donchian / filtros
        self.B = B            # ruptura
        self.X = X            # salida/trailing corto
        self.alpha_vol = alpha_vol
        self.delta_N = delta_N  # distancia mínima en N

        # Gestión de riesgo/salidas
        self.k_SL = k_SL
        self.R_target = R_target
        self.k_tr = k_tr

        # Gobernanza
        self.daily_stop_pct = daily_stop_pct
        self.max_trades_day = max_trades_day
        self.cooldown_min = cooldown_min

        # Estado
        self.trades_today = 0
        self.pnl_today = 0.0
        self.cooldown_until = None
        self.position = None  # dict con {side, qty, entry, SL, TP, TS}

    # ---------------------------
    # Cálculo de indicadores base
    # ---------------------------
    def calcular_indicadores(self, data):
        """
        data: OHLCV en self.tf (lista o DataFrame con columnas: time, open, high, low, close, volume)
        Debe devolver un dict con:
          ATR, ATR_MA, RSI, Vol_MA20, High_B, Low_B, High_X, Low_X
        """
        atr = ATR(data.high, data.low, data.close, period=self.p_atr)
        atr_ma = SMA(atr, self.p_atr_ma)
        rsi = RSI(data.close, period=self.p_rsi)
        vol_ma20 = SMA(data.volume, 20)

        high_B = ROLLING_MAX(data.high, self.B)
        low_B  = ROLLING_MIN(data.low, self.B)
        high_X = ROLLING_MAX(data.high, self.X)
        low_X  = ROLLING_MIN(data.low, self.X)

        return {
            "ATR": atr, "ATR_MA": atr_ma, "RSI": rsi, "VolMA20": vol_ma20,
            "High_B": high_B, "Low_B": low_B, "High_X": high_X, "Low_X": low_X
        }

    # ---------------------------
    # Sizing por N (ATR)
    # ---------------------------
    def calcular_tamano(self, price, N):
        riesgo_monetario = self.capital * self.riesgo
        sl_dist = self.k_SL * N
        valor_por_unidad = price  # contratos lineales USDT: ~price
        qty = riesgo_monetario / (sl_dist * valor_por_unidad)
        qty = self._ajustar_a_tick(qty)
        return max(qty, 0.0)

    # ---------------------------
    # Reglas de entrada
    # ---------------------------
    def logica_entrada(self, row, ind):
        """
        row: última vela (close, high, low, volume, time)
        ind: dict de indicadores en el índice de 'row'
        """
        if self._en_cooldown() or self._violacion_governance():
            return None

        N = ind["ATR"][-1]
        atr_ratio = N / ind["ATR_MA"][-1] if ind["ATR_MA"][-1] > 0 else 1.0
        if atr_ratio < 0.6 or atr_ratio > 2.2:  # volatility clamp
            return None

        vol_ok = row.volume >= ind["VolMA20"][-1] * self.alpha_vol
        rsi = ind["RSI"][-1]

        # Distancias de ruptura (en N)
        dist_long = row.close - ind["High_B"][-1]
        dist_short = ind["Low_B"][-1] - row.close

        # ---- Señal LONG ----
        if (row.close >= ind["High_B"][-1] and
            dist_long >= self.delta_N * N and
            vol_ok and
            rsi >= 52 and rsi > SMA(ind["RSI"], 9)[-1]):
            side = "LONG"
            qty = self.calcular_tamano(row.close, N)
            if qty > 0:
                sl = row.close - self.k_SL * N
                tp = row.close + self.R_target * (self.k_SL * N)
                ts = max(ind["High_X"][-1] - self.k_tr * N, sl)  # Chandelier inicial
                return {"side": side, "qty": qty, "entry": row.close, "SL": sl, "TP": tp, "TS": ts}

        # ---- Señal SHORT ----
        if (row.close <= ind["Low_B"][-1] and
            dist_short >= self.delta_N * N and
            vol_ok and
            rsi <= 48 and rsi < SMA(ind["RSI"], 9)[-1]):
            side = "SHORT"
            qty = self.calcular_tamano(row.close, N)
            if qty > 0:
                sl = row.close + self.k_SL * N
                tp = row.close - self.R_target * (self.k_SL * N)
                ts = min(ind["Low_X"][-1] + self.k_tr * N, sl)
                return {"side": side, "qty": qty, "entry": row.close, "SL": sl, "TP": tp, "TS": ts}

        return None

    # ---------------------------
    # Gestión de salida / trailing
    # ---------------------------
    def logica_salida(self, row, ind):
        if not self.position:
            return None

        N = ind["ATR"][-1]
        pos = self.position
        # Actualizar trailing por Chandelier + canal X
        if pos["side"] == "LONG":
            ts_new = max(pos["TS"], ind["High_X"][-1] - self.k_tr * N)
            pos["TS"] = max(ts_new, pos["SL"])  # Hard stop domina
            # Condiciones de salida
            if row.low <= pos["SL"] or row.low <= pos["TS"] or row.close <= ind["Low_X"][-1] or row.high >= pos["TP"]:
                return "EXIT"
        else:  # SHORT
            ts_new = min(pos["TS"], ind["Low_X"][-1] + self.k_tr * N)
            pos["TS"] = min(ts_new, pos["SL"])
            if row.high >= pos["SL"] or row.high >= pos["TS"] or row.close >= ind["High_X"][-1] or row.low <= pos["TP"]:
                return "EXIT"

        return None

    # ---------------------------
    # Loop principal
    # ---------------------------
    def ejecutar_bot(self, feed):
        """
        feed: generador/stream de velas OHLCV en self.tf
        """
        for data in feed:  # data es ventana deslizante con la última vela cerrada
            ind = self.calcular_indicadores(data)
            row = data[-1]  # última vela

            if not self.position:
                señal = self.logica_entrada(row, ind)
                if señal:
                    self._enviar_orden(señal)
                    self.position = señal
                    self.trades_today += 1
            else:
                accion = self.logica_salida(row, ind)
                if accion == "EXIT":
                    pnl = self._cerrar_posicion(row.close)
                    self.pnl_today += pnl
                    self.position = None
                    self._aplicar_cooldown_si_corresponde(pnl)

    # ---- Helpers (stubs a implementar con la API real) ----
    def _ajustar_a_tick(self, qty): ...
    def _enviar_orden(self, orden): ...
    def _cerrar_posicion(self, price): ...
    def _en_cooldown(self): ...
    def _violacion_governance(self):
        return (self.trades_today >= self.max_trades_day or
                abs(self.pnl_today) >= self.capital * self.daily_stop_pct)
    def _aplicar_cooldown_si_corresponde(self, pnl): ...
```

---

## Sugerencias de **sets iniciales** por par/timeframe

* **BTCUSDT 5m**: `B=24`, `X=10`, `p_atr=14`, `alpha_vol=1.3`, `delta_N=0.35`, `k_SL=1.5`, `R=2.0`, `k_tr=2.0`.
* **ETHUSDT 5m**: `B=30`, `X=12`, `alpha_vol=1.4`, resto igual.
* **BTCUSDT 1m** (más exigente): `B=60`, `X=20`, `alpha_vol=1.4–1.5`, `delta_N=0.5`, `k_SL=1.8`, `R=2.2`.
