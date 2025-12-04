//@version=5
strategy("Institutional Grade Strategy", overlay=true, initial_capital=10000, default_qty_type=strategy.percent_of_equity, default_qty_value=100, commission_type=strategy.commission.cash_per_order, commission_value=7, slippage=3, pyramiding=0)

// Indicadores
ema_8_len = input.int(8, "EMA 8", group="EMAs")
ema_21_len = input.int(21, "EMA 21", group="EMAs")
ema_50_len = input.int(50, "EMA 50", group="EMAs")
ema_200_len = input.int(200, "EMA 200", group="EMAs")
usar_ema200 = input.bool(true, "Filtrar EMA 200", group="EMAs")

rsi_len = input.int(14, "RSI Período", group="Momentum")
rsi_bull_zone = input.int(52, "RSI Bull Min", group="Momentum")
rsi_bear_zone = input.int(48, "RSI Bear Max", group="Momentum")
rsi_high = input.int(68, "RSI Sobrecompra", group="Momentum")
rsi_low = input.int(32, "RSI Sobrevenda", group="Momentum")

adx_len = input.int(14, "ADX Período", group="Tendência")
adx_forte = input.int(25, "ADX Forte", group="Tendência")
usar_di = input.bool(true, "Usar DI Cross", group="Tendência")

atr_len = input.int(14, "ATR Período", group="Volatilidade")
atr_min = input.float(0.5, "ATR Min Mult", group="Volatilidade")
filtro_vol = input.bool(true, "Filtrar Volatilidade", group="Volatilidade")

usar_pa = input.bool(true, "Price Action", group="Confirmações")
tamanho_candle = input.float(60, "Candle Min %", group="Confirmações")
usar_structure = input.bool(true, "Market Structure", group="Confirmações")
lookback = input.int(10, "Lookback", group="Confirmações")

rr_ratio = input.float(1.8, "Risk:Reward", group="Risco")
stop_atr = input.float(2.0, "Stop ATR Mult", group="Risco")
usar_stop_estrut = input.bool(true, "Stop Estrutural", group="Risco")

usar_trailing = input.bool(true, "Trailing Stop", group="Risco")
trail_start = input.float(50, "Trailing Start %", group="Risco")
trail_atr = input.float(1.5, "Trailing ATR", group="Risco")
usar_be = input.bool(true, "Breakeven", group="Risco")
be_trigger = input.float(40, "Breakeven %", group="Risco")

usar_sessao = input.bool(true, "Filtro Sessão", group="Tempo")
sess_london = input.bool(true, "Londres", group="Tempo")
sess_ny = input.bool(true, "Nova York", group="Tempo")
evitar_sex = input.bool(true, "Evitar Sexta 16h+", group="Tempo")
evitar_seg = input.bool(true, "Evitar Segunda 10h-", group="Tempo")

mostrar_emas = input.bool(true, "Mostrar EMAs", group="Visual")
mostrar_sinais = input.bool(true, "Mostrar Sinais", group="Visual")
mostrar_dash = input.bool(true, "Dashboard", group="Visual")

// Cálculos
e8 = ta.ema(close, ema_8_len)
e21 = ta.ema(close, ema_21_len)
e50 = ta.ema(close, ema_50_len)
e200 = ta.ema(close, ema_200_len)

atr = ta.atr(atr_len)
atr_ma = ta.sma(atr, 20)
vol_ok = not filtro_vol or atr > atr_ma * atr_min

[diPlus, diMinus, adx] = ta.dmi(adx_len, adx_len)
tend_forte = adx > adx_forte

rsi = ta.rsi(close, rsi_len)

[macd_line, signal_line, macd_hist] = ta.macd(close, 12, 26, 9)
macd_bull = macd_line > signal_line and macd_hist > 0
macd_bear = macd_line < signal_line and macd_hist < 0

// Structure
var float ultimo_hh = na
var float ultimo_ll = na

if usar_structure
    if ta.pivothigh(high, lookback, lookback)
        if na(ultimo_hh) or high[lookback] > ultimo_hh
            ultimo_hh := high[lookback]
    if ta.pivotlow(low, lookback, lookback)
        if na(ultimo_ll) or low[lookback] < ultimo_ll
            ultimo_ll := low[lookback]

struct_bull = usar_structure ? not na(ultimo_hh) and high >= ultimo_hh * 0.998 : true
struct_bear = usar_structure ? not na(ultimo_ll) and low <= ultimo_ll * 1.002 : true

