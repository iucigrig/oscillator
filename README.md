//@version=5

//Arranged by @ClassicScott

//A version of Stephen J. Klinger's, Klinger Oscillator (sometimes called Klinger Volume Oscillator). I've changed virtually nothing about the indicator itself,
//but added some lookback inputs for the EMAs the oscillator is derived from (traditionally 34 and 55), and added a few other things, as is my wont.

//But what is the Klinger Oscillator? Essentially, the calculation looks at the high, low, and close of the current period, and compares that to the
//previous period's. If it is greater, it adds volume, and if it is less, it subtracts volume. It then takes an EMA of two different lookback periods of that
//calculation and subtracts one from the other. That's your oscillator. There is then made a signal line of the oscillator that a trader can use, in combination
//with the zero line, for taking trades. Investopedia has a good article on it, so if you're looking for more specifics, check there.

//What I've done is add a selection of different moving averages that you may choose for the signal line. Usually it's a 13 period EMA, and that comes default, but
//here you could use an ALMA or HMA, or modular filter, etc. Find something that works for your style/algorithm.

//Of course there are all the usual additions of mine with the various ways of coloring the indicator and candles, adjustable Donchian Bands, and alerts.
//A new addition that I've just added to all my indicators (oscillators, anyway) are divergences. This is more or less just a copy and paste of the divergence
//indicator available in TradingView. In this case you can set it to plot divergences off either the Klinger or the signal line. Depending on which one you choose
//you may have to adjust pivot lookbacks, and lookback range. I've kept the settings default from the RSI TradingView version.


indicator(title='+ Klinger Oscillator', shorttitle='+ KVO', format=format.volume, timeframe='', timeframe_gaps=true)

src = input(defval=hlc3, title='Source for KVO')

var cumVol = 0.
cumVol += nz(volume)
sv = ta.change(src) >= 0 ? volume : -volume

lkbk_1 = input.int(34, title='Lookback for EMA 1', minval=1, group='KVO Inputs', tooltip='The Klinger Volume Oscillator is the difference between two exponential moving averages of different lengths. Standards are 34 and 55.')
lkbk_2 = input.int(55, title='Lookback for EMA 2', minval=1, group='KVO Inputs')


kvo = ta.ema(sv, lkbk_1) - ta.ema(sv, lkbk_2)


