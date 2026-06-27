// This Pine Script™ indicator is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © Trend Break Detector v6 + SMC

//@version=6
indicator("Trend Break Detector v6 + SMC", "TBDv6+SMC", overlay = true,
     max_lines_count = 500, max_labels_count = 500, max_boxes_count = 500)

// ═══════════════════════════════════════════════════════════════
//  НАСТРОЙКИ — SWING DETECTION
// ═══════════════════════════════════════════════════════════════

string GRP_SWING = "══ Swing Detection ══"
int    swing_len = input.int(5, "Slow Swing Length", minval = 2, maxval = 50, group = GRP_SWING,
     tooltip = "Для подтверждённых линий. Быстрые используют половину.")
bool   use_fast  = input.bool(true, "Fast Lines (early signals)", group = GRP_SWING,
     tooltip = "Линии по пивотам с половинной длиной — раньше видят разворот.")

// ═══════════════════════════════════════════════════════════════
//  НАСТРОЙКИ — TREND LINES
// ═══════════════════════════════════════════════════════════════

string GRP_LINE  = "══ Trend Lines ══"
int    lw        = input.int(2, "Trend Line Width", minval = 1, maxval = 4, group = GRP_LINE)
color  col_sup   = input.color(#00BFA5, "Support Color (bull)", group = GRP_LINE)
color  col_res   = input.color(#FF1744, "Resistance Color (bear)", group = GRP_LINE)
bool   extend_r  = input.bool(true, "Extend Line Right", group = GRP_LINE)
int    pool_size = input.int(3, "Max Active Lines / dir", minval = 1, maxval = 10, group = GRP_LINE)
int    min_slope_bars = input.int(0, "Min Slope Bars (0=auto)", minval = 0, maxval = 50, group = GRP_LINE,
     tooltip = "0 = auto (2x swing_len). Отсекает слишком крутые линии.")

// ═══════════════════════════════════════════════════════════════
//  НАСТРОЙКИ — BREAK DETECTION
// ═══════════════════════════════════════════════════════════════

string GRP_BRK   = "══ Break Detection ══"
string mode      = input.string("Close", "Break Source", options = ["Wick","Close","Both"], group = GRP_BRK)
int    atr_len   = input.int(14, "ATR Length", minval = 1, group = GRP_BRK)
float  atr_flt   = input.float(0.5, "ATR Filter (min distance)", minval = 0.0, step = 0.1, group = GRP_BRK)
bool   use_vol   = input.bool(true, "Require Volume Spike", group = GRP_BRK)
float  vol_mul   = input.float(1.3, "Volume vs SMA Ratio", minval = 0.5, step = 0.1, group = GRP_BRK)
bool   use_vol_proj = input.bool(true, "Volume Projection (real-time)", group = GRP_BRK)
int    confirm_bars = input.int(2, "Confirmation Bars (slow)", minval = 1, maxval = 5, group = GRP_BRK)
string trend_mode = input.string("Price vs EMA", "Trend Filter", options = ["EMA Cross","Price vs EMA","Off"], group = GRP_BRK)
float  inst_conf_atr = input.float(2.0, "Instant Confirm ATR (0=off)", minval = 0.0, step = 0.5, group = GRP_BRK)
bool   use_body  = input.bool(true, "Body Structure Filter", group = GRP_BRK,
     tooltip = "Пробой силён если close за линией > среднего тела свечи (10 баров).")
string mom_mode  = input.string("RSI", "Momentum Filter", options = ["Off","RSI","MACD","Both"], group = GRP_BRK,
     tooltip = "RSI>50 бычий / <50 медвежий. MACD гистограмма ускоряется в направлении.")
int    rsi_len   = input.int(14, "RSI Length", minval = 2, group = GRP_BRK)
int    max_hist  = input.int(50, "Max Historical Lines", minval = 5, maxval = 200, group = GRP_BRK)

// ═══════════════════════════════════════════════════════════════
//  НАСТРОЙКИ — SMC / STRUCTURE
// ═══════════════════════════════════════════════════════════════

string GRP_SMC   = "══ SMC / Structure ══"
bool   use_smc   = input.bool(true, "Enable SMC Module", group = GRP_SMC)
bool   show_bos  = input.bool(true, "Show BOS Signals", group = GRP_SMC)
bool   show_choch = input.bool(true, "Show CHOCH Signals", group = GRP_SMC)
bool   show_swing_lvl = input.bool(true, "Show Swing Level Lines", group = GRP_SMC)
color  col_bos   = input.color(#2962FF, "BOS Color", group = GRP_SMC)
color  col_choch = input.color(#E040FB, "CHOCH Color", group = GRP_SMC)

string GRP_DISP  = "══ Displacement ══"
float  disp_mul  = input.float(1.5, "Displacement Body Multiplier", minval = 1.0, step = 0.1, group = GRP_DISP,
     tooltip = "Свеча импульсная если тело > avg_body x этот множитель.")
bool   req_disp  = input.bool(false, "Require Displacement for SMC Break", group = GRP_DISP,
     tooltip = "Если включено, BOS/CHOCH только при импульсной свече.")

string GRP_OB    = "══ Order Blocks ══"
bool   show_ob   = input.bool(true, "Show Order Blocks", group = GRP_OB)
int    ob_max    = input.int(5, "Max Active Order Blocks", minval = 1, maxval = 20, group = GRP_OB)
int    ob_lookback = input.int(10, "OB Lookback Bars", minval = 3, maxval = 30, group = GRP_OB)
bool   ob_mitigate = input.bool(true, "Remove Mitigated OBs", group = GRP_OB,
     tooltip = "Убирать OB когда цена закрывается за его границей.")
color  col_ob_bull = input.color(color.new(#00BFA5, 80), "Bullish OB Color", group = GRP_OB)
color  col_ob_bear = input.color(color.new(#FF1744, 80), "Bearish OB Color", group = GRP_OB)

string GRP_FVG   = "══ Fair Value Gaps ══"
bool   show_fvg  = input.bool(true, "Show FVG Zones", group = GRP_FVG)
int    fvg_max   = input.int(5, "Max Active FVGs", minval = 1, maxval = 20, group = GRP_FVG)
bool   fvg_remove_filled = input.bool(true, "Remove Filled FVGs", group = GRP_FVG)
color  col_fvg_bull = input.color(color.new(#00BFA5, 85), "Bullish FVG Color", group = GRP_FVG)
color  col_fvg_bear = input.color(color.new(#FF1744, 85), "Bearish FVG Color", group = GRP_FVG)

string GRP_LIQ   = "══ Liquidity ══"
bool   show_liq  = input.bool(true, "Show Equal H/L Liquidity", group = GRP_LIQ)
float  eq_atr_pct = input.float(0.1, "Equal Level ATR Threshold", minval = 0.01, step = 0.01, group = GRP_LIQ,
     tooltip = "Два свинга = equal если разница < ATR x порог.")
bool   show_sweep = input.bool(true, "Mark Liquidity Sweeps", group = GRP_LIQ)
color  col_liq   = input.color(color.new(#FFD600, 50), "Liquidity Level Color", group = GRP_LIQ)
int    liq_max   = input.int(5, "Max Liquidity Levels", minval = 1, maxval = 10, group = GRP_LIQ)

string GRP_PD    = "══ Premium / Discount ══"
bool   show_pd   = input.bool(false, "Show Premium/Discount Zones", group = GRP_PD)
color  col_premium = input.color(color.new(#FF1744, 93), "Premium Zone Color", group = GRP_PD)
color  col_discount = input.color(color.new(#00BFA5, 93), "Discount Zone Color", group = GRP_PD)

string GRP_MTF   = "══ Multi-Timeframe ══"
bool   use_mtf   = input.bool(false, "Enable MTF Confirmation", group = GRP_MTF,
     tooltip = "Бычий сигнал только если старший ТФ не медвежий. +1 бар задержка HTF.")
string htf       = input.timeframe("15", "Higher Timeframe", group = GRP_MTF)

// ═══════════════════════════════════════════════════════════════
//  НАСТРОЙКИ — VISUALS
// ═══════════════════════════════════════════════════════════════

string GRP_VIS   = "══ Visuals ══"
bool   show_bg   = input.bool(true, "Highlight Break Bars", group = GRP_VIS)
color  bg_up     = input.color(color.new(#00BFA5, 90), "Bull BG Color", group = GRP_VIS)
color  bg_dn     = input.color(color.new(#FF1744, 90), "Bear BG Color", group = GRP_VIS)
bool   show_tent = input.bool(false, "Show Tentative Signals", group = GRP_VIS)
bool   show_fast = input.bool(true, "Show Fast Break Signals", group = GRP_VIS)
bool   show_ema  = input.bool(true, "Show EMA Cloud", group = GRP_VIS)
int    ema_fl    = input.int(20, "EMA Fast", group = GRP_VIS)
int    ema_sl    = input.int(50, "EMA Slow", group = GRP_VIS)

// ═══════════════════════════════════════════════════════════════
//  НАСТРОЙКИ — ALERTS
// ═══════════════════════════════════════════════════════════════

string GRP_ALR   = "══ Alerts ══"
bool   alr_bull  = input.bool(true, "Alert: Bullish Break", group = GRP_ALR)
bool   alr_bear  = input.bool(true, "Alert: Bearish Break", group = GRP_ALR)
bool   alr_fast  = input.bool(true, "Alert: Fast Break", group = GRP_ALR)
bool   alr_tent  = input.bool(false, "Alert: Tentative", group = GRP_ALR)
bool   alr_bos   = input.bool(true, "Alert: BOS", group = GRP_ALR)
bool   alr_choch = input.bool(true, "Alert: CHOCH", group = GRP_ALR)
bool   alr_sweep = input.bool(true, "Alert: Liquidity Sweep", group = GRP_ALR)

// ═══════════════════════════════════════════════════════════════
//  ПРИМИТИВЫ
// ═══════════════════════════════════════════════════════════════

atr     = ta.atr(atr_len)
vol_sma = ta.sma(volume, 20)

bool   vol_warmup  = bar_index >= 20
float  vol_sma_ok  = not na(vol_sma) ? vol_sma : 0.0
bool   vp_active = use_vol and use_vol_proj and barstate.isrealtime and time_close > time
float  vol_pct     = vp_active ? math.max(0.2, math.min(1.0, (timenow - time) / (time_close - time))) : 1.0
float  vol_thr     = vol_mul * vol_sma_ok * vol_pct
vol_ok = use_vol ? (vol_warmup and not na(volume) and not na(vol_sma) and volume >= vol_thr) : true

emaFast = ta.ema(close, ema_fl)
emaSlow = ta.ema(close, ema_sl)
pEMA1 = plot(show_ema ? emaFast : na, "EMA Fast", color = color.new(col_sup, 40))
pEMA2 = plot(show_ema ? emaSlow : na, "EMA Slow", color = color.new(col_res, 40))
fill(pEMA1, pEMA2, color = emaFast > emaSlow ? color.new(col_sup, 92) : color.new(col_res, 92), title = "EMA Cloud")

trend_bear_ok = trend_mode == "Off" ? true : trend_mode == "EMA Cross" ? (emaFast < emaSlow) : (close < emaFast)
trend_bull_ok = trend_mode == "Off" ? true : trend_mode == "EMA Cross" ? (emaFast > emaSlow) : (close > emaFast)

eff_min_slope = min_slope_bars > 0 ? min_slope_bars : swing_len * 2

avg_body = ta.sma(math.abs(close - open), 10)

rsi_val = ta.rsi(close, rsi_len)
rsi_bull_ok = rsi_val > 50
rsi_bear_ok = rsi_val < 50

[macd_l, macd_s, macd_h] = ta.macd(close, 12, 26, 9)
macd_bull_ok = not na(macd_h) and macd_h > 0 and macd_h > macd_h[1]
macd_bear_ok = not na(macd_h) and macd_h < 0 and macd_h < macd_h[1]

get_mom_bull() =>
    mom_mode == "Off" or (mom_mode == "RSI" and rsi_bull_ok) or (mom_mode == "MACD" and macd_bull_ok) or (mom_mode == "Both" and rsi_bull_ok and macd_bull_ok)

get_mom_bear() =>
    mom_mode == "Off" or (mom_mode == "RSI" and rsi_bear_ok) or (mom_mode == "MACD" and macd_bear_ok) or (mom_mode == "Both" and rsi_bear_ok and macd_bear_ok)

mom_bull_ok = get_mom_bull()
mom_bear_ok = get_mom_bear()

candle_body = math.abs(close - open)
is_displacement = not na(avg_body) and avg_body > 0 and candle_body > avg_body * disp_mul
disp_ok = not req_disp or is_displacement

// ═══════════════════════════════════════════════════════════════
//  СОСТОЯНИЕ — МЕДЛЕННЫЕ ЛИНИИ
// ═══════════════════════════════════════════════════════════════

var float[] sh_prices = array.new_float(0)
var int[]   sh_bars   = array.new_int(0)
var float[] sl_prices = array.new_float(0)
var int[]   sl_bars   = array.new_int(0)

var line[]  sup_pool = array.new_line(0)
var int[]   sup_x1 = array.new_int(0)
var float[] sup_y1 = array.new_float(0)
var int[]   sup_x2 = array.new_int(0)
var float[] sup_y2 = array.new_float(0)
var bool[]  sup_cd = array.new_bool(0)

var line[]  res_pool = array.new_line(0)
var int[]   res_x1 = array.new_int(0)
var float[] res_y1 = array.new_float(0)
var int[]   res_x2 = array.new_int(0)
var float[] res_y2 = array.new_float(0)
var bool[]  res_cd = array.new_bool(0)

var line[] hist_lines = array.new_line(0)

var int pend_bear_key = -1
var int pend_bear_cnt = 0
var int pend_bear_lb  = -1

var int pend_bull_key = -1
var int pend_bull_cnt = 0
var int pend_bull_lb  = -1

// ═══════════════════════════════════════════════════════════════
//  СОСТОЯНИЕ — БЫСТРЫЕ ЛИНИИ
// ═══════════════════════════════════════════════════════════════

int fast_len = math.max(2, swing_len / 2)

var float[] fsh_prices = array.new_float(0)
var int[]   fsh_bars   = array.new_int(0)
var float[] fsl_prices = array.new_float(0)
var int[]   fsl_bars   = array.new_int(0)

var line  fsup_ln = na
var int   fsup_x1 = -1
var float fsup_y1 = na
var int   fsup_x2 = -1
var float fsup_y2 = na
var int   fsup_cd_bar = -1

var line  fres_ln = na
var int   fres_x1 = -1
var float fres_y1 = na
var int   fres_x2 = -1
var float fres_y2 = na
var int   fres_cd_bar = -1

// ═══════════════════════════════════════════════════════════════
//  СОСТОЯНИЕ — SMC / СТРУКТУРА
// ═══════════════════════════════════════════════════════════════

var int   mkt_structure = 0    // 1=bullish, -1=bearish, 0=undefined
var float key_sh = na          // текущий ключевой swing high (уровень для пробоя вверх)
var int   key_sh_bar = -1
var float key_sl = na          // текущий ключевой swing low (уровень для пробоя вниз)
var int   key_sl_bar = -1
var float prev_sh_price = na   // предыдущий swing high (для определения HH/LH)
var float prev_sl_price = na   // предыдущий swing low (для определения HL/LL)
var bool  sh_broken = true     // ключевой SH уже пробит?
var bool  sl_broken = true     // ключевой SL уже пробит?

var line  swing_h_line = na
var line  swing_l_line = na

// Order Block
var box[]   ob_boxes = array.new_box(0)
var float[] ob_tops  = array.new_float(0)
var float[] ob_bots  = array.new_float(0)
var int[]   ob_dirs  = array.new_int(0)

// Fair Value Gap
var box[]   fvg_boxes = array.new_box(0)
var float[] fvg_tops  = array.new_float(0)
var float[] fvg_bots  = array.new_float(0)
var int[]   fvg_dirs  = array.new_int(0)

// Liquidity
var line[]  liq_lines  = array.new_line(0)
var float[] liq_prices = array.new_float(0)
var int[]   liq_dirs   = array.new_int(0)  // 1=equal highs, -1=equal lows

// ═══════════════════════════════════════════════════════════════
//  ФЛАГИ СИГНАЛОВ
// ═══════════════════════════════════════════════════════════════

bull_break = false
bear_break = false
bull_tent  = false
bear_tent  = false
fast_bull  = false
fast_bear  = false
smc_bull   = false
smc_bear   = false
smc_is_bos_bull   = false
smc_is_choch_bull = false
smc_is_bos_bear   = false
smc_is_choch_bear = false
sweep_bull = false
sweep_bear = false

// ═══════════════════════════════════════════════════════════════
//  MTF (request.security — глобальный уровень)
// ═══════════════════════════════════════════════════════════════

htf_struct = request.security(syminfo.tickerid, htf, mkt_structure, barmerge.gaps_off, barmerge.lookahead_off)
int htf_s = use_mtf and not na(htf_struct) ? htf_struct : 0
bool mtf_bull_ok = not use_mtf or htf_s >= 0
bool mtf_bear_ok = not use_mtf or htf_s <= 0

// ═══════════════════════════════════════════════════════════════
//  ПОМОЩНИКИ
// ═══════════════════════════════════════════════════════════════

clean_hist() =>
    while array.size(hist_lines) > max_hist
        line.delete(array.shift(hist_lines))

calc_line_at(int lx1, float ly1, int lx2, float ly2, int tx) =>
    d = lx2 - lx1
    s = d != 0 ? (ly2 - ly1) / d : 0.0
    ly1 + s * (tx - lx1)

remove_sup(int idx) =>
    if idx >= 0 and idx < array.size(sup_pool)
        ln = array.get(sup_pool, idx)
        if not na(ln)
            line.set_style(ln, line.style_dotted)
            line.set_color(ln, color.new(col_sup, 50))
            array.push(hist_lines, ln)
            clean_hist()
        array.remove(sup_pool, idx)
        array.remove(sup_x1, idx)
        array.remove(sup_y1, idx)
        array.remove(sup_x2, idx)
        array.remove(sup_y2, idx)
        array.remove(sup_cd, idx)

remove_res(int idx) =>
    if idx >= 0 and idx < array.size(res_pool)
        ln = array.get(res_pool, idx)
        if not na(ln)
            line.set_style(ln, line.style_dotted)
            line.set_color(ln, color.new(col_res, 50))
            array.push(hist_lines, ln)
            clean_hist()
        array.remove(res_pool, idx)
        array.remove(res_x1, idx)
        array.remove(res_y1, idx)
        array.remove(res_x2, idx)
        array.remove(res_y2, idx)
        array.remove(res_cd, idx)

retire_sup() =>
    if array.size(sup_pool) >= pool_size
        l = array.size(sup_pool) - 1
        ln = array.get(sup_pool, l)
        if not na(ln)
            array.push(hist_lines, ln)
            clean_hist()
        array.remove(sup_pool, l)
        array.remove(sup_x1, l)
        array.remove(sup_y1, l)
        array.remove(sup_x2, l)
        array.remove(sup_y2, l)
        array.remove(sup_cd, l)

retire_res() =>
    if array.size(res_pool) >= pool_size
        l = array.size(res_pool) - 1
        ln = array.get(res_pool, l)
        if not na(ln)
            array.push(hist_lines, ln)
            clean_hist()
        array.remove(res_pool, l)
        array.remove(res_x1, l)
        array.remove(res_y1, l)
        array.remove(res_x2, l)
        array.remove(res_y2, l)
        array.remove(res_cd, l)

body_bull_ok(float lp) =>
    if not use_body or na(avg_body) or avg_body <= 0
        true
    else
        float src = mode == "Wick" ? high : close
        (src - lp) > avg_body

body_bear_ok(float lp) =>
    if not use_body or na(avg_body) or avg_body <= 0
        true
    else
        float src = mode == "Wick" ? low : close
        (lp - src) > avg_body

// ═══════════════════════════════════════════════════════════════
//  МЕДЛЕННЫЕ ПИВОТЫ + ЛИНИИ + SMC SWING TRACKING
// ═══════════════════════════════════════════════════════════════

pivot_h = ta.pivothigh(high, swing_len, swing_len)
pivot_l = ta.pivotlow(low, swing_len, swing_len)

if not na(pivot_h)
    bh = bar_index - swing_len
    hp = high[swing_len]
    array.unshift(sh_prices, hp)
    array.unshift(sh_bars, bh)
    if array.size(sh_prices) > 20
        array.pop(sh_prices)
        array.pop(sh_bars)
    if array.size(sh_prices) >= 2
        y1 = array.get(sh_prices, 0)
        x1 = array.get(sh_bars, 0)
        y2 = array.get(sh_prices, 1)
        x2 = array.get(sh_bars, 1)
        if x1 != x2 and math.abs(x1 - x2) >= eff_min_slope
            retire_res()
            ln = line.new(x1, y1, x2, y2, xloc = xloc.bar_index, color = col_res, width = lw)
            array.unshift(res_pool, ln)
            array.unshift(res_x1, x1)
            array.unshift(res_y1, y1)
            array.unshift(res_x2, x2)
            array.unshift(res_y2, y2)
            array.unshift(res_cd, true)
    // SMC: обновить swing high
    if use_smc
        prev_sh_price := key_sh
        key_sh := hp
        key_sh_bar := bh
        sh_broken := false
        if not na(swing_h_line)
            line.delete(swing_h_line)
            swing_h_line := na

if not na(pivot_l)
    bl = bar_index - swing_len
    lp = low[swing_len]
    array.unshift(sl_prices, lp)
    array.unshift(sl_bars, bl)
    if array.size(sl_prices) > 20
        array.pop(sl_prices)
        array.pop(sl_bars)
    if array.size(sl_prices) >= 2
        y1 = array.get(sl_prices, 0)
        x1 = array.get(sl_bars, 0)
        y2 = array.get(sl_prices, 1)
        x2 = array.get(sl_bars, 1)
        if x1 != x2 and math.abs(x1 - x2) >= eff_min_slope
            retire_sup()
            ln = line.new(x1, y1, x2, y2, xloc = xloc.bar_index, color = col_sup, width = lw)
            array.unshift(sup_pool, ln)
            array.unshift(sup_x1, x1)
            array.unshift(sup_y1, y1)
            array.unshift(sup_x2, x2)
            array.unshift(sup_y2, y2)
            array.unshift(sup_cd, true)
    // SMC: обновить swing low
    if use_smc
        prev_sl_price := key_sl
        key_sl := lp
        key_sl_bar := bl
        sl_broken := false
        if not na(swing_l_line)
            line.delete(swing_l_line)
            swing_l_line := na

// ═══════════════════════════════════════════════════════════════
//  БЫСТРЫЕ ПИВОТЫ + ЛИНИИ
// ═══════════════════════════════════════════════════════════════

f_pivot_h = use_fast and fast_len < swing_len ? ta.pivothigh(high, fast_len, fast_len) : na
f_pivot_l = use_fast and fast_len < swing_len ? ta.pivotlow(low, fast_len, fast_len) : na

if not na(f_pivot_h)
    bh = bar_index - fast_len
    hp = high[fast_len]
    array.unshift(fsh_prices, hp)
    array.unshift(fsh_bars, bh)
    if array.size(fsh_prices) > 10
        array.pop(fsh_prices)
        array.pop(fsh_bars)
    if array.size(fsh_prices) >= 2
        y1 = array.get(fsh_prices, 0)
        x1 = array.get(fsh_bars, 0)
        y2 = array.get(fsh_prices, 1)
        x2 = array.get(fsh_bars, 1)
        if x1 != x2 and math.abs(x1 - x2) >= math.max(2, fast_len)
            if not na(fres_ln)
                line.delete(fres_ln)
            fres_ln := line.new(x1, y1, x2, y2, xloc = xloc.bar_index, color = color.new(col_res, 40), width = 1, style = line.style_dashed)
            fres_x1 := x1
            fres_y1 := y1
            fres_x2 := x2
            fres_y2 := y2
            fres_cd_bar := bar_index

if not na(f_pivot_l)
    bl = bar_index - fast_len
    lp = low[fast_len]
    array.unshift(fsl_prices, lp)
    array.unshift(fsl_bars, bl)
    if array.size(fsl_prices) > 10
        array.pop(fsl_prices)
        array.pop(fsl_bars)
    if array.size(fsl_prices) >= 2
        y1 = array.get(fsl_prices, 0)
        x1 = array.get(fsl_bars, 0)
        y2 = array.get(fsl_prices, 1)
        x2 = array.get(fsl_bars, 1)
        if x1 != x2 and math.abs(x1 - x2) >= math.max(2, fast_len)
            if not na(fsup_ln)
                line.delete(fsup_ln)
            fsup_ln := line.new(x1, y1, x2, y2, xloc = xloc.bar_index, color = color.new(col_sup, 40), width = 1, style = line.style_dashed)
            fsup_x1 := x1
            fsup_y1 := y1
            fsup_x2 := x2
            fsup_y2 := y2
            fsup_cd_bar := bar_index

if not na(fres_ln) and bar_index > fres_x1 + swing_len
    line.delete(fres_ln)
    fres_ln := na
if not na(fsup_ln) and bar_index > fsup_x1 + swing_len
    line.delete(fsup_ln)
    fsup_ln := na

// ═══════════════════════════════════════════════════════════════
//  ПРОДЛЕНИЕ ЛИНИЙ ВПРАВО
// ═══════════════════════════════════════════════════════════════

if barstate.islast and extend_r
    if array.size(sup_pool) > 0
        for i = 0 to array.size(sup_pool) - 1
            ln = array.get(sup_pool, i)
            if not na(ln)
                ey = calc_line_at(array.get(sup_x1, i), array.get(sup_y1, i), array.get(sup_x2, i), array.get(sup_y2, i), bar_index + 5)
                line.set_xy2(ln, bar_index + 5, ey)
    if array.size(res_pool) > 0
        for i = 0 to array.size(res_pool) - 1
            ln = array.get(res_pool, i)
            if not na(ln)
                ey = calc_line_at(array.get(res_x1, i), array.get(res_y1, i), array.get(res_x2, i), array.get(res_y2, i), bar_index + 5)
                line.set_xy2(ln, bar_index + 5, ey)
    if not na(fsup_ln)
        ey = calc_line_at(fsup_x1, fsup_y1, fsup_x2, fsup_y2, bar_index + 5)
        line.set_xy2(fsup_ln, bar_index + 5, ey)
    if not na(fres_ln)
        ey = calc_line_at(fres_x1, fres_y1, fres_x2, fres_y2, bar_index + 5)
        line.set_xy2(fres_ln, bar_index + 5, ey)

// ═══════════════════════════════════════════════════════════════
//  SMC: MITIGATION / FILL CHECK (до создания новых)
// ═══════════════════════════════════════════════════════════════

// OB mitigation
if use_smc and show_ob and ob_mitigate and array.size(ob_boxes) > 0
    for i = array.size(ob_boxes) - 1 to 0
        d = array.get(ob_dirs, i)
        t = array.get(ob_tops, i)
        b = array.get(ob_bots, i)
        mitigated = d == 1 ? close < b : close > t
        if mitigated
            box.delete(array.get(ob_boxes, i))
            array.remove(ob_boxes, i)
            array.remove(ob_tops, i)
            array.remove(ob_bots, i)
            array.remove(ob_dirs, i)

// FVG fill check
if use_smc and show_fvg and fvg_remove_filled and array.size(fvg_boxes) > 0
    for i = array.size(fvg_boxes) - 1 to 0
        d = array.get(fvg_dirs, i)
        t = array.get(fvg_tops, i)
        b = array.get(fvg_bots, i)
        filled = d == 1 ? low <= b : high >= t
        if filled
            box.delete(array.get(fvg_boxes, i))
            array.remove(fvg_boxes, i)
            array.remove(fvg_tops, i)
            array.remove(fvg_bots, i)
            array.remove(fvg_dirs, i)

// ═══════════════════════════════════════════════════════════════
//  SMC: LIQUIDITY SWEEP CHECK
// ═══════════════════════════════════════════════════════════════

if use_smc and show_liq and show_sweep and array.size(liq_lines) > 0
    for i = array.size(liq_lines) - 1 to 0
        p = array.get(liq_prices, i)
        d = array.get(liq_dirs, i)
        if d == 1 and high > p and close < p
            sweep_bull := true
            label.new(bar_index, high, "SWEEP", style = label.style_label_down,
                 color = col_liq, textcolor = color.white, size = size.tiny)
            line.delete(array.get(liq_lines, i))
            array.remove(liq_lines, i)
            array.remove(liq_prices, i)
            array.remove(liq_dirs, i)
        else if d == -1 and low < p and close > p
            sweep_bear := true
            label.new(bar_index, low, "SWEEP", style = label.style_label_up,
                 color = col_liq, textcolor = color.white, size = size.tiny)
            line.delete(array.get(liq_lines, i))
            array.remove(liq_lines, i)
            array.remove(liq_prices, i)
            array.remove(liq_dirs, i)

// ═══════════════════════════════════════════════════════════════
//  SMC: BOS / CHOCH DETECTION
// ═══════════════════════════════════════════════════════════════

if use_smc and barstate.isconfirmed
    // Bullish: close пробивает ключевой swing high
    if not na(key_sh) and not sh_broken and close > key_sh and disp_ok and mtf_bull_ok
        sh_broken := true
        smc_bull := true
        if mkt_structure == -1
            smc_is_choch_bull := true
        else
            smc_is_bos_bull := true
        mkt_structure := 1
        // Удалить линию swing high (пробита)
        if not na(swing_h_line)
            line.delete(swing_h_line)
            swing_h_line := na
    // Bearish: close пробивает ключевой swing low
    if not na(key_sl) and not sl_broken and close < key_sl and disp_ok and mtf_bear_ok
        sl_broken := true
        smc_bear := true
        if mkt_structure == 1
            smc_is_choch_bear := true
        else
            smc_is_bos_bear := true
        mkt_structure := -1
        if not na(swing_l_line)
            line.delete(swing_l_line)
            swing_l_line := na

// ═══════════════════════════════════════════════════════════════
//  SMC: ORDER BLOCK CREATION (при BOS/CHOCH)
// ═══════════════════════════════════════════════════════════════

if use_smc and show_ob
    // Bullish OB: последняя медвежья свеча перед бычьим импульсом
    if smc_bull
        for j = 1 to ob_lookback
            if close[j] < open[j]
                if array.size(ob_boxes) >= ob_max
                    box.delete(array.shift(ob_boxes))
                    array.shift(ob_tops)
                    array.shift(ob_bots)
                    array.shift(ob_dirs)
                bx = box.new(bar_index - j, high[j], bar_index + 30, low[j],
                     bgcolor = col_ob_bull, border_color = col_sup, border_width = 1)
                array.push(ob_boxes, bx)
                array.push(ob_tops, high[j])
                array.push(ob_bots, low[j])
                array.push(ob_dirs, 1)
                break
    // Bearish OB: последняя бычья свеча перед медвежьим импульсом
    if smc_bear
        for j = 1 to ob_lookback
            if close[j] > open[j]
                if array.size(ob_boxes) >= ob_max
                    box.delete(array.shift(ob_boxes))
                    array.shift(ob_tops)
                    array.shift(ob_bots)
                    array.shift(ob_dirs)
                bx = box.new(bar_index - j, high[j], bar_index + 30, low[j],
                     bgcolor = col_ob_bear, border_color = col_res, border_width = 1)
                array.push(ob_boxes, bx)
                array.push(ob_tops, high[j])
                array.push(ob_bots, low[j])
                array.push(ob_dirs, -1)
                break

// ═══════════════════════════════════════════════════════════════
//  SMC: FVG DETECTION (каждый подтверждённый бар)
// ═══════════════════════════════════════════════════════════════

if use_smc and show_fvg and barstate.isconfirmed and bar_index > 2
    // Bullish FVG: low текущего > high 2 бара назад (gap up)
    if low > high[2]
        if array.size(fvg_boxes) >= fvg_max
            box.delete(array.shift(fvg_boxes))
            array.shift(fvg_tops)
            array.shift(fvg_bots)
            array.shift(fvg_dirs)
        ft = low
        fb = high[2]
        bx = box.new(bar_index - 2, ft, bar_index + 20, fb,
             bgcolor = col_fvg_bull, border_color = color.new(col_sup, 70), border_width = 1)
        array.push(fvg_boxes, bx)
        array.push(fvg_tops, ft)
        array.push(fvg_bots, fb)
        array.push(fvg_dirs, 1)
    // Bearish FVG: high текущего < low 2 бара назад (gap down)
    if high < low[2]
        if array.size(fvg_boxes) >= fvg_max
            box.delete(array.shift(fvg_boxes))
            array.shift(fvg_tops)
            array.shift(fvg_bots)
            array.shift(fvg_dirs)
        ft = low[2]
        fb = high
        bx = box.new(bar_index - 2, ft, bar_index + 20, fb,
             bgcolor = col_fvg_bear, border_color = color.new(col_res, 70), border_width = 1)
        array.push(fvg_boxes, bx)
        array.push(fvg_tops, ft)
        array.push(fvg_bots, fb)
        array.push(fvg_dirs, -1)

// ═══════════════════════════════════════════════════════════════
//  SMC: LIQUIDITY DETECTION (equal highs / equal lows)
// ═══════════════════════════════════════════════════════════════

if use_smc and show_liq and not na(atr)
    // Equal Highs: текущий SH ≈ предыдущий SH
    if not na(pivot_h) and not na(prev_sh_price) and not na(key_sh)
        eq_thresh = atr * eq_atr_pct
        if math.abs(key_sh - prev_sh_price) < eq_thresh
            if array.size(liq_lines) >= liq_max
                line.delete(array.shift(liq_lines))
                array.shift(liq_prices)
                array.shift(liq_dirs)
            avg_p = (key_sh + prev_sh_price) / 2.0
            ln = line.new(key_sh_bar, avg_p, bar_index, avg_p,
                 color = col_liq, style = line.style_dotted, width = 2)
            array.push(liq_lines, ln)
            array.push(liq_prices, avg_p)
            array.push(liq_dirs, 1)
    // Equal Lows: текущий SL ≈ предыдущий SL
    if not na(pivot_l) and not na(prev_sl_price) and not na(key_sl)
        eq_thresh = atr * eq_atr_pct
        if math.abs(key_sl - prev_sl_price) < eq_thresh
            if array.size(liq_lines) >= liq_max
                line.delete(array.shift(liq_lines))
                array.shift(liq_prices)
                array.shift(liq_dirs)
            avg_p = (key_sl + prev_sl_price) / 2.0
            ln = line.new(key_sl_bar, avg_p, bar_index, avg_p,
                 color = col_liq, style = line.style_dotted, width = 2)
            array.push(liq_lines, ln)
            array.push(liq_prices, avg_p)
            array.push(liq_dirs, -1)

// ═══════════════════════════════════════════════════════════════
//  SMC: SWING LEVEL LINES + BOX EXTENSION
// ═══════════════════════════════════════════════════════════════

if use_smc and show_swing_lvl
    // Swing High line
    if not na(key_sh) and not sh_broken
        if na(swing_h_line)
            swing_h_line := line.new(key_sh_bar, key_sh, bar_index, key_sh,
                 color = color.new(col_res, 30), style = line.style_dashed, width = 1)
        else
            line.set_x2(swing_h_line, bar_index)
    else if not na(swing_h_line)
        line.delete(swing_h_line)
        swing_h_line := na
    // Swing Low line
    if not na(key_sl) and not sl_broken
        if na(swing_l_line)
            swing_l_line := line.new(key_sl_bar, key_sl, bar_index, key_sl,
                 color = color.new(col_sup, 30), style = line.style_dashed, width = 1)
        else
            line.set_x2(swing_l_line, bar_index)
    else if not na(swing_l_line)
        line.delete(swing_l_line)
        swing_l_line := na

// Extend OB boxes
if use_smc and show_ob and array.size(ob_boxes) > 0
    for i = 0 to array.size(ob_boxes) - 1
        box.set_right(array.get(ob_boxes, i), bar_index + 5)

// Extend FVG boxes
if use_smc and show_fvg and array.size(fvg_boxes) > 0
    for i = 0 to array.size(fvg_boxes) - 1
        box.set_right(array.get(fvg_boxes, i), bar_index + 5)

// Extend Liquidity lines
if use_smc and show_liq and array.size(liq_lines) > 0
    for i = 0 to array.size(liq_lines) - 1
        line.set_x2(array.get(liq_lines, i), bar_index + 5)

// ═══════════════════════════════════════════════════════════════
//  МЕДЛЕННЫЕ: ОБНАРУЖЕНИЕ ПРОБОЯ
// ═══════════════════════════════════════════════════════════════

min_dist = na(atr) ? 0.0 : atr_flt * atr

if pend_bear_key >= 0
    if array.size(sup_pool) == 0
        pend_bear_key := -1
        pend_bear_cnt := 0
        pend_bear_lb  := -1
    else
        found = false
        for i = 0 to array.size(sup_pool) - 1
            if array.get(sup_x1, i) == pend_bear_key
                found := true
                break
        if not found
            pend_bear_key := -1
            pend_bear_cnt := 0
            pend_bear_lb  := -1

if pend_bull_key >= 0
    if array.size(res_pool) == 0
        pend_bull_key := -1
        pend_bull_cnt := 0
        pend_bull_lb  := -1
    else
        found = false
        for i = 0 to array.size(res_pool) - 1
            if array.get(res_x1, i) == pend_bull_key
                found := true
                break
        if not found
            pend_bull_key := -1
            pend_bull_cnt := 0
            pend_bull_lb  := -1

int   new_bear_idx = -1
float closest_bear = na
float bdist_bear   = na

if array.size(sup_pool) > 0
    for i = 0 to array.size(sup_pool) - 1
        if array.get(sup_cd, i)
            array.set(sup_cd, i, false)
            continue
        sx1 = array.get(sup_x1, i)
        if bar_index <= sx1
            continue
        lp = calc_line_at(sx1, array.get(sup_y1, i), array.get(sup_x2, i), array.get(sup_y2, i), bar_index)
        wd = low < lp - min_dist
        cld = close < lp - min_dist
        trig = mode == "Wick" ? wd : mode == "Close" ? cld : (wd or cld)
        if trig
            d = lp - low
            if na(closest_bear) or lp > closest_bear
                closest_bear := lp
                bdist_bear := d
                new_bear_idx := i

bool inst_bear = inst_conf_atr > 0 and not na(bdist_bear) and not na(atr) and bdist_bear >= inst_conf_atr * atr
int  eff_cb   = inst_bear ? 1 : confirm_bars

if new_bear_idx >= 0 and vol_ok and body_bear_ok(closest_bear) and mom_bear_ok and trend_bear_ok
    lk = array.get(sup_x1, new_bear_idx)
    if lk == pend_bear_key
        if bar_index > pend_bear_lb
            pend_bear_cnt += 1
            pend_bear_lb := bar_index
    else
        pend_bear_key := lk
        pend_bear_cnt := 1
        pend_bear_lb  := bar_index
        bear_tent := true
    if pend_bear_cnt >= eff_cb and barstate.isconfirmed
        bear_break := true
        remove_sup(new_bear_idx)
        pend_bear_key := -1
        pend_bear_cnt := 0
        pend_bear_lb  := -1
else if pend_bear_key >= 0 and bar_index > pend_bear_lb and barstate.isconfirmed
    pend_bear_key := -1
    pend_bear_cnt := 0
    pend_bear_lb  := -1

int   new_bull_idx = -1
float closest_bull = na
float bdist_bull   = na

if array.size(res_pool) > 0
    for i = 0 to array.size(res_pool) - 1
        if array.get(res_cd, i)
            array.set(res_cd, i, false)
            continue
        rx1 = array.get(res_x1, i)
        if bar_index <= rx1
            continue
        lp = calc_line_at(rx1, array.get(res_y1, i), array.get(res_x2, i), array.get(res_y2, i), bar_index)
        wu = high > lp + min_dist
        clu = close > lp + min_dist
        trig = mode == "Wick" ? wu : mode == "Close" ? clu : (wu or clu)
        if trig
            d = high - lp
            if na(closest_bull) or lp < closest_bull
                closest_bull := lp
                bdist_bull := d
                new_bull_idx := i

bool inst_bull = inst_conf_atr > 0 and not na(bdist_bull) and not na(atr) and bdist_bull >= inst_conf_atr * atr
int  eff_cbl   = inst_bull ? 1 : confirm_bars

if new_bull_idx >= 0 and vol_ok and body_bull_ok(closest_bull) and mom_bull_ok and trend_bull_ok
    lk = array.get(res_x1, new_bull_idx)
    if lk == pend_bull_key
        if bar_index > pend_bull_lb
            pend_bull_cnt += 1
            pend_bull_lb := bar_index
    else
        pend_bull_key := lk
        pend_bull_cnt := 1
        pend_bull_lb  := bar_index
        bull_tent := true
    if pend_bull_cnt >= eff_cbl and barstate.isconfirmed
        bull_break := true
        remove_res(new_bull_idx)
        pend_bull_key := -1
        pend_bull_cnt := 0
        pend_bull_lb  := -1
else if pend_bull_key >= 0 and bar_index > pend_bull_lb and barstate.isconfirmed
    pend_bull_key := -1
    pend_bull_cnt := 0
    pend_bull_lb  := -1

// ═══════════════════════════════════════════════════════════════
//  БЫСТРЫЕ ЛИНИИ: ОБНАРУЖЕНИЕ ПРОБОЯ
// ═══════════════════════════════════════════════════════════════

if use_fast and fast_len < swing_len
    if not na(fsup_ln) and bar_index > fsup_cd_bar and bar_index > fsup_x1
        flp = calc_line_at(fsup_x1, fsup_y1, fsup_x2, fsup_y2, bar_index)
        fwd = low < flp - min_dist
        fcd = close < flp - min_dist
        ftrig = mode == "Wick" ? fwd : mode == "Close" ? fcd : (fwd or fcd)
        if ftrig and vol_ok and body_bear_ok(flp) and mom_bear_ok and trend_bear_ok and barstate.isconfirmed
            fast_bear := true
            line.delete(fsup_ln)
            fsup_ln := na

    if not na(fres_ln) and bar_index > fres_cd_bar and bar_index > fres_x1
        flp = calc_line_at(fres_x1, fres_y1, fres_x2, fres_y2, bar_index)
        fwu = high > flp + min_dist
        fcu = close > flp + min_dist
        ftrig = mode == "Wick" ? fwu : mode == "Close" ? fcu : (fwu or fcu)
        if ftrig and vol_ok and body_bull_ok(flp) and mom_bull_ok and trend_bull_ok and barstate.isconfirmed
            fast_bull := true
            line.delete(fres_ln)
            fres_ln := na

// ═══════════════════════════════════════════════════════════════
//  ОТРИСОВКА СИГНАЛОВ
// ═══════════════════════════════════════════════════════════════

// Trendline breaks
plotshape(bull_break, title = "Bull Break", style = shape.labelup, location = location.belowbar,
     color = #00BFA5, textcolor = color.white, text = "BREAK UP", size = size.normal)

plotshape(bear_break, title = "Bear Break", style = shape.labeldown, location = location.abovebar,
     color = #FF1744, textcolor = color.white, text = "BREAK DN", size = size.normal)

plotshape(show_fast and fast_bull and not bull_break, title = "Fast Bull", style = shape.labelup,
     location = location.belowbar, color = #2979FF, textcolor = color.white, text = "FAST UP", size = size.small)

plotshape(show_fast and fast_bear and not bear_break, title = "Fast Bear", style = shape.labeldown,
     location = location.abovebar, color = #FF9100, textcolor = color.white, text = "FAST DN", size = size.small)

plotshape(show_tent and bull_tent and not bull_break, title = "Bull Tent", style = shape.triangleup,
     location = location.belowbar, color = color.new(#00BFA5, 50), size = size.small)

plotshape(show_tent and bear_tent and not bear_break, title = "Bear Tent", style = shape.triangledown,
     location = location.abovebar, color = color.new(#FF1744, 50), size = size.small)

bgcolor(bull_break and not bear_break and show_bg ? bg_up : na, title = "Bull BG")
bgcolor(bear_break and not bull_break and show_bg ? bg_dn : na, title = "Bear BG")

// SMC: BOS signals
plotshape(use_smc and show_bos and smc_is_bos_bull, title = "BOS Bull", style = shape.labelup,
     location = location.belowbar, color = col_bos, textcolor = color.white, text = "BOS", size = size.small)

plotshape(use_smc and show_bos and smc_is_bos_bear, title = "BOS Bear", style = shape.labeldown,
     location = location.abovebar, color = col_bos, textcolor = color.white, text = "BOS", size = size.small)

// SMC: CHOCH signals
plotshape(use_smc and show_choch and smc_is_choch_bull, title = "CHOCH Bull", style = shape.labelup,
     location = location.belowbar, color = col_choch, textcolor = color.white, text = "CHOCH", size = size.normal)

plotshape(use_smc and show_choch and smc_is_choch_bear, title = "CHOCH Bear", style = shape.labeldown,
     location = location.abovebar, color = col_choch, textcolor = color.white, text = "CHOCH", size = size.normal)

// SMC: Displacement marker
plotshape(use_smc and is_displacement and close > open, title = "Disp Bull",
     style = shape.diamond, location = location.belowbar, color = color.new(col_bos, 60), size = size.tiny)
plotshape(use_smc and is_displacement and close < open, title = "Disp Bear",
     style = shape.diamond, location = location.abovebar, color = color.new(col_choch, 60), size = size.tiny)

// ═══════════════════════════════════════════════════════════════
//  PREMIUM / DISCOUNT ZONES
// ═══════════════════════════════════════════════════════════════

pd_high = key_sh
pd_low  = key_sl
pd_mid  = not na(pd_high) and not na(pd_low) ? (pd_high + pd_low) / 2.0 : na

in_premium  = not na(pd_mid) and close > pd_mid
in_discount = not na(pd_mid) and close < pd_mid

bgcolor(use_smc and show_pd and in_premium ? col_premium : na, title = "Premium Zone")
bgcolor(use_smc and show_pd and in_discount ? col_discount : na, title = "Discount Zone")
plot(use_smc and show_pd and not na(pd_mid) ? pd_mid : na, "Equilibrium",
     color = color.new(color.gray, 50), style = plot.style_cross)

// ═══════════════════════════════════════════════════════════════
//  ИНФОРМАЦИОННАЯ ПАНЕЛЬ
// ═══════════════════════════════════════════════════════════════

var table tbl = table.new(position.top_right, 2, 12,
     bgcolor = color.new(#1E1E1E, 10), border_color = color.new(color.gray, 60), border_width = 1)

if barstate.islast
    sc = array.size(sup_pool)
    rc = array.size(res_pool)

    float bs = na
    if sc > 0
        for i = 0 to sc - 1
            p = calc_line_at(array.get(sup_x1, i), array.get(sup_y1, i), array.get(sup_x2, i), array.get(sup_y2, i), bar_index)
            if na(bs) or p > bs
                bs := p

    float br = na
    if rc > 0
        for i = 0 to rc - 1
            p = calc_line_at(array.get(res_x1, i), array.get(res_y1, i), array.get(res_x2, i), array.get(res_y2, i), bar_index)
            if na(br) or p < br
                br := p

    table.cell(tbl, 0, 0, "TBD v6+SMC", text_color = color.white, text_size = size.small)
    table.cell(tbl, 1, 0, barstate.isrealtime ? "LIVE" : "HIST", text_color = barstate.isrealtime ? #00BFA5 : color.gray, text_size = size.small)

    table.cell(tbl, 0, 1, "Lines", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 1, str.tostring(sc) + " S / " + str.tostring(rc) + " R", text_color = color.white, text_size = size.small)

    string tfm = trend_mode == "EMA Cross" ? "Cross" : trend_mode == "Price vs EMA" ? "Price" : "OFF"
    bool bt = trend_bull_ok
    table.cell(tbl, 0, 2, "Trend", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 2, trend_mode == "Off" ? "OFF" : (bt ? tfm + ":BULL" : tfm + ":BEAR"),
         text_color = trend_mode == "Off" ? color.gray : (bt ? col_sup : col_res), text_size = size.small)

    table.cell(tbl, 0, 3, "Sup", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 3, not na(bs) ? str.tostring(bs, format.mintick) : "---", text_color = not na(bs) ? col_sup : color.gray, text_size = size.small)

    table.cell(tbl, 0, 4, "Res", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 4, not na(br) ? str.tostring(br, format.mintick) : "---", text_color = not na(br) ? col_res : color.gray, text_size = size.small)

    table.cell(tbl, 0, 5, "Momentum", text_color = color.gray, text_size = size.small)
    string mm = mom_mode == "Off" ? "OFF" : mom_mode + (mom_bull_ok ? " B" : " S")
    table.cell(tbl, 1, 5, mm, text_color = mom_mode == "Off" ? color.gray : (mom_bull_ok ? col_sup : col_res), text_size = size.small)

    table.cell(tbl, 0, 6, "Confirm", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 6, str.tostring(confirm_bars) + "b" + (inst_conf_atr > 0 ? " IC" : ""), text_color = color.gray, text_size = size.small)

    table.cell(tbl, 0, 7, "Vol Proj", text_color = color.gray, text_size = size.small)
    string vp = vp_active ? str.tostring(math.round(vol_pct * 100)) + "%" : (use_vol_proj ? "ON" : "OFF")
    table.cell(tbl, 1, 7, vp, text_color = vp_active ? color.yellow : color.gray, text_size = size.small)

    // SMC rows
    table.cell(tbl, 0, 8, "Structure", text_color = color.gray, text_size = size.small)
    string struct_txt = mkt_structure == 1 ? "BULL" : mkt_structure == -1 ? "BEAR" : "---"
    color  struct_col = mkt_structure == 1 ? col_sup : mkt_structure == -1 ? col_res : color.gray
    table.cell(tbl, 1, 8, use_smc ? struct_txt : "OFF", text_color = use_smc ? struct_col : color.gray, text_size = size.small)

    table.cell(tbl, 0, 9, "Key SH", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 9, use_smc and not na(key_sh) ? str.tostring(key_sh, format.mintick) + (sh_broken ? " X" : "") : "---",
         text_color = sh_broken ? color.gray : col_res, text_size = size.small)

    table.cell(tbl, 0, 10, "Key SL", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 10, use_smc and not na(key_sl) ? str.tostring(key_sl, format.mintick) + (sl_broken ? " X" : "") : "---",
         text_color = sl_broken ? color.gray : col_sup, text_size = size.small)

    table.cell(tbl, 0, 11, "Zone", text_color = color.gray, text_size = size.small)
    string zone_txt = not use_smc or na(pd_mid) ? "---" : in_premium ? "PREMIUM" : in_discount ? "DISCOUNT" : "EQ"
    color  zone_col = not use_smc or na(pd_mid) ? color.gray : in_premium ? col_res : in_discount ? col_sup : color.gray
    table.cell(tbl, 1, 11, zone_txt, text_color = zone_col, text_size = size.small)

// ═══════════════════════════════════════════════════════════════
//  АЛЕРТЫ
// ═══════════════════════════════════════════════════════════════

// Trendline break alerts
alertcondition(bull_break and alr_bull, "Bullish Break",
     "Bullish break on {{ticker}} {{interval}} at {{close}}")

alertcondition(bear_break and alr_bear, "Bearish Break",
     "Bearish break on {{ticker}} {{interval}} at {{close}}")

alertcondition((bull_break and alr_bull) or (bear_break and alr_bear), "Any Break",
     "Break on {{ticker}} {{interval}} at {{close}}")

alertcondition((fast_bull or fast_bear) and alr_fast and not bull_break and not bear_break, "Fast Break (early)",
     "Fast break on {{ticker}} {{interval}} at {{close}} — early signal")

alertcondition(((bull_tent and not bull_break) or (bear_tent and not bear_break)) and alr_tent, "Tentative Break",
     "Tentative on {{ticker}} {{interval}} at {{close}}")

// SMC alerts
alertcondition((smc_is_bos_bull or smc_is_bos_bear) and alr_bos, "BOS",
     "BOS on {{ticker}} {{interval}} at {{close}}")

alertcondition((smc_is_choch_bull or smc_is_choch_bear) and alr_choch, "CHOCH",
     "CHOCH on {{ticker}} {{interval}} at {{close}}")

alertcondition((sweep_bull or sweep_bear) and alr_sweep, "Liquidity Sweep",
     "Liquidity sweep on {{ticker}} {{interval}} at {{close}}")

alertcondition(smc_is_choch_bull and alr_choch, "CHOCH Bullish",
     "Bullish CHOCH on {{ticker}} {{interval}} at {{close}}")

alertcondition(smc_is_choch_bear and alr_choch, "CHOCH Bearish",
     "Bearish CHOCH on {{ticker}} {{interval}} at {{close}}")