// Price Action
corpo = math.abs(close - open)
sombra = high - low
tam_candle = sombra > 0 ? corpo / sombra * 100 : 0

candle_bull = close > open and tam_candle > tamanho_candle
candle_bear = close < open and tam_candle > tamanho_candle

pa_bull = usar_pa ? candle_bull : true
pa_bear = usar_pa ? candle_bear : true

// Sessão
hora = hour(time, "UTC")
dia = dayofweek(time)

sessao_ok = not usar_sessao or (sess_london and hora >= 8 and hora < 17) or (sess_ny and hora >= 13 and hora < 22)
evitar_hora = (evitar_sex and dia == dayofweek.friday and hora >= 16) or (evitar_seg and dia == dayofweek.monday and hora < 10)

// Score LONG
var int score_long = 0
score_long := 0

if e8 > e21 and e21 > e50
    score_long += 2
if not usar_ema200 or close > e200
    score_long += 1
if tend_forte
    score_long += 1
if rsi > rsi_bull_zone and rsi < rsi_high
    score_long += 1
if macd_bull
    score_long += 1
if pa_bull
    score_long += 1
if struct_bull
    score_long += 1
if not usar_di or diPlus > diMinus
    score_long += 1
if vol_ok
    score_long += 1

// Score SHORT
var int score_short = 0
score_short := 0

if e8 < e21 and e21 < e50
    score_short += 2
if not usar_ema200 or close < e200
    score_short += 1
if tend_forte
    score_short += 1
if rsi < rsi_bear_zone and rsi > rsi_low
    score_short += 1
if macd_bear
    score_short += 1
if pa_bear
    score_short += 1
if struct_bear
    score_short += 1
if not usar_di or diMinus > diPlus
    score_short += 1
if vol_ok
    score_short += 1

// Triggers
trigger_long = score_long >= 8 and ta.crossover(e8, e21) and sessao_ok and not evitar_hora
trigger_short = score_short >= 8 and ta.crossunder(e8, e21) and sessao_ok and not evitar_hora

// Gestão
var float entry_price = na
var float stop_loss = na
var float take_profit = na
var float trailing_stop_price = na
var bool be_ativo = false

swing_low = ta.lowest(low, lookback)
swing_high = ta.highest(high, lookback)

if trigger_long and strategy.position_size == 0
    entry_price := close
    stop_atr_calc = close - atr * stop_atr
    stop_struct = swing_low - atr * 0.5
    stop_loss := usar_stop_estrut ? math.max(stop_atr_calc, stop_struct) : stop_atr_calc
    risco = entry_price - stop_loss
    take_profit := entry_price + risco * rr_ratio
    trailing_stop_price := na
    be_ativo := false

if trigger_short and strategy.position_size == 0
    entry_price := close
    stop_atr_calc = close + atr * stop_atr
    stop_struct = swing_high + atr * 0.5
    stop_loss := usar_stop_estrut ? math.min(stop_atr_calc, stop_struct) : stop_atr_calc
    risco = stop_loss - entry_price
    take_profit := entry_price - risco * rr_ratio
    trailing_stop_price := na
    be_ativo := false

// Execução
if trigger_long and strategy.position_size == 0
    strategy.entry("LONG", strategy.long, comment="LONG Score:" + str.tostring(score_long))

if trigger_short and strategy.position_size == 0
    strategy.entry("SHORT", strategy.short, comment="SHORT Score:" + str.tostring(score_short))

// Gestão Posição
if strategy.position_size != 0
    risco_total = strategy.position_size > 0 ? entry_price - stop_loss : stop_loss - entry_price
    lucro = strategy.position_size > 0 ? close - entry_price : entry_price - close
    lucro_pct = lucro / risco_total * 100
    
    if strategy.position_size > 0
        if usar_be and not be_ativo and lucro_pct >= be_trigger
            stop_loss := entry_price + atr * 0.2
            be_ativo := true
        
        if usar_trailing and lucro_pct >= trail_start
            novo_trail = close - atr * trail_atr
            if na(trailing_stop_price) or novo_trail > trailing_stop_price
                trailing_stop_price := novo_trail
        
        stop_final = not na(trailing_stop_price) ? trailing_stop_price : stop_loss
        strategy.exit("Exit LONG", "LONG", stop=stop_final, limit=take_profit)
    
    if strategy.position_size < 0
        if usar_be and not be_ativo and lucro_pct >= be_trigger
            stop_loss := entry_price - atr * 0.2
            be_ativo := true
        
        if usar_trailing and lucro_pct >= trail_start
            novo_trail = close + atr * trail_atr
            if na(trailing_stop_price) or novo_trail < trailing_stop_price
                trailing_stop_price := novo_trail
        
        stop_final = not na(trailing_stop_price) ? trailing_stop_price : stop_loss
        strategy.exit("Exit SHORT", "SHORT", stop=stop_final, limit=take_profit)