hline(0, title='Zero Line', linestyle=hline.style_dotted, color=color.new(#787b86, 0))


////////////////////////////////////////////////////////////////////////////////


i_sig_ma = input.string(defval='EMA', options=['ALMA', 'EMA', 'DEMA', 'TEMA', 'FRAMA', 'HMA', 'JMA', 'LSMA', 'MF', 'RDMA', 'RMA', 'SMA', 'TMA', 'VAMA', 'VWMA', 'WMA'], title='Signal Type', group='Signal Inputs')
i_sig_lkbk = input.int(13, title='Signal Lookback', minval=1, group='Signal Inputs', tooltip='Typically the signal line is a 13 period EMA.')


////SIGNAL MOVING AVERAGE INPUTS AND CALCULATIONS

//ALMA
sig_alma_offset = input.float(title='* ALMA Offset', step=0.05, defval=0.85, inline='alma', group='Signal Inputs')
sig_alma_sigma = input.float(title='* ALMA Sigma', step=0.5, defval=6, inline='alma', group='Signal Inputs')

//FRAMA
sig_fc = input.int(defval=1, minval=1, title='* FRAMA Fast Period', inline='frama', group='Signal Inputs')
sig_sc = input.int(defval=200, minval=1, title='* FRAMA Slow Period', inline='frama', group='Signal Inputs')

//JMA
sig_jurik_phase = input.int(title='* Jurik (JMA) Phase', defval=1, inline='jma', group='Signal Inputs')
sig_jurik_power = input.float(title='* Jurik (JMA) Power', defval=1, minval=0.1, maxval=10, step=0.1, inline='jma', group='Signal Inputs')

//LSMA
sig_offset = input.int(title='* Least Squares (LSMA) - Offset', defval=0, group='Signal Inputs')

//MF
i_sig_beta = input.float(0.8, minval=0, maxval=1, step=0.1, title='* Modular Filter - Beta', inline='mf', group='Signal Inputs')
sig_feedback = input.bool(false, title='* Modular Filter - Feedback', inline='mf', group='Signal Inputs')
sig_z = input.float(0.5, title='* Modular Filter - Feedback Weighting', step=0.1, minval=0, maxval=1, inline='mf', group='Signal Inputs')

//TEMA
sig_tema(kvo, i_sig_lkbk) =>
    sig_ema1 = ta.ema(kvo, i_sig_lkbk)
    sig_ema2 = ta.ema(sig_ema1, i_sig_lkbk)
    sig_ema3 = ta.ema(sig_ema2, i_sig_lkbk)
    3 * sig_ema1 - 3 * sig_ema2 + sig_ema3

//VAMA
sig_volatility_lookback = input.int(10, title='* Volatility Adjusted (VAMA) - Lookback Length', minval=1, group='Signal Inputs')


////LIST OF MOVING AVERAGES
sig_ma(type, kvo, i_sig_lkbk) =>
    float result = 0
    if type == 'ALMA'
        result := ta.alma(kvo, i_sig_lkbk, sig_alma_offset, sig_alma_sigma)
    if type == 'EMA'
        result := ta.ema(kvo, i_sig_lkbk)
    if type == 'DEMA'
        e = ta.ema(kvo, i_sig_lkbk)
        result := 2 * e - ta.ema(e, i_sig_lkbk)
    if type == 'TEMA'
        e = ta.ema(kvo, i_sig_lkbk)
        result := 3 * (e - ta.ema(e, i_sig_lkbk)) + ta.ema(ta.ema(e, i_sig_lkbk), i_sig_lkbk)
    if type == 'FRAMA'
        i_sig_lkbk_ = i_sig_lkbk / 2
        sig_e = 2.7182818284590452353602874713527
        sig_w = math.log(2 / (sig_sc + 1)) / math.log(sig_e)  // Natural logarithm (ln(2/(SC+1))) workaround
        sig_h1 = ta.highest(high, i_sig_lkbk_)
        sig_l1 = ta.lowest(low, i_sig_lkbk_)
        sig_n1 = (sig_h1 - sig_l1) / i_sig_lkbk_
        sig_h2 = ta.highest(high, i_sig_lkbk_)[i_sig_lkbk_]
        sig_l2 = ta.lowest(low, i_sig_lkbk_)[i_sig_lkbk_]
        sig_n2 = (sig_h2 - sig_l2) / i_sig_lkbk_
        sig_h3 = ta.highest(high, i_sig_lkbk)
        sig_l3 = ta.lowest(low, i_sig_lkbk)
        sig_n3 = (sig_h3 - sig_l3) / i_sig_lkbk
        sig_dimen1 = (math.log(sig_n1 + sig_n2) - math.log(sig_n3)) / math.log(2)
        sig_dimen = sig_n1 > 0 and sig_n2 > 0 and sig_n3 > 0 ? sig_dimen1 : nz(sig_dimen1[1])
        sig_alpha1 = math.exp(sig_w * (sig_dimen - 1))
        sig_oldalpha = sig_alpha1 > 1 ? 1 : sig_alpha1 < 0.01 ? 0.01 : sig_alpha1
        sig_oldn = (2 - sig_oldalpha) / sig_oldalpha
        sig_n = (sig_sc - sig_fc) * (sig_oldn - 1) / (sig_sc - 1) + sig_fc
        sig_alpha_ = 2 / (sig_n + 1)
        sig_alpha = sig_alpha_ < 2 / (sig_sc + 1) ? 2 / (sig_sc + 1) : sig_alpha_ > 1 ? 1 : sig_alpha_
        sig_frama = 0.0
        sig_frama := (1 - sig_alpha) * nz(sig_frama[1]) + sig_alpha * kvo
        result := sig_frama
    if type == 'HMA'
        result := ta.wma(2 * ta.wma(kvo, i_sig_lkbk / 2) - ta.wma(kvo, i_sig_lkbk), math.round(math.sqrt(i_sig_lkbk)))
    if type == 'JMA'
        /// Copyright © 2018 Alex Orekhov (everget)
        /// Copyright © 2017 Jurik Research and Consulting.
        sig_phaseRatio = sig_jurik_phase < -100 ? 0.5 : sig_jurik_phase > 100 ? 2.5 : sig_jurik_phase / 100 + 1.5
        sig_beta = 0.45 * (i_sig_lkbk - 1) / (0.45 * (i_sig_lkbk - 1) + 2)
        sig_alpha = math.pow(sig_beta, sig_jurik_power)
        sig_jma = 0.0
        sig_e0 = 0.0
        sig_e0 := (1 - sig_alpha) * kvo + sig_alpha * nz(sig_e0[1])
        sig_e1 = 0.0
        sig_e1 := (kvo - sig_e0) * (1 - sig_beta) + sig_beta * nz(sig_e1[1])
        sig_e2 = 0.0
        sig_e2 := (sig_e0 + sig_phaseRatio * sig_e1 - nz(sig_jma[1])) * math.pow(1 - sig_alpha, 2) + math.pow(sig_alpha, 2) * nz(sig_e2[1])
        sig_jma := sig_e2 + nz(sig_jma[1])
        result := sig_jma
    if type == 'LSMA'
        result := ta.linreg(kvo, i_sig_lkbk, sig_offset)
    if type == 'MF'
        sig_ts = 0.
        sig_b = 0.
        sig_c = 0.
        sig_os = 0.
        //----
        sig_alpha = 2 / (i_sig_lkbk + 1)
        sig_a = sig_feedback ? sig_z * kvo + (1 - sig_z) * nz(sig_ts[1], kvo) : kvo
        //----
        sig_b := sig_a > sig_alpha * sig_a + (1 - sig_alpha) * nz(sig_b[1], sig_a) ? sig_a : sig_alpha * sig_a + (1 - sig_alpha) * nz(sig_b[1], sig_a)
        sig_c := sig_a < sig_alpha * sig_a + (1 - sig_alpha) * nz(sig_c[1], sig_a) ? sig_a : sig_alpha * sig_a + (1 - sig_alpha) * nz(sig_c[1], sig_a)
        sig_os := sig_a == sig_b ? 1 : sig_a == sig_c ? 0 : sig_os[1]
        //----
        sig_upper = i_sig_beta * sig_b + (1 - i_sig_beta) * sig_c
        sig_lower = i_sig_beta * sig_c + (1 - i_sig_beta) * sig_b
        sig_ts := sig_os * sig_upper + (1 - sig_os) * sig_lower
        result := sig_ts
    if type == 'RDMA'
        sig_sma_200 = ta.sma(kvo, 200)
        sig_sma_100 = ta.sma(kvo, 100)
        sig_sma_50 = ta.sma(kvo, 50)
        sig_sma_24 = ta.sma(kvo, 24)
        sig_sma_9 = ta.sma(kvo, 9)
        sig_sma_5 = ta.sma(kvo, 5)
        result := (sig_sma_200 + sig_sma_100 + sig_sma_50 + sig_sma_24 + sig_sma_9 + sig_sma_5) / 6
    if type == 'RMA'
        result := ta.rma(kvo, i_sig_lkbk)
    if type == 'SMA'
        result := ta.sma(kvo, i_sig_lkbk)
    if type == 'TMA'
        result := ta.sma(ta.sma(kvo, math.ceil(i_sig_lkbk / 2)), math.floor(i_sig_lkbk / 2) + 1)
    if type == 'VAMA'
        /// Copyright © 2019 to present, Joris Duyck (JD)
        sig_mid = ta.ema(kvo, i_sig_lkbk)
        sig_dev = kvo - sig_mid
        sig_vol_up = ta.highest(sig_dev, sig_volatility_lookback)
        sig_vol_down = ta.lowest(sig_dev, sig_volatility_lookback)
        result := sig_mid + math.avg(sig_vol_up, sig_vol_down)
    if type == 'VWMA'
        result := ta.vwma(kvo, i_sig_lkbk)
    if type == 'WMA'
        result := ta.wma(kvo, i_sig_lkbk)
    result

sig = sig_ma(i_sig_ma, kvo, i_sig_lkbk)


////////////////////////////////////////////////////////////////////////////////


////SIGNAL LINE CROSSING SHAPE PLOTS
c_cross_up = input.color(color.new(#ff9800, 10), title='Bullish Cross', inline='x_pos', group='Signal Line Crosses')
c_cross_down = input.color(color.new(#2962ff, 10), title='Bearish Cross', inline='x_neg', group='Signal Line Crosses')
ma_cross_up = ta.crossover(kvo, sig)
ma_cross_down = ta.crossunder(kvo, sig)
plotshape(ma_cross_up, style=shape.circle, location=location.bottom, color=c_cross_up, title='Signal Line Cross Up')
plotshape(ma_cross_down, style=shape.circle, location=location.top, color=c_cross_down, title='Signal Line Cross Down')


////////////////////////////////////////////////////////////////////////////////


////DONCHIAN CHANNEL
dc = input.bool(defval=true, title='Donchian Channel?', group='Donchian Channel')
dc_lkbk = input.int(defval=55, title='Lookback', group='Donchian Channel')
band_width = input.float(defval=8, step=0.5, title='Band Thickness', group='Donchian Channel', tooltip='Defines the thickness of the band. Larger number is thinner. Realistically you wouldn\'t want to go below 4, I would think, or above 10.')

upper = ta.highest(kvo, dc_lkbk)
lower = ta.lowest(kvo, dc_lkbk)

band = (upper - lower) / band_width

inner_upper = upper - band
inner_lower = lower + band

upper_plot = plot(dc ? upper : na, color=color.new(#ef5350, 0), title='Upper DC', display=display.none)
lower_plot = plot(dc ? lower : na, color=color.new(#2196f3, 0), title='Lower DC', display=display.none)
i_upper_plot = plot(dc ? inner_upper : na, color=color.new(#ef5350, 0), title='Inner Upper DC', display=display.none)
i_lower_plot = plot(dc ? inner_lower : na, color=color.new(#2196f3, 0), title='Inner Lower DC', display=display.none)

dc_pos_fill = input.color(color.new(#787b86, 85), title='Lower Band Fill', inline='dcc', group='Donchian Channel')
dc_neg_fill = input.color(color.new(#787b86, 85), title='Upper Band Fill', inline='dcc', group='Donchian Channel')

fill(upper_plot, i_upper_plot, color=dc_neg_fill, title='Upper DC Band')
fill(lower_plot, i_lower_plot, color=dc_pos_fill, title='Lower DC Band')


////////////////////////////////////////////////////////////////////////////////


////KLINGER OSCILLATOR COLORS AND PLOTS
i_kvo_color_selection = input.string(defval='Gradient', options=['Gradient', 'Center Line', 'Signal Line', 'OB/OS'], title='Reference for KVO Colors', group='KVO Inputs', tooltip='Defines the color reference for the KVO. Gradient is a gradient between the upper and lower Donchian bands. OB/OS is simply \'overbought\' or \'oversold,\' using the Donchian channels. And Signal is the KVO in reference to the signal line.')

bull_kvo_color = input.color(color.new(#ff9800, 0), title='Bullish', inline='kvo_c', group='KVO Inputs')
neut_kvo_color = input.color(color.new(#787b86, 0), title='Neutral', inline='kvo_c', group='KVO Inputs')
bear_kvo_color = input.color(color.new(#2962ff, 0), title='Bearish', inline='kvo_c', group='KVO Inputs')

kvo_1 = color.from_gradient(kvo, lower, upper, bear_kvo_color, bull_kvo_color)

kvo_2 = kvo > 0 ? bull_kvo_color : bear_kvo_color

kvo_3 = kvo > sig and kvo > 0 ? bull_kvo_color : kvo < sig and kvo < 0 ? bear_kvo_color : neut_kvo_color

kvo_4 = kvo >= inner_upper ? bull_kvo_color : kvo <= inner_lower ? bear_kvo_color : neut_kvo_color

kvo_color_selection = i_kvo_color_selection == 'Gradient' ? kvo_1 : i_kvo_color_selection == 'Center Line' ? kvo_2 : i_kvo_color_selection == 'Signal Line' ? kvo_3 : i_kvo_color_selection == 'OB/OS' ? kvo_4 : na

p_kvo = plot(kvo, color=kvo_color_selection, title='KVO')


////////////////////////////////////////////////////////////////////////////////


////SIGNAL LINE COLORS AND PLOTS
i_sig_color_selection = input.string(defval='Gradient', options=['Gradient', 'Center Line'], title='Reference for Signal Colors', group='Signal Inputs', tooltip='Defines the color reference for the signal line. See tooltip for KVO.')

bull_sig_color = input.color(color.new(#00bcd4, 79), title='Bullish', inline='sig_c', group='Signal Inputs')
neut_sig_color = input.color(color.new(#787b86, 79), title='Neutral', inline='sig_c', group='Signal Inputs')
bear_sig_color = input.color(color.new(#e91e63, 79), title='Bearish', inline='sig_c', group='Signal Inputs')

sig_1 = color.from_gradient(sig, inner_lower, inner_upper, bear_sig_color, bull_sig_color)

sig_2 = sig > sig[1] and sig > 0 ? bull_sig_color : sig < sig[1] and sig < 0 ? bear_sig_color : neut_sig_color

sig_color_selection = i_sig_color_selection == 'Gradient' ? sig_1 : i_sig_color_selection == 'Center Line' ? sig_2 : na

p_sig = plot(sig, color=sig_color_selection, style=plot.style_area, title='Signal Line')


////////////////////////////////////////////////////////////////////////////////


////CANDLE COLOR
candle_color = input.bool(defval=true, title='Colored Candles?', group='Candle Colors')
i_cc_selection = input.string(defval='KVO | Gradient', options=['KVO | Gradient', 'KVO | Center Line', 'KVO | Signal Line', 'KVO | OB/OS', 'Signal Line | Gradient', 'Signal Line | Zero Line'], title='Reference for Candle Colors', group='Candle Colors', tooltip='Defines the color reference for the candles. See tooltip for KVO.')

cc_kvo_up = input.color(color.new(#ff9800, 0), title='KVO Bullish', inline='cc_kvo', group='Candle Colors')
cc_sig_up = input.color(color.new(#00bcd4, 0), title='Signal Bullish', inline='cc_sig', group='Candle Colors')
cc_neut = input.color(color.new(#787b86, 0), title='Neutral', inline='cc', group='Candle Colors')
cc_sig_down = input.color(color.new(#e91e63, 0), title='Signal Bearish', inline='cc_sig', group='Candle Colors')
cc_kvo_down = input.color(color.new(#2962ff, 0), title='KVO Bearish', inline='cc_kvo', group='Candle Colors')

cc_1 = color.from_gradient(kvo, lower, upper, cc_kvo_down, cc_kvo_up)

cc_2 = kvo > 0 ? cc_kvo_up : cc_kvo_down

cc_3 = kvo > sig and kvo > 0 ? cc_kvo_up : kvo < sig and kvo < 0 ? cc_kvo_down : cc_neut

cc_4 = kvo >= inner_upper ? cc_kvo_up : kvo <= inner_lower ? cc_kvo_down : cc_neut

cc_5 = color.from_gradient(sig, inner_lower, inner_upper, cc_sig_down, cc_sig_up)

cc_6 = sig > sig[1] and sig > 0 ? cc_sig_up : sig < sig[1] and sig < 0 ? cc_sig_down : cc_neut

cc_selection = i_cc_selection == 'KVO | Gradient' ? cc_1 : i_cc_selection == 'KVO | Center Line' ? cc_2 : i_cc_selection == 'KVO | Signal Line' ? cc_3 : i_cc_selection == 'KVO | OB/OS' ? cc_4 : i_cc_selection == 'Signal Line | Gradient' ? cc_5 : i_cc_selection == 'Signal Line | Zero Line' ? cc_6 : na

barcolor(candle_color ? cc_selection : na, title='Colored Candles')


////////////////////////////////////////////////////////////////////////////////


////DIVERGENCES
i_div_select = input.string(defval='KVO', options=['KVO', 'Signal'], title='Source Select for Divergences', group='Divergences')
div_select = i_div_select == 'KVO' ? kvo : i_div_select == 'Signal' ? sig : na

lkbk_right = input.int(title='Pivot Lookback Right', defval=5, group='Divergences')
lkbk_left = input.int(title='Pivot Lookback Left', defval=5, group='Divergences')
max_range = input.int(title='Max of Lookback Range', defval=60, group='Divergences')
min_range = input.int(title='Min of Lookback Range', defval=5, group='Divergences')
plot_bull = input.bool(title='Bull?', defval=true, group='Divergences')
plot_hdn_bull = input.bool(title='Hidden Bull?', defval=false, group='Divergences')
plot_bear = input.bool(title='Bear?', defval=true, group='Divergences')
plot_hdn_bear = input.bool(title='Hidden Bear?', defval=false, group='Divergences')

////COLOR INPUTS
c_bull = input.color(color.new(color.blue, 0), title='Bull', inline='bull div', group='Divergences')
c_hdn_bull = input.color(color.new(color.yellow, 0), title='Hidden Bull', inline='bull div', group='Divergences')
c_bear = input.color(color.new(color.red, 0), title='Bear', inline='bear div', group='Divergences')
c_hdn_bear = input.color(color.new(color.fuchsia, 0), title='Hidden Bear', inline='bear div', group='Divergences')
c_text = input.color(color.new(color.white, 0), title='Text  ', inline='text', group='Divergences')
c_na = color.new(color.white, 100)



pl_found = na(ta.pivotlow(div_select, lkbk_left, lkbk_right)) ? false : true

ph_found = na(ta.pivothigh(div_select, lkbk_left, lkbk_right)) ? false : true

_inRange(cond) =>
    bars = ta.barssince(cond == true)
    min_range <= bars and bars <= max_range


////
// Bullish
// Osc: Higher Low

osc_hl = div_select[lkbk_right] > ta.valuewhen(pl_found, div_select[lkbk_right], 1) and _inRange(pl_found[1])

// Price: Lower Low

price_ll = low[lkbk_right] < ta.valuewhen(pl_found, low[lkbk_right], 1)
bull_cond = plot_bull and price_ll and osc_hl and pl_found

plot(pl_found ? div_select[lkbk_right] : na, offset=-lkbk_right, title='Bull', linewidth=1, color=bull_cond ? c_bull : c_na)

plotshape(bull_cond ? div_select[lkbk_right] : na, offset=-lkbk_right, title='Bullish Label', text=' Bull ', style=shape.labelup, location=location.absolute, color=c_bull, textcolor=c_text, display=display.none)


////
// Hidden Bullish
// Osc: Lower Low

osc_ll = div_select[lkbk_right] < ta.valuewhen(pl_found, div_select[lkbk_right], 1) and _inRange(pl_found[1])

// Price: Higher Low

price_hl = low[lkbk_right] > ta.valuewhen(pl_found, low[lkbk_right], 1)

hdn_bull_cond = plot_hdn_bull and price_hl and osc_ll and pl_found

plot(pl_found ? div_select[lkbk_right] : na, offset=-lkbk_right, title='Hidden Bull', linewidth=1, color=hdn_bull_cond ? c_hdn_bull : c_na)

plotshape(hdn_bull_cond ? div_select[lkbk_right] : na, offset=-lkbk_right, title='Hidden Bullish Label', text=' H. Bull ', style=shape.labelup, location=location.absolute, color=c_bull, textcolor=c_text, display=display.none)


////
// Bearish
// Osc: Lower High

osc_lh = div_select[lkbk_right] < ta.valuewhen(ph_found, div_select[lkbk_right], 1) and _inRange(ph_found[1])

// Price: Higher High

price_hh = high[lkbk_right] > ta.valuewhen(ph_found, high[lkbk_right], 1)

bear_cond = plot_bear and price_hh and osc_lh and ph_found

plot(ph_found ? div_select[lkbk_right] : na, offset=-lkbk_right, title='Bear', linewidth=1, color=bear_cond ? c_bear : c_na)

plotshape(bear_cond ? div_select[lkbk_right] : na, offset=-lkbk_right, title='Bearish Label', text=' Bear ', style=shape.labeldown, location=location.absolute, color=c_bear, textcolor=c_text, display=display.none)


////
// Hidden Bearish
// Osc: Higher High

osc_hh = div_select[lkbk_right] > ta.valuewhen(ph_found, div_select[lkbk_right], 1) and _inRange(ph_found[1])

// Price: Lower High

price_lh = high[lkbk_right] < ta.valuewhen(ph_found, high[lkbk_right], 1)

hdn_bear_cond = plot_hdn_bear and price_lh and osc_hh and ph_found

plot(ph_found ? div_select[lkbk_right] : na, offset=-lkbk_right, title='Hidden Bear', linewidth=1, color=hdn_bear_cond ? c_hdn_bear : c_na)

plotshape(hdn_bear_cond ? div_select[lkbk_right] : na, offset=-lkbk_right, title='Hidden Bearish Label', text=' H. Bear ', style=shape.labeldown, location=location.absolute, color=c_bear, textcolor=c_text, display=display.none)


////////////////////////////////////////////////////////////////////////////////


alertcondition(ta.cross(kvo, inner_upper), title='Klinger Oscillator Crossing into Upper Donchian Channel', message='The Klinger Oscillator has crossed into upper Donchian Channel.')
alertcondition(ta.cross(kvo, inner_lower), title='Klinger Oscillator Crossing into Lower Donchian Channel', message='The Klinger Oscillator has crossed into lower Donchian Channel.')
alertcondition(ta.cross(kvo, sig), title='Klinger Oscillator Crossing Signal Line', message='The Klinger Oscillator has crossed the signal line.')
alertcondition(ta.cross(kvo, 0), title='Klinger Oscillator Crossing Center Line', message='The Klinger Oscillator has crossed the center line.')
alertcondition(ta.cross(sig, 0), title='Signal Line Crossing Center Line', message='The Klinger Oscillator signal line has crossed the center line.')
alertcondition(bull_cond, title='Bull Div', message='+ Klinger Oscillator bull div')
alertcondition(hdn_bull_cond, title='Hidden Bull Div', message='+ Klinger Oscillator hidden bull div')
alertcondition(bear_cond, title='Bear Div', message='+ Klinger Oscillator bear div')
alertcondition(hdn_bear_cond, title='Hidden Bear Div', message='+ Klinger Oscillator hidden bear div')


