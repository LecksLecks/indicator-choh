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
bool   auto_swing_len = input.bool(true, "Auto Scale by Timeframe", group = GRP_SWING,
     tooltip = "Автомасштаб: 1H→10, 4H→8, 15m→8. Отключите для ручного управления.")

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
bool   show_swing_trail = input.bool(true, "Show HH/HL/LH/LL Labels", group = GRP_SMC)
int    smc_confirm = input.int(1, "SMC Confirmation Bars", minval = 1, maxval = 5, group = GRP_SMC)
bool   show_sfp  = input.bool(true, "Show SFP (Swing Failure)", group = GRP_SMC,
     tooltip = "Тень пробивает swing level, но close возвращается.")
bool   show_div  = input.bool(true, "Show Divergence at Pivots", group = GRP_SMC,
     tooltip = "RSI-дивергенция: цена HH но RSI LH (медвежья), цена LL но RSI HL (бычья).")
color  col_bos   = input.color(#2962FF, "BOS Color", group = GRP_SMC)
color  col_choch = input.color(#E040FB, "CHOCH Color", group = GRP_SMC)
float  smc_min_swing = input.float(0.5, "Min Swing Size (ATR x)", minval = 0.0, step = 0.1, group = GRP_SMC,
     tooltip = "Мин. размах свинга для BOS/CHOCH. 0=любой. Фильтрует шум в рейндже.")
int    smc_min_conf  = input.int(2, "Min Confluence for Labels", minval = 0, maxval = 5, group = GRP_SMC,
     tooltip = "BOS/CHOCH лейблы только при confluence >= этого значения. 0=все.")

string GRP_DISP  = "══ Displacement ══"
float  disp_mul  = input.float(1.5, "Displacement Body Multiplier", minval = 1.0, step = 0.1, group = GRP_DISP,
     tooltip = "Свеча импульсная если тело > avg_body x этот множитель.")
bool   req_disp  = input.bool(false, "Require Displacement for SMC Break", group = GRP_DISP)

string GRP_OB    = "══ Order Blocks ══"
bool   show_ob   = input.bool(true, "Show Order Blocks", group = GRP_OB)
int    ob_max    = input.int(5, "Max Active Order Blocks", minval = 1, maxval = 20, group = GRP_OB)
int    ob_lookback = input.int(10, "OB Lookback Bars", minval = 3, maxval = 30, group = GRP_OB)
bool   ob_mitigate = input.bool(true, "Remove Mitigated OBs", group = GRP_OB)
bool   ob_use_body = input.bool(false, "OB: Use Body Only (refine)", group = GRP_OB)
bool   show_breaker = input.bool(true, "Show Breaker Blocks", group = GRP_OB)
color  col_ob_bull = input.color(color.new(#00BFA5, 80), "Bullish OB Color", group = GRP_OB)
color  col_ob_bear = input.color(color.new(#FF1744, 80), "Bearish OB Color", group = GRP_OB)

string GRP_FVG   = "══ Fair Value Gaps ══"
bool   show_fvg  = input.bool(true, "Show FVG Zones", group = GRP_FVG)
int    fvg_max   = input.int(5, "Max Active FVGs", minval = 1, maxval = 20, group = GRP_FVG)
bool   fvg_remove_filled = input.bool(true, "Remove Filled FVGs", group = GRP_FVG)
float  fvg_min_atr = input.float(0.2, "FVG Min Size (ATR x)", minval = 0.0, step = 0.05, group = GRP_FVG)
color  col_fvg_bull = input.color(color.new(#00BFA5, 85), "Bullish FVG Color", group = GRP_FVG)
color  col_fvg_bear = input.color(color.new(#FF1744, 85), "Bearish FVG Color", group = GRP_FVG)

string GRP_LIQ   = "══ Liquidity ══"
bool   show_liq  = input.bool(true, "Show Equal H/L Liquidity", group = GRP_LIQ)
float  eq_atr_pct = input.float(0.1, "Equal Level ATR Threshold", minval = 0.01, step = 0.01, group = GRP_LIQ)
bool   show_sweep = input.bool(true, "Mark Liquidity Sweeps", group = GRP_LIQ)
bool   show_induce = input.bool(true, "Show Inducement (IDM)", group = GRP_LIQ)
color  col_liq   = input.color(color.new(#FFD600, 50), "Liquidity Level Color", group = GRP_LIQ)
int    liq_max   = input.int(5, "Max Liquidity Levels", minval = 1, maxval = 10, group = GRP_LIQ)

string GRP_SESS  = "══ Sessions ══"
bool   show_sessions = input.bool(false, "Show Session Ranges", group = GRP_SESS)
bool   show_asian_sess = input.bool(true, "Asian Session", group = GRP_SESS)
string sess_asian = input.string("0000-0800", "Asian Time (UTC)", group = GRP_SESS)
color  col_asian = input.color(color.new(#FFD600, 88), "Asian Color", group = GRP_SESS)
bool   show_london_sess = input.bool(false, "London Session", group = GRP_SESS)
string sess_london = input.string("0800-1600", "London Time (UTC)", group = GRP_SESS)
color  col_london = input.color(color.new(#2962FF, 88), "London Color", group = GRP_SESS)
bool   show_ny_sess = input.bool(false, "NY Session", group = GRP_SESS)
string sess_ny = input.string("1300-2100", "NY Time (UTC)", group = GRP_SESS)
color  col_ny = input.color(color.new(#00BFA5, 88), "NY Color", group = GRP_SESS)

string GRP_KZ    = "══ Kill Zones ══"
bool   show_kz   = input.bool(false, "Show Kill Zones", group = GRP_KZ,
     tooltip = "Подсветка фона в окнах макс. институциональной активности.")
color  col_kz    = input.color(color.new(#FF6D00, 92), "KZ Background", group = GRP_KZ)
bool   use_time_filter = input.bool(false, "No-Trade Time Filter", group = GRP_KZ)
string no_trade_time = input.string("2100-0000", "No-Trade Window (UTC)", group = GRP_KZ,
     tooltip = "В это время composite-сигналы подавляются.")

string GRP_PD    = "══ Premium / Discount ══"
bool   show_pd   = input.bool(false, "Show Premium/Discount Zones", group = GRP_PD)
color  col_premium = input.color(color.new(#FF1744, 93), "Premium Zone Color", group = GRP_PD)
color  col_discount = input.color(color.new(#00BFA5, 93), "Discount Zone Color", group = GRP_PD)

string GRP_MTF   = "══ Multi-Timeframe ══"
bool   use_mtf   = input.bool(false, "Enable MTF Confirmation", group = GRP_MTF)
string htf       = input.timeframe("15", "Higher Timeframe", group = GRP_MTF)

string GRP_SIGNAL = "══ Composite Signal ══"
bool   show_ote  = input.bool(true, "Show OTE Zone (Fib 0.618-0.786)", group = GRP_SIGNAL,
     tooltip = "Optimal Trade Entry — зона отката по Фибоначчи после BOS/CHOCH.")
bool   show_composite = input.bool(true, "Show Composite BUY/SELL", group = GRP_SIGNAL)
int    composite_min_conf = input.int(2, "Min Confluence for Signal", minval = 1, maxval = 5, group = GRP_SIGNAL)
float  tp_atr_mul = input.float(2.5, "TP ATR Multiplier", minval = 0.5, step = 0.5, group = GRP_SIGNAL,
     tooltip = "TP = entry ± ATR × этот множитель. Реалистичнее чем key_sh/key_sl на старших ТФ.")
float  sl_atr_buf_mul = input.float(1.0, "SL ATR Buffer", minval = 0.1, step = 0.1, group = GRP_SIGNAL,
     tooltip = "Буфер под/над OTE зоной для SL. Больше = меньше стоп-аутов.")
int    composite_cooldown = input.int(10, "Signal Cooldown (bars)", minval = 1, maxval = 100, group = GRP_SIGNAL,
     tooltip = "Мин. баров между composite-сигналами. Снижает частоту сигналов.")
bool   show_sltp = input.bool(true, "Show SL/TP Lines", group = GRP_SIGNAL)
bool   show_stats = input.bool(true, "Show Backtest Stats", group = GRP_SIGNAL)

string GRP_VIS   = "══ Visuals ══"
bool   show_bg   = input.bool(true, "Highlight Break Bars", group = GRP_VIS)
color  bg_up     = input.color(color.new(#00BFA5, 90), "Bull BG Color", group = GRP_VIS)
color  bg_dn     = input.color(color.new(#FF1744, 90), "Bear BG Color", group = GRP_VIS)
bool   show_tent = input.bool(false, "Show Tentative Signals", group = GRP_VIS)
bool   show_fast = input.bool(true, "Show Fast Break Signals", group = GRP_VIS)
bool   show_ema  = input.bool(true, "Show EMA Cloud", group = GRP_VIS)
int    ema_fl    = input.int(20, "EMA Fast", group = GRP_VIS)
int    ema_sl    = input.int(50, "EMA Slow", group = GRP_VIS)
bool   use_candle_color = input.bool(false, "Color Candles by Structure", group = GRP_VIS)

string GRP_MULTI = "══ Multi-Symbol ══"
bool   show_multi = input.bool(false, "Show Multi-Symbol Panel", group = GRP_MULTI)
string sym1_input = input.symbol("BINANCE:BTCUSDT", "Symbol 1", group = GRP_MULTI)
string sym2_input = input.symbol("BINANCE:ETHUSDT", "Symbol 2", group = GRP_MULTI)

string GRP_ALR   = "══ Alerts ══"
bool   alr_bull  = input.bool(true, "Alert: Bullish Break", group = GRP_ALR)
bool   alr_bear  = input.bool(true, "Alert: Bearish Break", group = GRP_ALR)
bool   alr_fast  = input.bool(true, "Alert: Fast Break", group = GRP_ALR)
bool   alr_tent  = input.bool(false, "Alert: Tentative", group = GRP_ALR)
bool   alr_bos   = input.bool(true, "Alert: BOS", group = GRP_ALR)
bool   alr_choch = input.bool(true, "Alert: CHOCH", group = GRP_ALR)
bool   alr_sweep = input.bool(true, "Alert: Liquidity Sweep", group = GRP_ALR)
bool   alr_induce = input.bool(false, "Alert: Inducement (IDM)", group = GRP_ALR)
bool   alr_composite = input.bool(true, "Alert: Composite Signal", group = GRP_ALR)
bool   alr_sfp   = input.bool(true, "Alert: SFP", group = GRP_ALR)

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

// Auto swing length scaling by timeframe
int tf_secs = timeframe.in_seconds()
int eff_swing_len = auto_swing_len ? (tf_secs >= 86400 ? swing_len : tf_secs >= 14400 ? math.max(swing_len, 8) : tf_secs >= 3600 ? math.max(swing_len, 10) : tf_secs >= 900 ? math.max(swing_len, 8) : swing_len) : swing_len

eff_min_slope = min_slope_bars > 0 ? min_slope_bars : eff_swing_len * 2

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

// Kill Zones (EST = America/New_York)
bool in_kz_london  = not na(time(timeframe.period, "0200-0500", "America/New_York"))
bool in_kz_ny_open = not na(time(timeframe.period, "0700-1000", "America/New_York"))
bool in_kz_ny_close = not na(time(timeframe.period, "1000-1100", "America/New_York"))
bool in_any_kz = in_kz_london or in_kz_ny_open or in_kz_ny_close

// Time filter
bool in_no_trade = use_time_filter and no_trade_time != "" and not na(time(timeframe.period, no_trade_time, "UTC"))

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

int fast_len = math.max(2, eff_swing_len / 2)

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

var int   mkt_structure = 0
var float key_sh = na
var int   key_sh_bar = -1
var float key_sl = na
var int   key_sl_bar = -1
var float prev_sh_price = na
var float prev_sl_price = na
var bool  sh_broken = true
var bool  sl_broken = true

var line  swing_h_line = na
var line  swing_l_line = na

var int   pend_smc_bull_key = -1
var int   pend_smc_bull_cnt = 0
var int   pend_smc_bull_lb  = -1
var int   pend_smc_bear_key = -1
var int   pend_smc_bear_cnt = 0
var int   pend_smc_bear_lb  = -1

var box[]   ob_boxes = array.new_box(0)
var float[] ob_tops  = array.new_float(0)
var float[] ob_bots  = array.new_float(0)
var int[]   ob_dirs  = array.new_int(0)
var int[]   ob_brk   = array.new_int(0)

var box[]   fvg_boxes = array.new_box(0)
var float[] fvg_tops  = array.new_float(0)
var float[] fvg_bots  = array.new_float(0)
var int[]   fvg_dirs  = array.new_int(0)

var line[]  liq_lines  = array.new_line(0)
var float[] liq_prices = array.new_float(0)
var int[]   liq_dirs   = array.new_int(0)

var line[]  idm_lines  = array.new_line(0)
var float[] idm_prices = array.new_float(0)
var int[]   idm_dirs   = array.new_int(0)

// OTE
var box   ote_box = na
var float ote_top_price = na
var float ote_bot_price = na
var int   ote_dir = 0
var bool  ote_buy_fired = false
var bool  ote_sell_fired = false

// SL/TP lines
var line  sl_line = na
var line  tp_line = na

// Divergence
var float prev_pivot_h_rsi = na
var float prev_pivot_l_rsi = na

// Backtest
var int   bt_total = 0
var int   bt_wins = 0
var int   bt_losses = 0
var float bt_entry = na
var float bt_sl = na
var float bt_tp = na
var int   bt_dir = 0
var bool  bt_in_trade = false
var float bt_last_rr = 0.0
var float bt_sum_rr = 0.0
var int   last_composite_bar = -1000
var bool  bt_trailed = false

// Sessions
var float asian_hi = na
var float asian_lo = na
var int   asian_start = -1
var box   asian_box = na
var float london_hi = na
var float london_lo = na
var int   london_start = -1
var box   london_box = na
var float ny_hi = na
var float ny_lo = na
var int   ny_start = -1
var box   ny_box = na

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
idm_swept  = false
sfp_bull   = false
sfp_bear   = false
div_bull   = false
div_bear   = false
composite_buy  = false
composite_sell = false

// ═══════════════════════════════════════════════════════════════
//  MTF + MULTI-SYMBOL (request.security — global)
// ═══════════════════════════════════════════════════════════════

htf_struct = request.security(syminfo.tickerid, htf, mkt_structure, barmerge.gaps_off, barmerge.lookahead_off)
int htf_s = use_mtf and not na(htf_struct) ? htf_struct : 0
bool mtf_bull_ok = not use_mtf or htf_s >= 0
bool mtf_bear_ok = not use_mtf or htf_s <= 0

[sym1_ef, sym1_es] = request.security(sym1_input, timeframe.period, [ta.ema(close, ema_fl), ta.ema(close, ema_sl)], barmerge.gaps_off, barmerge.lookahead_off)
[sym2_ef, sym2_es] = request.security(sym2_input, timeframe.period, [ta.ema(close, ema_fl), ta.ema(close, ema_sl)], barmerge.gaps_off, barmerge.lookahead_off)

bool sym1_bull = not na(sym1_ef) and not na(sym1_es) and sym1_ef > sym1_es
bool sym2_bull = not na(sym2_ef) and not na(sym2_es) and sym2_ef > sym2_es

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
//  МЕДЛЕННЫЕ ПИВОТЫ + ЛИНИИ + SMC SWING + DIVERGENCE
// ═══════════════════════════════════════════════════════════════

pivot_h = ta.pivothigh(high, eff_swing_len, eff_swing_len)
pivot_l = ta.pivotlow(low, eff_swing_len, eff_swing_len)

if not na(pivot_h)
    bh = bar_index - eff_swing_len
    hp = high[eff_swing_len]
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
    if use_smc
        bool is_hh = na(key_sh) or hp > key_sh
        // Divergence: HH in price but LH in RSI
        if show_div and not na(prev_sh_price) and not na(prev_pivot_h_rsi) and not na(rsi_val[eff_swing_len])
            float cur_rsi = rsi_val[eff_swing_len]
            if hp > prev_sh_price and cur_rsi < prev_pivot_h_rsi
                div_bear := true
                label.new(bh, hp, "DIV", style = label.style_label_down,
                     color = color.new(col_res, 50), textcolor = col_res, size = size.tiny)
        prev_pivot_h_rsi := rsi_val[eff_swing_len]
        prev_sh_price := key_sh
        key_sh := hp
        key_sh_bar := bh
        sh_broken := false
        pend_smc_bull_key := -1
        pend_smc_bull_cnt := 0
        pend_smc_bull_lb  := -1
        if not na(swing_h_line)
            line.delete(swing_h_line)
            swing_h_line := na
        if show_swing_trail
            string sh_txt = is_hh ? "HH" : "LH"
            color  sh_col = is_hh ? col_sup : col_res
            label.new(bh, hp, sh_txt, style = label.style_label_down,
                 color = color.new(sh_col, 70), textcolor = sh_col, size = size.tiny)

if not na(pivot_l)
    bl = bar_index - eff_swing_len
    lp = low[eff_swing_len]
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
    if use_smc
        bool is_hl = na(key_sl) or lp > key_sl
        // Divergence: LL in price but HL in RSI
        if show_div and not na(prev_sl_price) and not na(prev_pivot_l_rsi) and not na(rsi_val[eff_swing_len])
            float cur_rsi = rsi_val[eff_swing_len]
            if lp < prev_sl_price and cur_rsi > prev_pivot_l_rsi
                div_bull := true
                label.new(bl, lp, "DIV", style = label.style_label_up,
                     color = color.new(col_sup, 50), textcolor = col_sup, size = size.tiny)
        prev_pivot_l_rsi := rsi_val[eff_swing_len]
        prev_sl_price := key_sl
        key_sl := lp
        key_sl_bar := bl
        sl_broken := false
        pend_smc_bear_key := -1
        pend_smc_bear_cnt := 0
        pend_smc_bear_lb  := -1
        if not na(swing_l_line)
            line.delete(swing_l_line)
            swing_l_line := na
        if show_swing_trail
            string sl_txt = is_hl ? "HL" : "LL"
            color  sl_col = is_hl ? col_sup : col_res
            label.new(bl, lp, sl_txt, style = label.style_label_up,
                 color = color.new(sl_col, 70), textcolor = sl_col, size = size.tiny)

// ═══════════════════════════════════════════════════════════════
//  БЫСТРЫЕ ПИВОТЫ + ЛИНИИ + INDUCEMENT
// ═══════════════════════════════════════════════════════════════

f_pivot_h = use_fast and fast_len < eff_swing_len ? ta.pivothigh(high, fast_len, fast_len) : na
f_pivot_l = use_fast and fast_len < eff_swing_len ? ta.pivotlow(low, fast_len, fast_len) : na

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
    if use_smc and show_induce and not na(key_sh) and not na(key_sl) and not na(atr)
        if hp < key_sh and hp > key_sl
            if array.size(idm_lines) >= liq_max
                line.delete(array.shift(idm_lines))
                array.shift(idm_prices)
                array.shift(idm_dirs)
            iln = line.new(bh, hp, bar_index, hp,
                 color = color.new(col_liq, 60), style = line.style_dotted, width = 1)
            array.push(idm_lines, iln)
            array.push(idm_prices, hp)
            array.push(idm_dirs, 1)

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
    if use_smc and show_induce and not na(key_sh) and not na(key_sl) and not na(atr)
        if lp > key_sl and lp < key_sh
            if array.size(idm_lines) >= liq_max
                line.delete(array.shift(idm_lines))
                array.shift(idm_prices)
                array.shift(idm_dirs)
            iln = line.new(bl, lp, bar_index, lp,
                 color = color.new(col_liq, 60), style = line.style_dotted, width = 1)
            array.push(idm_lines, iln)
            array.push(idm_prices, lp)
            array.push(idm_dirs, -1)

if not na(fres_ln) and bar_index > fres_x1 + eff_swing_len
    line.delete(fres_ln)
    fres_ln := na
if not na(fsup_ln) and bar_index > fsup_x1 + eff_swing_len
    line.delete(fsup_ln)
    fsup_ln := na

// ═══════════════════════════════════════════════════════════════
//  SESSION TRACKING
// ═══════════════════════════════════════════════════════════════

bool in_asian_raw  = not na(time(timeframe.period, sess_asian, "UTC"))
bool in_london_raw = not na(time(timeframe.period, sess_london, "UTC"))
bool in_ny_raw     = not na(time(timeframe.period, sess_ny, "UTC"))
bool in_asian  = show_sessions and show_asian_sess and in_asian_raw
bool in_london = show_sessions and show_london_sess and in_london_raw
bool in_ny     = show_sessions and show_ny_sess and in_ny_raw

if in_asian
    if not in_asian[1] or na(asian_hi)
        asian_hi := high
        asian_lo := low
        asian_start := bar_index
        if not na(asian_box)
            box.delete(asian_box)
            asian_box := na
    else
        asian_hi := math.max(asian_hi, high)
        asian_lo := math.min(asian_lo, low)
    if na(asian_box)
        asian_box := box.new(asian_start, asian_hi, bar_index, asian_lo,
             bgcolor = col_asian, border_color = color.new(col_asian, 40), border_width = 1)
    else
        box.set_top(asian_box, asian_hi)
        box.set_bottom(asian_box, asian_lo)
        box.set_right(asian_box, bar_index)

if in_london
    if not in_london[1] or na(london_hi)
        london_hi := high
        london_lo := low
        london_start := bar_index
        if not na(london_box)
            box.delete(london_box)
            london_box := na
    else
        london_hi := math.max(london_hi, high)
        london_lo := math.min(london_lo, low)
    if na(london_box)
        london_box := box.new(london_start, london_hi, bar_index, london_lo,
             bgcolor = col_london, border_color = color.new(col_london, 40), border_width = 1)
    else
        box.set_top(london_box, london_hi)
        box.set_bottom(london_box, london_lo)
        box.set_right(london_box, bar_index)

if in_ny
    if not in_ny[1] or na(ny_hi)
        ny_hi := high
        ny_lo := low
        ny_start := bar_index
        if not na(ny_box)
            box.delete(ny_box)
            ny_box := na
    else
        ny_hi := math.max(ny_hi, high)
        ny_lo := math.min(ny_lo, low)
    if na(ny_box)
        ny_box := box.new(ny_start, ny_hi, bar_index, ny_lo,
             bgcolor = col_ny, border_color = color.new(col_ny, 40), border_width = 1)
    else
        box.set_top(ny_box, ny_hi)
        box.set_bottom(ny_box, ny_lo)
        box.set_right(ny_box, bar_index)

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
//  SMC: OB MITIGATION / BREAKER + FVG FILL
// ═══════════════════════════════════════════════════════════════

if use_smc and show_ob and ob_mitigate and array.size(ob_boxes) > 0
    for i = array.size(ob_boxes) - 1 to 0
        d = array.get(ob_dirs, i)
        t = array.get(ob_tops, i)
        b = array.get(ob_bots, i)
        is_brk = array.get(ob_brk, i)
        mitigated = d == 1 ? close < b : close > t
        if mitigated
            if is_brk == 0 and show_breaker
                new_dir = d * -1
                array.set(ob_dirs, i, new_dir)
                array.set(ob_brk, i, 1)
                new_bg = new_dir == 1 ? color.new(col_sup, 85) : color.new(col_res, 85)
                new_bc = new_dir == 1 ? col_sup : col_res
                box.set_bgcolor(array.get(ob_boxes, i), new_bg)
                box.set_border_color(array.get(ob_boxes, i), new_bc)
                box.set_border_style(array.get(ob_boxes, i), line.style_dashed)
            else
                box.delete(array.get(ob_boxes, i))
                array.remove(ob_boxes, i)
                array.remove(ob_tops, i)
                array.remove(ob_bots, i)
                array.remove(ob_dirs, i)
                array.remove(ob_brk, i)

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
//  SMC: LIQUIDITY SWEEP + INDUCEMENT SWEEP
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

if use_smc and show_induce and array.size(idm_lines) > 0
    for i = array.size(idm_lines) - 1 to 0
        p = array.get(idm_prices, i)
        d = array.get(idm_dirs, i)
        if d == 1 and high > p and close < p
            idm_swept := true
            label.new(bar_index, high, "IDM", style = label.style_label_down,
                 color = color.new(col_liq, 30), textcolor = col_liq, size = size.tiny)
            line.delete(array.get(idm_lines, i))
            array.remove(idm_lines, i)
            array.remove(idm_prices, i)
            array.remove(idm_dirs, i)
        else if d == -1 and low < p and close > p
            idm_swept := true
            label.new(bar_index, low, "IDM", style = label.style_label_up,
                 color = color.new(col_liq, 30), textcolor = col_liq, size = size.tiny)
            line.delete(array.get(idm_lines, i))
            array.remove(idm_lines, i)
            array.remove(idm_prices, i)
            array.remove(idm_dirs, i)

// ═══════════════════════════════════════════════════════════════
//  SMC: SFP (Swing Failure Pattern)
// ═══════════════════════════════════════════════════════════════

if use_smc and show_sfp and barstate.isconfirmed
    if not na(key_sh) and not sh_broken and high > key_sh and close < key_sh
        sfp_bear := true
        label.new(bar_index, high, "SFP", style = label.style_label_down,
             color = color.new(col_res, 40), textcolor = color.white, size = size.small)
    if not na(key_sl) and not sl_broken and low < key_sl and close > key_sl
        sfp_bull := true
        label.new(bar_index, low, "SFP", style = label.style_label_up,
             color = color.new(col_sup, 40), textcolor = color.white, size = size.small)

// ═══════════════════════════════════════════════════════════════
//  SMC: BOS / CHOCH DETECTION
// ═══════════════════════════════════════════════════════════════

// Min swing size filter — reject noise in ranging markets
float swing_range = not na(key_sh) and not na(key_sl) ? key_sh - key_sl : 0.0
bool swing_size_ok = smc_min_swing <= 0 or na(atr) or swing_range >= atr * smc_min_swing

if use_smc and barstate.isconfirmed
    if not na(key_sh) and not sh_broken and close > key_sh and disp_ok and mtf_bull_ok and swing_size_ok
        if key_sh_bar == pend_smc_bull_key
            if bar_index > pend_smc_bull_lb
                pend_smc_bull_cnt += 1
                pend_smc_bull_lb := bar_index
        else
            pend_smc_bull_key := key_sh_bar
            pend_smc_bull_cnt := 1
            pend_smc_bull_lb  := bar_index
        if pend_smc_bull_cnt >= smc_confirm
            sh_broken := true
            smc_bull := true
            if mkt_structure == -1
                smc_is_choch_bull := true
            else
                smc_is_bos_bull := true
            mkt_structure := 1
            pend_smc_bull_key := -1
            pend_smc_bull_cnt := 0
            pend_smc_bull_lb  := -1
            if not na(swing_h_line)
                line.delete(swing_h_line)
                swing_h_line := na
    else if pend_smc_bull_key >= 0 and bar_index > pend_smc_bull_lb
        pend_smc_bull_key := -1
        pend_smc_bull_cnt := 0
        pend_smc_bull_lb  := -1

    if not na(key_sl) and not sl_broken and close < key_sl and disp_ok and mtf_bear_ok and swing_size_ok
        if key_sl_bar == pend_smc_bear_key
            if bar_index > pend_smc_bear_lb
                pend_smc_bear_cnt += 1
                pend_smc_bear_lb := bar_index
        else
            pend_smc_bear_key := key_sl_bar
            pend_smc_bear_cnt := 1
            pend_smc_bear_lb  := bar_index
        if pend_smc_bear_cnt >= smc_confirm
            sl_broken := true
            smc_bear := true
            if mkt_structure == 1
                smc_is_choch_bear := true
            else
                smc_is_bos_bear := true
            mkt_structure := -1
            pend_smc_bear_key := -1
            pend_smc_bear_cnt := 0
            pend_smc_bear_lb  := -1
            if not na(swing_l_line)
                line.delete(swing_l_line)
                swing_l_line := na
    else if pend_smc_bear_key >= 0 and bar_index > pend_smc_bear_lb
        pend_smc_bear_key := -1
        pend_smc_bear_cnt := 0
        pend_smc_bear_lb  := -1

// ═══════════════════════════════════════════════════════════════
//  OTE ZONE (после BOS/CHOCH)
// ═══════════════════════════════════════════════════════════════

if use_smc and show_ote
    if smc_bull and not na(key_sh) and not na(key_sl) and key_sh > key_sl
        float rng = key_sh - key_sl
        ote_top_price := key_sh - rng * 0.618
        ote_bot_price := key_sh - rng * 0.786
        ote_dir := 1
        ote_buy_fired := false
        if not na(ote_box)
            box.delete(ote_box)
        ote_box := box.new(bar_index, ote_top_price, bar_index + 50, ote_bot_price,
             bgcolor = color.new(col_sup, 88), border_color = color.new(col_sup, 50),
             border_width = 1, border_style = line.style_dotted)
    if smc_bear and not na(key_sh) and not na(key_sl) and key_sh > key_sl
        float rng = key_sh - key_sl
        ote_top_price := key_sl + rng * 0.786
        ote_bot_price := key_sl + rng * 0.618
        ote_dir := -1
        ote_sell_fired := false
        if not na(ote_box)
            box.delete(ote_box)
        ote_box := box.new(bar_index, ote_top_price, bar_index + 50, ote_bot_price,
             bgcolor = color.new(col_res, 88), border_color = color.new(col_res, 50),
             border_width = 1, border_style = line.style_dotted)
    if not na(ote_box)
        box.set_right(ote_box, bar_index + 5)

// ═══════════════════════════════════════════════════════════════
//  SMC: ORDER BLOCK CREATION
// ═══════════════════════════════════════════════════════════════

if use_smc and show_ob
    if smc_bull
        for j = 1 to ob_lookback
            if close[j] < open[j]
                if array.size(ob_boxes) >= ob_max
                    box.delete(array.shift(ob_boxes))
                    array.shift(ob_tops)
                    array.shift(ob_bots)
                    array.shift(ob_dirs)
                    array.shift(ob_brk)
                float ot = ob_use_body ? math.max(open[j], close[j]) : high[j]
                float ob2 = ob_use_body ? math.min(open[j], close[j]) : low[j]
                bx = box.new(bar_index - j, ot, bar_index + 30, ob2,
                     bgcolor = col_ob_bull, border_color = col_sup, border_width = 1)
                array.push(ob_boxes, bx)
                array.push(ob_tops, ot)
                array.push(ob_bots, ob2)
                array.push(ob_dirs, 1)
                array.push(ob_brk, 0)
                break
    if smc_bear
        for j = 1 to ob_lookback
            if close[j] > open[j]
                if array.size(ob_boxes) >= ob_max
                    box.delete(array.shift(ob_boxes))
                    array.shift(ob_tops)
                    array.shift(ob_bots)
                    array.shift(ob_dirs)
                    array.shift(ob_brk)
                float ot = ob_use_body ? math.max(open[j], close[j]) : high[j]
                float ob2 = ob_use_body ? math.min(open[j], close[j]) : low[j]
                bx = box.new(bar_index - j, ot, bar_index + 30, ob2,
                     bgcolor = col_ob_bear, border_color = col_res, border_width = 1)
                array.push(ob_boxes, bx)
                array.push(ob_tops, ot)
                array.push(ob_bots, ob2)
                array.push(ob_dirs, -1)
                array.push(ob_brk, 0)
                break

// ═══════════════════════════════════════════════════════════════
//  SMC: FVG DETECTION
// ═══════════════════════════════════════════════════════════════

if use_smc and show_fvg and barstate.isconfirmed and bar_index > 2
    float fvg_min = fvg_min_atr > 0 and not na(atr) ? atr * fvg_min_atr : 0.0
    if low > high[2]
        float gap = low - high[2]
        if gap > fvg_min
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
    if high < low[2]
        float gap = low[2] - high
        if gap > fvg_min
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
//  SMC: LIQUIDITY DETECTION
// ═══════════════════════════════════════════════════════════════

if use_smc and show_liq and not na(atr)
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
//  SMC: SWING LEVEL LINES + EXTENSION
// ═══════════════════════════════════════════════════════════════

if use_smc and show_swing_lvl
    if not na(key_sh) and not sh_broken
        if na(swing_h_line)
            swing_h_line := line.new(key_sh_bar, key_sh, bar_index, key_sh,
                 color = color.new(col_res, 30), style = line.style_dashed, width = 1)
        else
            line.set_x2(swing_h_line, bar_index)
    else if not na(swing_h_line)
        line.delete(swing_h_line)
        swing_h_line := na
    if not na(key_sl) and not sl_broken
        if na(swing_l_line)
            swing_l_line := line.new(key_sl_bar, key_sl, bar_index, key_sl,
                 color = color.new(col_sup, 30), style = line.style_dashed, width = 1)
        else
            line.set_x2(swing_l_line, bar_index)
    else if not na(swing_l_line)
        line.delete(swing_l_line)
        swing_l_line := na

if use_smc and show_ob and array.size(ob_boxes) > 0
    for i = 0 to array.size(ob_boxes) - 1
        box.set_right(array.get(ob_boxes, i), bar_index + 5)

if use_smc and show_fvg and array.size(fvg_boxes) > 0
    for i = 0 to array.size(fvg_boxes) - 1
        box.set_right(array.get(fvg_boxes, i), bar_index + 5)

if use_smc and show_liq and array.size(liq_lines) > 0
    for i = 0 to array.size(liq_lines) - 1
        line.set_x2(array.get(liq_lines, i), bar_index + 5)

if use_smc and show_induce and array.size(idm_lines) > 0
    for i = 0 to array.size(idm_lines) - 1
        line.set_x2(array.get(idm_lines, i), bar_index + 5)

// ═══════════════════════════════════════════════════════════════
//  CONFLUENCE SCORING
// ═══════════════════════════════════════════════════════════════

pd_high = key_sh
pd_low  = key_sl
pd_mid  = not na(pd_high) and not na(pd_low) ? (pd_high + pd_low) / 2.0 : na

in_premium  = not na(pd_mid) and close > pd_mid
in_discount = not na(pd_mid) and close < pd_mid

near_ob_bull = false
near_ob_bear = false
if use_smc and array.size(ob_boxes) > 0
    for i = 0 to array.size(ob_boxes) - 1
        d = array.get(ob_dirs, i)
        t = array.get(ob_tops, i)
        b = array.get(ob_bots, i)
        if d == 1 and low <= t and close >= b
            near_ob_bull := true
        if d == -1 and high >= b and close <= t
            near_ob_bear := true

near_fvg_bull = false
near_fvg_bear = false
if use_smc and array.size(fvg_boxes) > 0
    for i = 0 to array.size(fvg_boxes) - 1
        d = array.get(fvg_dirs, i)
        t = array.get(fvg_tops, i)
        b = array.get(fvg_bots, i)
        if d == 1 and low <= t and close >= b
            near_fvg_bull := true
        if d == -1 and high >= b and close <= t
            near_fvg_bear := true

int conf_bull = 0
int conf_bear = 0
if is_displacement and close > open
    conf_bull += 1
if is_displacement and close < open
    conf_bear += 1
if in_discount
    conf_bull += 1
if in_premium
    conf_bear += 1
if use_mtf and htf_s > 0
    conf_bull += 1
if use_mtf and htf_s < 0
    conf_bear += 1
if vol_ok
    conf_bull += 1
    conf_bear += 1
if near_ob_bull or near_fvg_bull
    conf_bull += 1
if near_ob_bear or near_fvg_bear
    conf_bear += 1

int confluence = mkt_structure == 1 ? conf_bull : mkt_structure == -1 ? conf_bear : math.max(conf_bull, conf_bear)

// ═══════════════════════════════════════════════════════════════
//  COMPOSITE SIGNAL + SL/TP + BACKTEST
// ═══════════════════════════════════════════════════════════════

// Backtest exit check + trailing stop (before new entries)
if bt_in_trade
    if bt_dir == 1
        // Trailing: move SL to breakeven after 1.5R profit
        if not bt_trailed and not na(bt_entry) and not na(bt_sl)
            float be_level = bt_entry + (bt_entry - bt_sl) * 1.5
            if high >= be_level
                bt_sl := bt_entry
                bt_trailed := true
                if show_sltp and not na(sl_line)
                    line.set_y1(sl_line, bt_sl)
                    line.set_y2(sl_line, bt_sl)
        if high >= bt_tp
            bt_wins += 1
            bt_sum_rr += bt_last_rr
            bt_in_trade := false
        else if low <= bt_sl
            if bt_trailed
                bt_wins += 1
                bt_sum_rr += 0.5
            else
                bt_losses += 1
            bt_in_trade := false
    else if bt_dir == -1
        // Trailing: move SL to breakeven after 1.5R profit
        if not bt_trailed and not na(bt_entry) and not na(bt_sl)
            float be_level = bt_entry - (bt_sl - bt_entry) * 1.5
            if low <= be_level
                bt_sl := bt_entry
                bt_trailed := true
                if show_sltp and not na(sl_line)
                    line.set_y1(sl_line, bt_sl)
                    line.set_y2(sl_line, bt_sl)
        if low <= bt_tp
            bt_wins += 1
            bt_sum_rr += bt_last_rr
            bt_in_trade := false
        else if high >= bt_sl
            if bt_trailed
                bt_wins += 1
                bt_sum_rr += 0.5
            else
                bt_losses += 1
            bt_in_trade := false

// Clean SL/TP lines when trade exits
if not bt_in_trade and show_sltp
    if not na(sl_line)
        line.delete(sl_line)
        sl_line := na
    if not na(tp_line)
        line.delete(tp_line)
        tp_line := na

// Composite entry signal
bool in_ote_buy  = ote_dir == 1 and not na(ote_top_price) and not na(ote_bot_price) and close <= ote_top_price and close >= ote_bot_price
bool in_ote_sell = ote_dir == -1 and not na(ote_top_price) and not na(ote_bot_price) and close >= ote_bot_price and close <= ote_top_price

bool cd_ok = bar_index - last_composite_bar >= composite_cooldown

if use_smc and show_composite and barstate.isconfirmed and not bt_in_trade and not in_no_trade and cd_ok
    if mkt_structure == 1 and conf_bull >= composite_min_conf and not ote_buy_fired
        bool ote_cond = show_ote ? in_ote_buy : (near_ob_bull or near_fvg_bull or in_discount)
        if ote_cond
            composite_buy := true
            ote_buy_fired := true
            last_composite_bar := bar_index
            bt_entry := close
            // SL: below OTE zone bottom + ATR buffer; TP: ATR-based
            float sl_atr_buf = not na(atr) ? atr * sl_atr_buf_mul : close * 0.01
            bt_sl := show_ote and not na(ote_bot_price) ? ote_bot_price - sl_atr_buf : (near_ob_bull and array.size(ob_bots) > 0 ? array.get(ob_bots, array.size(ob_bots) - 1) - sl_atr_buf : (not na(key_sl) ? key_sl : close - (not na(atr) ? atr * 1.5 : close * 0.015)))
            bt_tp := close + (not na(atr) ? atr * tp_atr_mul : close * 0.025)
            bt_trailed := false
            float denom = bt_entry - bt_sl
            bt_last_rr := denom > 0 ? (bt_tp - bt_entry) / denom : 1.0
            bt_dir := 1
            bt_in_trade := true
            bt_total += 1
            if show_sltp
                sl_line := line.new(bar_index, bt_sl, bar_index + 1, bt_sl,
                     color = col_res, style = line.style_dotted, width = 1, extend = extend.right)
                tp_line := line.new(bar_index, bt_tp, bar_index + 1, bt_tp,
                     color = col_sup, style = line.style_dotted, width = 1, extend = extend.right)
            label.new(bar_index, low, "BUY\nR:R " + str.tostring(bt_last_rr, "#.#") + ":1\nConf " + str.tostring(conf_bull) + "/5",
                 style = label.style_label_up, color = col_sup, textcolor = color.white, size = size.normal)

    if mkt_structure == -1 and conf_bear >= composite_min_conf and not ote_sell_fired
        bool ote_cond = show_ote ? in_ote_sell : (near_ob_bear or near_fvg_bear or in_premium)
        if ote_cond
            composite_sell := true
            ote_sell_fired := true
            last_composite_bar := bar_index
            bt_entry := close
            // SL: above OTE zone top + ATR buffer; TP: ATR-based
            float sl_atr_buf = not na(atr) ? atr * sl_atr_buf_mul : close * 0.01
            bt_sl := show_ote and not na(ote_top_price) ? ote_top_price + sl_atr_buf : (near_ob_bear and array.size(ob_tops) > 0 ? array.get(ob_tops, array.size(ob_tops) - 1) + sl_atr_buf : (not na(key_sh) ? key_sh : close + (not na(atr) ? atr * 1.5 : close * 0.015)))
            bt_tp := close - (not na(atr) ? atr * tp_atr_mul : close * 0.025)
            bt_trailed := false
            float denom = bt_sl - bt_entry
            bt_last_rr := denom > 0 ? (bt_entry - bt_tp) / denom : 1.0
            bt_dir := -1
            bt_in_trade := true
            bt_total += 1
            if show_sltp
                sl_line := line.new(bar_index, bt_sl, bar_index + 1, bt_sl,
                     color = col_res, style = line.style_dotted, width = 1, extend = extend.right)
                tp_line := line.new(bar_index, bt_tp, bar_index + 1, bt_tp,
                     color = col_sup, style = line.style_dotted, width = 1, extend = extend.right)
            label.new(bar_index, high, "SELL\nR:R " + str.tostring(bt_last_rr, "#.#") + ":1\nConf " + str.tostring(conf_bear) + "/5",
                 style = label.style_label_down, color = col_res, textcolor = color.white, size = size.normal)

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

if use_fast and fast_len < eff_swing_len
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

// BOS/CHOCH labels with confluence
if use_smc and show_bos and smc_is_bos_bull and conf_bull >= smc_min_conf
    label.new(bar_index, low, "BOS " + str.tostring(conf_bull) + "/5",
         style = label.style_label_up, color = col_bos, textcolor = color.white, size = size.small)
if use_smc and show_bos and smc_is_bos_bear and conf_bear >= smc_min_conf
    label.new(bar_index, high, "BOS " + str.tostring(conf_bear) + "/5",
         style = label.style_label_down, color = col_bos, textcolor = color.white, size = size.small)
if use_smc and show_choch and smc_is_choch_bull and conf_bull >= smc_min_conf
    label.new(bar_index, low, "CHOCH " + str.tostring(conf_bull) + "/5",
         style = label.style_label_up, color = col_choch, textcolor = color.white, size = size.normal)
if use_smc and show_choch and smc_is_choch_bear and conf_bear >= smc_min_conf
    label.new(bar_index, high, "CHOCH " + str.tostring(conf_bear) + "/5",
         style = label.style_label_down, color = col_choch, textcolor = color.white, size = size.normal)

plotshape(use_smc and is_displacement and close > open, title = "Disp Bull",
     style = shape.diamond, location = location.belowbar, color = color.new(col_bos, 60), size = size.tiny)
plotshape(use_smc and is_displacement and close < open, title = "Disp Bear",
     style = shape.diamond, location = location.abovebar, color = color.new(col_choch, 60), size = size.tiny)

// ═══════════════════════════════════════════════════════════════
//  PREMIUM / DISCOUNT + KILL ZONES + CANDLE COLORING
// ═══════════════════════════════════════════════════════════════

bgcolor(use_smc and show_pd and in_premium ? col_premium : na, title = "Premium Zone")
bgcolor(use_smc and show_pd and in_discount ? col_discount : na, title = "Discount Zone")
plot(use_smc and show_pd and not na(pd_mid) ? pd_mid : na, "Equilibrium",
     color = color.new(color.gray, 50), style = plot.style_cross)

bgcolor(show_kz and in_any_kz ? col_kz : na, title = "Kill Zone BG")

barcolor(use_candle_color ?
     (mkt_structure == 1 and in_discount ? color.new(col_sup, 20) :
      mkt_structure == -1 and in_premium ? color.new(col_res, 20) :
      color.new(color.gray, 50)) : na, title = "Structure Candle Color")

// ═══════════════════════════════════════════════════════════════
//  ИНФОРМАЦИОННАЯ ПАНЕЛЬ (РУССКИЙ)
// ═══════════════════════════════════════════════════════════════

var table tbl = table.new(position.top_right, 2, 18,
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

    table.cell(tbl, 0, 0, "TBDv6+SMC", text_color = color.white, text_size = size.small)
    table.cell(tbl, 1, 0, barstate.isrealtime ? "LIVE" : "ИСТОРИЯ", text_color = barstate.isrealtime ? #00BFA5 : color.gray, text_size = size.small)

    table.cell(tbl, 0, 1, "Линии", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 1, str.tostring(sc) + " П / " + str.tostring(rc) + " С", text_color = color.white, text_size = size.small)

    string tfm = trend_mode == "EMA Cross" ? "Cross" : trend_mode == "Price vs EMA" ? "Price" : "ВЫКЛ"
    bool bt2 = trend_bull_ok
    table.cell(tbl, 0, 2, "Тренд", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 2, trend_mode == "Off" ? "ВЫКЛ" : (bt2 ? tfm + ":БЫК" : tfm + ":МЕДВ"),
         text_color = trend_mode == "Off" ? color.gray : (bt2 ? col_sup : col_res), text_size = size.small)

    table.cell(tbl, 0, 3, "Поддержка", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 3, not na(bs) ? str.tostring(bs, format.mintick) : "---", text_color = not na(bs) ? col_sup : color.gray, text_size = size.small)

    table.cell(tbl, 0, 4, "Сопротив.", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 4, not na(br) ? str.tostring(br, format.mintick) : "---", text_color = not na(br) ? col_res : color.gray, text_size = size.small)

    table.cell(tbl, 0, 5, "Моментум", text_color = color.gray, text_size = size.small)
    string mm = mom_mode == "Off" ? "ВЫКЛ" : mom_mode + (mom_bull_ok ? " Б" : " М")
    table.cell(tbl, 1, 5, mm, text_color = mom_mode == "Off" ? color.gray : (mom_bull_ok ? col_sup : col_res), text_size = size.small)

    table.cell(tbl, 0, 6, "Подтвержд.", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 6, str.tostring(confirm_bars) + "б" + (inst_conf_atr > 0 ? " ИП" : ""), text_color = color.gray, text_size = size.small)

    table.cell(tbl, 0, 7, "Объём", text_color = color.gray, text_size = size.small)
    string vp = vp_active ? str.tostring(math.round(vol_pct * 100)) + "%" : (use_vol_proj ? "ВКЛ" : "ВЫКЛ")
    table.cell(tbl, 1, 7, vp, text_color = vp_active ? color.yellow : color.gray, text_size = size.small)

    table.cell(tbl, 0, 8, "Структура", text_color = color.gray, text_size = size.small)
    string struct_txt = mkt_structure == 1 ? "БЫЧЬЯ" : mkt_structure == -1 ? "МЕДВЕЖЬЯ" : "---"
    color  struct_col = mkt_structure == 1 ? col_sup : mkt_structure == -1 ? col_res : color.gray
    table.cell(tbl, 1, 8, use_smc ? struct_txt : "ВЫКЛ", text_color = use_smc ? struct_col : color.gray, text_size = size.small)

    table.cell(tbl, 0, 9, "Ключ. SH", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 9, use_smc and not na(key_sh) ? str.tostring(key_sh, format.mintick) + (sh_broken ? " X" : "") : "---",
         text_color = sh_broken ? color.gray : col_res, text_size = size.small)

    table.cell(tbl, 0, 10, "Ключ. SL", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 10, use_smc and not na(key_sl) ? str.tostring(key_sl, format.mintick) + (sl_broken ? " X" : "") : "---",
         text_color = sl_broken ? color.gray : col_sup, text_size = size.small)

    table.cell(tbl, 0, 11, "Зона", text_color = color.gray, text_size = size.small)
    string zone_txt = not use_smc or na(pd_mid) ? "---" : in_premium ? "ПРЕМИУМ" : in_discount ? "ДИСКОНТ" : "EQ"
    color  zone_col = not use_smc or na(pd_mid) ? color.gray : in_premium ? col_res : in_discount ? col_sup : color.gray
    table.cell(tbl, 1, 11, zone_txt, text_color = zone_col, text_size = size.small)

    table.cell(tbl, 0, 12, "OB / FVG", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 12, use_smc ? str.tostring(array.size(ob_boxes)) + " / " + str.tostring(array.size(fvg_boxes)) : "ВЫКЛ",
         text_color = color.white, text_size = size.small)

    table.cell(tbl, 0, 13, "Confluence", text_color = color.gray, text_size = size.small)
    color  conf_col = confluence >= 4 ? col_sup : confluence >= 2 ? color.yellow : color.gray
    table.cell(tbl, 1, 13, use_smc ? str.tostring(confluence) + "/5" : "ВЫКЛ",
         text_color = use_smc ? conf_col : color.gray, text_size = size.small)

    table.cell(tbl, 0, 14, "Kill Zone", text_color = color.gray, text_size = size.small)
    string kz_txt = not show_kz ? "ВЫКЛ" : in_kz_london ? "ЛОНДОН" : in_kz_ny_open ? "НЬЮ-ЙОРК" : in_kz_ny_close ? "SILVER BULLET" : "---"
    table.cell(tbl, 1, 14, kz_txt, text_color = in_any_kz and show_kz ? color.orange : color.gray, text_size = size.small)

    table.cell(tbl, 0, 15, "Сигнал", text_color = color.gray, text_size = size.small)
    string sig_txt = bt_in_trade ? (bt_dir == 1 ? "BUY" : "SELL") : "---"
    color  sig_col = bt_in_trade ? (bt_dir == 1 ? col_sup : col_res) : color.gray
    table.cell(tbl, 1, 15, show_composite ? sig_txt : "ВЫКЛ", text_color = sig_col, text_size = size.small)

    table.cell(tbl, 0, 16, "R:R", text_color = color.gray, text_size = size.small)
    table.cell(tbl, 1, 16, bt_in_trade ? str.tostring(bt_last_rr, "#.#") + ":1" : "---",
         text_color = bt_in_trade ? color.white : color.gray, text_size = size.small)

    table.cell(tbl, 0, 17, "Win Rate", text_color = color.gray, text_size = size.small)
    float wr = bt_total > 0 ? bt_wins * 100.0 / bt_total : 0
    float avg_rr = bt_wins > 0 ? bt_sum_rr / bt_wins : 0
    string wr_txt = show_stats and bt_total > 0 ? str.tostring(math.round(wr)) + "% (" + str.tostring(bt_wins) + "/" + str.tostring(bt_total) + ") R:" + str.tostring(avg_rr, "#.#") : "---"
    table.cell(tbl, 1, 17, wr_txt, text_color = wr >= 50 ? col_sup : bt_total > 0 ? col_res : color.gray, text_size = size.small)

// ═══════════════════════════════════════════════════════════════
//  MULTI-SYMBOL PANEL
// ═══════════════════════════════════════════════════════════════

var table tbl2 = table.new(position.bottom_right, 3, 3,
     bgcolor = color.new(#1E1E1E, 10), border_color = color.new(color.gray, 60), border_width = 1)

if show_multi and barstate.islast
    table.cell(tbl2, 0, 0, "Символ", text_color = color.gray, text_size = size.small)
    table.cell(tbl2, 1, 0, "Тренд", text_color = color.gray, text_size = size.small)
    table.cell(tbl2, 2, 0, "EMA", text_color = color.gray, text_size = size.small)

    // Symbol 1
    string s1_name = str.contains(sym1_input, ":") ? str.substring(sym1_input, str.pos(sym1_input, ":") + 1) : sym1_input
    table.cell(tbl2, 0, 1, s1_name, text_color = color.white, text_size = size.small)
    table.cell(tbl2, 1, 1, sym1_bull ? "БЫК" : "МЕДВ", text_color = sym1_bull ? col_sup : col_res, text_size = size.small)
    table.cell(tbl2, 2, 1, not na(sym1_ef) ? str.tostring(sym1_ef, format.mintick) : "---", text_color = color.gray, text_size = size.small)

    // Symbol 2
    string s2_name = str.contains(sym2_input, ":") ? str.substring(sym2_input, str.pos(sym2_input, ":") + 1) : sym2_input
    table.cell(tbl2, 0, 2, s2_name, text_color = color.white, text_size = size.small)
    table.cell(tbl2, 1, 2, sym2_bull ? "БЫК" : "МЕДВ", text_color = sym2_bull ? col_sup : col_res, text_size = size.small)
    table.cell(tbl2, 2, 2, not na(sym2_ef) ? str.tostring(sym2_ef, format.mintick) : "---", text_color = color.gray, text_size = size.small)

// ═══════════════════════════════════════════════════════════════
//  АЛЕРТЫ
// ═══════════════════════════════════════════════════════════════

alertcondition(bull_break and alr_bull, "Bullish Break",
     "Bullish break on {{ticker}} {{interval}} at {{close}}")
alertcondition(bear_break and alr_bear, "Bearish Break",
     "Bearish break on {{ticker}} {{interval}} at {{close}}")
alertcondition((bull_break and alr_bull) or (bear_break and alr_bear), "Any Break",
     "Break on {{ticker}} {{interval}} at {{close}}")
alertcondition((fast_bull or fast_bear) and alr_fast and not bull_break and not bear_break, "Fast Break (early)",
     "Fast break on {{ticker}} {{interval}} at {{close}}")
alertcondition(((bull_tent and not bull_break) or (bear_tent and not bear_break)) and alr_tent, "Tentative Break",
     "Tentative on {{ticker}} {{interval}} at {{close}}")
alertcondition((smc_is_bos_bull or smc_is_bos_bear) and alr_bos, "BOS",
     "BOS on {{ticker}} {{interval}} at {{close}}")
alertcondition((smc_is_choch_bull or smc_is_choch_bear) and alr_choch, "CHOCH",
     "CHOCH on {{ticker}} {{interval}} at {{close}}")
alertcondition((sweep_bull or sweep_bear) and alr_sweep, "Liquidity Sweep",
     "Liquidity sweep on {{ticker}} {{interval}} at {{close}}")
alertcondition(idm_swept and alr_induce, "Inducement (IDM)",
     "Inducement swept on {{ticker}} {{interval}} at {{close}}")
alertcondition((composite_buy or composite_sell) and alr_composite, "Composite Signal",
     "Composite signal on {{ticker}} {{interval}} at {{close}}")
alertcondition((sfp_bull or sfp_bear) and alr_sfp, "SFP",
     "SFP on {{ticker}} {{interval}} at {{close}}")

// Enriched alerts
string zone_str = in_premium ? "ПРЕМИУМ" : in_discount ? "ДИСКОНТ" : "EQ"
string struct_str = mkt_structure == 1 ? "БЫЧЬЯ" : mkt_structure == -1 ? "МЕДВЕЖЬЯ" : "---"

if smc_is_bos_bull and alr_bos
    alert("BOS UP | " + syminfo.ticker + " " + timeframe.period + " | " + str.tostring(close, format.mintick) + " | " + struct_str + " | " + zone_str + " | Conf:" + str.tostring(conf_bull) + "/5", alert.freq_once_per_bar)
if smc_is_bos_bear and alr_bos
    alert("BOS DN | " + syminfo.ticker + " " + timeframe.period + " | " + str.tostring(close, format.mintick) + " | " + struct_str + " | " + zone_str + " | Conf:" + str.tostring(conf_bear) + "/5", alert.freq_once_per_bar)
if smc_is_choch_bull and alr_choch
    alert("CHOCH UP | " + syminfo.ticker + " " + timeframe.period + " | " + str.tostring(close, format.mintick) + " | БЫЧЬЯ | " + zone_str + " | Conf:" + str.tostring(conf_bull) + "/5", alert.freq_once_per_bar)
if smc_is_choch_bear and alr_choch
    alert("CHOCH DN | " + syminfo.ticker + " " + timeframe.period + " | " + str.tostring(close, format.mintick) + " | МЕДВЕЖЬЯ | " + zone_str + " | Conf:" + str.tostring(conf_bear) + "/5", alert.freq_once_per_bar)
if composite_buy and alr_composite
    alert("BUY | " + syminfo.ticker + " " + timeframe.period + " | " + str.tostring(close, format.mintick) + " | SL:" + str.tostring(bt_sl, format.mintick) + " | TP:" + str.tostring(bt_tp, format.mintick) + " | R:R " + str.tostring(bt_last_rr, "#.#") + ":1 | Conf:" + str.tostring(conf_bull) + "/5", alert.freq_once_per_bar)
if composite_sell and alr_composite
    alert("SELL | " + syminfo.ticker + " " + timeframe.period + " | " + str.tostring(close, format.mintick) + " | SL:" + str.tostring(bt_sl, format.mintick) + " | TP:" + str.tostring(bt_tp, format.mintick) + " | R:R " + str.tostring(bt_last_rr, "#.#") + ":1 | Conf:" + str.tostring(conf_bear) + "/5", alert.freq_once_per_bar)
if (sfp_bull or sfp_bear) and alr_sfp
    alert("SFP | " + syminfo.ticker + " " + timeframe.period + " | " + str.tostring(close, format.mintick) + " | " + struct_str + " | " + zone_str, alert.freq_once_per_bar)
if (sweep_bull or sweep_bear) and alr_sweep
    alert("SWEEP | " + syminfo.ticker + " " + timeframe.period + " | " + str.tostring(close, format.mintick) + " | " + struct_str, alert.freq_once_per_bar)
if idm_swept and alr_induce
    alert("IDM | " + syminfo.ticker + " " + timeframe.period + " | " + str.tostring(close, format.mintick) + " | " + struct_str + " | " + zone_str, alert.freq_once_per_bar)