// Visual
plot(mostrar_emas ? e8 : na, "EMA 8", color.new(color.blue, 0), 2)
plot(mostrar_emas ? e21 : na, "EMA 21", color.new(color.orange, 0), 2)
plot(mostrar_emas ? e50 : na, "EMA 50", color.new(color.purple, 0), 2)
plot(mostrar_emas ? e200 : na, "EMA 200", color.new(color.red, 0), 3)

plotshape(trigger_long and mostrar_sinais, title="LONG", style=shape.triangleup, location=location.belowbar, color=color.new(color.green, 0), size=size.normal)
plotshape(trigger_short and mostrar_sinais, title="SHORT", style=shape.triangledown, location=location.abovebar, color=color.new(color.red, 0), size=size.normal)

bgcolor(trigger_long ? color.new(color.green, 93) : na)
bgcolor(trigger_short ? color.new(color.red, 93) : na)
bgcolor(not sessao_ok ? color.new(color.gray, 97) : na)

// Dashboard
if mostrar_dash
    var table dash = table.new(position.top_right, 2, 8, bgcolor=color.new(color.black, 90), frame_color=color.gray, frame_width=1)
    
    if barstate.islast
        table.cell(dash, 0, 0, "STRATEGY", bgcolor=color.new(color.blue, 70), text_color=color.white)
        table.cell(dash, 1, 0, "v1.0", bgcolor=color.new(color.blue, 70), text_color=color.white)
        
        score_l_color = score_long >= 8 ? color.new(color.green, 70) : color.new(color.red, 70)
        table.cell(dash, 0, 1, "Long Score", text_color=color.white)
        table.cell(dash, 1, 1, str.tostring(score_long) + "/10", bgcolor=score_l_color, text_color=color.white)
        
        score_s_color = score_short >= 8 ? color.new(color.red, 70) : color.new(color.green, 70)
        table.cell(dash, 0, 2, "Short Score", text_color=color.white)
        table.cell(dash, 1, 2, str.tostring(score_short) + "/10", bgcolor=score_s_color, text_color=color.white)
        
        adx_color = adx > adx_forte ? color.new(color.green, 70) : color.new(color.red, 70)
        table.cell(dash, 0, 3, "ADX", text_color=color.white)
        table.cell(dash, 1, 3, str.tostring(adx, "#.#"), bgcolor=adx_color, text_color=color.white)
        
        rsi_color = rsi > 50 ? color.new(color.green, 70) : color.new(color.red, 70)
        table.cell(dash, 0, 4, "RSI", text_color=color.white)
        table.cell(dash, 1, 4, str.tostring(rsi, "#.#"), bgcolor=rsi_color, text_color=color.white)
        
        tend = e8 > e21 and e21 > e50 ? "ALTA" : e8 < e21 and e21 < e50 ? "BAIXA" : "LATERAL"
        tend_color = e8 > e21 and e21 > e50 ? color.new(color.green, 70) : e8 < e21 and e21 < e50 ? color.new(color.red, 70) : color.new(color.orange, 70)
        table.cell(dash, 0, 5, "Tendencia", text_color=color.white)
        table.cell(dash, 1, 5, tend, bgcolor=tend_color, text_color=color.white)
        
        sess = hora >= 8 and hora < 17 ? "LONDON" : hora >= 13 and hora < 22 ? "NY" : "CLOSED"
        sess_color = sessao_ok ? color.new(color.green, 70) : color.new(color.red, 70)
        table.cell(dash, 0, 6, "Sessao", text_color=color.white)
        table.cell(dash, 1, 6, sess, bgcolor=sess_color, text_color=color.white)
        
        table.cell(dash, 0, 7, "R:R", text_color=color.white)
        table.cell(dash, 1, 7, "1:" + str.tostring(rr_ratio, "#.#"), bgcolor=color.new(color.blue, 70), text_color=color.white)

alertcondition(trigger_long, "LONG Signal", "COMPRA Score: {{plot_0}}")
alertcondition(trigger_short, "SHORT Signal", "VENDA Score: {{plot_0}}")
