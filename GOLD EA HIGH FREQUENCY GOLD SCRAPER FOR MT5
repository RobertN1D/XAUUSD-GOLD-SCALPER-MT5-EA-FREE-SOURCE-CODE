//+------------------------------------------------------------------+
//|                                       GOLD-HFT-SCALPER.mq5       |
//|                  PURE-ALGORITHMIC HIGH-FREQUENCY GOLD SCALPER    |
//|                                                                  |
//|   No AI. No HTTP. No JSON. No WebRequest whitelist.              |
//|                                                                  |
//|   Multi-Strategy Setup-Ranking Engine:                           |
//|   ════════════════════════════════════════════════════════       |
//|     [1]  Spike / Momentum capture                                |
//|     [2]  Bull/Bear Flag continuation (M5)                        |
//|     [3]  Opening Range Breakout (London/NY)                      |
//|     [4]  VWAP Reclaim / Reject                                   |
//|     [5]  ABCD pattern continuation                               |
//|     [6]  Key-Level Bounce / Break (round, PDH/PDL/PWH/PWL)       |
//|     [7]  EMA pullback (M5 EMA9 / M15 EMA20) in trend             |
//|     [8]  Post-news continuation                                  |
//|     [9]  Fade (RSI extreme + magnet + exhaustion, tightest SL)   |
//|                                                                  |
//|   Each cycle every enabled generator proposes 0..1 setup with    |
//|   a numeric score. The Ranker picks the highest-scoring proposal |
//|   above InpMinSetupScore that clears global hard gates, then     |
//|   the proposal's own SL/targets are used (capped by risk inputs).|
//|   AutoProtect runs every tick to defend the trade.               |
//+------------------------------------------------------------------+
#property copyright "GOLD-HFT-SCALPER - Pure Algorithmic Multi-Strategy"
#property version   "1.11"
#property description "9-strategy point-ranking HFT Gold scalper. No AI. No HTTP. v1.11 = production-readiness pass: cached news block, DOM toggle, R:R-uncapped clamp for tight-SL setups, exit-tag queue, throttled redraw, adaptive retrace, weekend-gap close, 500-tick buffer."
#property strict

#include <Trade\Trade.mqh>

//+------------------------------------------------------------------+
//| FORWARD DECLARATIONS                                             |
//+------------------------------------------------------------------+
ENUM_TIMEFRAMES EffectiveEntryBarTF();
string          EntryBarTfLabel();
string          RegimeLabelForAI();
void            ScalperLogInit();
void            ScalperLog(const string evt, const string primary, const string secondary, const string detail);
string          ScalperEscapeField(const string s);
bool            BlockNewEntriesDueToHighImpactNews();
double          SpreadRoundTripUsdApprox();
int             CountPositions();
void            CheckSession();
void            RunAnalysis();
void            UpdateMarketData();
double          DollarsPerPriceUnit();

//+------------------------------------------------------------------+
//| INPUT PARAMETERS                                                 |
//+------------------------------------------------------------------+

// ─── Trade ────────────────────────────────────────────────────────
input group "═══ Trade ═══"
input double   InpLotSize             = 0.01;            // Base lot size (scaled by volatility-regime multiplier below)
input double   InpLotMulQuiet         = 0.5;             // Lot multiplier when regime = VOL_QUIET (cut size: dead market)
input double   InpLotMulNormal        = 1.0;             // Lot multiplier when regime = VOL_NORMAL (full size)
input double   InpLotMulElevated      = 1.0;             // Lot multiplier when regime = VOL_ELEVATED (full size: clean trends)
input double   InpLotMulExplosive     = 0.5;             // Lot multiplier when regime = VOL_EXPLOSIVE (cut size: slip risk)
input bool     InpAsymSize            = true;            // Asymmetric position sizing: scale lot by setup score band
input double   InpAsymMul_60_75       = 0.50;            // Lot multiplier for score 60..74 (weakest tradeable)
input double   InpAsymMul_75_85       = 0.75;            // Lot multiplier for score 75..84 (good)
input double   InpAsymMul_85_plus     = 1.00;            // Lot multiplier for score >= 85 (A+ setup)
input double   InpMaxRiskDollars      = 6.00;            // Hard cap risk per trade ($)
input int      InpMaxPositions        = 1;               // Max concurrent positions
input ENUM_TIMEFRAMES InpEntryNewBarTimeframe = PERIOD_M1; // 'New bar' timeframe for ranker cycle (M1=fastest, M5=calmer)
input bool     InpEntryOnNewBarOnly   = false;           // true: only run ranker on a fresh bar; false: throttle by interval only
input int      InpEntryIntervalSec    = 3;               // Min seconds between ranker cycles (HFT: keep small)
input int      InpOrderCommentMaxLen  = 31;              // MT5 broker comment length cap

// ─── Ranker ───────────────────────────────────────────────────────
input group "═══ Ranker ═══"
input double   InpMinSetupScore       = 60.0;            // Min final score required to fire (0..200 range typical)
input double   InpMinRRToFirstTarget  = 1.10;            // Min reward:risk to T1 to be allowed
input int      InpRankerLogTopN       = 3;               // Log top-N proposals per cycle (panel shows top 3)
input bool     InpVerboseSkips        = true;            // Print skip reasons (off = quieter logs)

// ─── Spike Setup ──────────────────────────────────────────────────
input group "═══ [1] Spike Setup (HFT impulse capture) ═══"
input bool     InpEnableSpike         = true;
input double   InpSpikeBodyAtrMult    = 1.8;             // Last M1 body must be ≥ this × ATR(M1) (raised: only catch real spikes, not noise)
input int      InpSpikeMinImbalance   = 35;              // |microstructure imbalance| ≥ this % (raised: avoid mid-spike chasing)
input double   InpSpikeMaxSpreadP90Mul= 1.10;            // Skip Spike if g_Spread > p90 × this (avoid spread-blowout entries)
input double   InpSpikeMinTickVel     = 8.0;             // |tick velocity| pts/sec
input double   InpSpikeSlAtrMult      = 0.6;             // SL = body-mid ± this × ATR(M1) (tight)
input double   InpSpikeRawScore       = 70.0;            // Strategy raw score (before context)

// ─── Flag Setup ───────────────────────────────────────────────────
input group "═══ [2] Bull/Bear Flag Setup ═══"
input bool     InpEnableFlag          = true;
input double   InpFlagMinM5TrendPct   = 70.0;            // Min M5 trend strength %
input int      InpFlagMaxPullbackBars = 4;               // Max counter-color pullback bars
input int      InpFlagMinPullbackBars = 1;               // Min pullback bars
input double   InpFlagSlAtrMult       = 1.0;             // SL = beyond pullback extreme + this × ATR(M5)
input double   InpFlagRawScore        = 65.0;            // Strategy raw score

// ─── ORB Setup ────────────────────────────────────────────────────
input group "═══ [3] Opening Range Breakout Setup ═══"
input bool     InpEnableORB           = true;
input int      InpORBFirstMinutes     = 15;              // Opening range minutes (anchored to InpVWAPAnchorMode)
input int      InpVWAPAnchorMode      = 0;               // 0=server midnight, 1=London open hour
input bool     InpORBRequireRetest    = false;           // true: must retest break-line + reject before fire
input double   InpORBSlAtrMult        = 1.1;             // SL = opposite ORB side + this × ATR(M5)
input double   InpORBRawScore         = 68.0;            // Strategy raw score

// ─── VWAP Setup ───────────────────────────────────────────────────
input group "═══ [4] VWAP Reclaim / Reject Setup ═══"
input bool     InpEnableVWAPReclaim   = true;
input bool     InpEnableVWAPReject    = true;
input int      InpVWAPMinDistPts      = 20;              // Min dist from VWAP (in points) for reclaim/reject signal
input double   InpVWAPSlAtrMult       = 1.0;             // SL distance multiplier
input double   InpVWAPRawScore        = 60.0;            // Strategy raw score

// ─── ABCD Setup ───────────────────────────────────────────────────
input group "═══ [5] ABCD Pattern Setup ═══"
input bool     InpEnableABCD          = true;
input int      InpABCDImpulseMinPts   = 60;              // Min A→B impulse (points)
input double   InpABCDPullbackMin     = 0.30;            // 30%–70% of impulse window
input double   InpABCDPullbackMax     = 0.70;
input double   InpABCDSlAtrMult       = 1.0;             // SL distance multiplier
input double   InpABCDRawScore        = 74.0;            // Strategy raw score (raised: noisy detection — needs strong context)
input double   InpABCDMinADX          = 22.0;            // ABCD invalid below this M5 ADX (no-trend filter)

// ─── Level Setup ──────────────────────────────────────────────────
input group "═══ [6] Key-Level Bounce / Break Setup ═══"
input bool     InpEnableLevelBounce   = true;
input bool     InpEnableLevelBreak    = true;
input int      InpLevelMaxDistPts     = 30;              // Max distance to a key level to consider it relevant
input double   InpLevelSlAtrMult      = 0.9;             // SL distance multiplier
input double   InpLevelRawScore       = 72.0;            // Strategy raw score (raised: requires ADX>=22 OR confirmed magnet)
input double   InpLevelMinADX         = 22.0;            // Level setups invalid below this ADX unless price is at a confirmed magnet

// ─── EMA Pullback Setup ───────────────────────────────────────────
input group "═══ [7] EMA Pullback Setup ═══"
input bool     InpEnableEMAPullback   = true;
input ENUM_TIMEFRAMES InpEMAPullbackTF = PERIOD_M5;      // M5 EMA9 (fast) or M15 EMA20 (calmer)
input int      InpEMAPullbackMaxPts   = 25;              // Max distance from EMA to count as 'touched'
input double   InpEMAPullbackSlAtrMult = 1.0;            // SL distance multiplier
input double   InpEMAPullbackRawScore = 58.0;            // Strategy raw score

// ─── News Continuation Setup ──────────────────────────────────────
input group "═══ [8] Post-News Continuation Setup ═══"
input bool     InpEnableNewsCont      = true;
input int      InpNewsContMaxSecAfter = 180;             // Max seconds since high-impact event close
input double   InpNewsContBodyAtrMult = 1.2;             // Last M1 body × ATR(M1) min for confirmation
input double   InpNewsContSlAtrMult   = 0.9;             // SL distance multiplier
input double   InpNewsContRawScore    = 75.0;            // Strategy raw score (high — explosive moves)

// ─── Fade Setup ───────────────────────────────────────────────────
input group "═══ [9] RSI Extreme Fade Setup (counter-trend) ═══"
input bool     InpEnableFade          = false;           // Disabled by default — pure RSI-extreme fades are coin-flips at retail latency
input int      InpFadeRsiLongCap      = 22;              // Long fade only if RSI ≤ this (oversold)
input int      InpFadeRsiShortFloor   = 78;              // Short fade only if RSI ≥ this (overbought)
input bool     InpFadeRequireMagnet   = true;            // Only at key magnet level
input bool     InpFadeRequireExhaustion = true;          // Require microstructure exhaustion
input double   InpFadeSlAtrMult       = 0.5;             // SL: very tight
input double   InpFadeRawScore        = 55.0;            // Strategy raw score (small bonus only)

// ─── Hard Gates ───────────────────────────────────────────────────
input group "═══ Hard Gates ═══"
input int      InpEntryMaxSpreadPts   = 40;              // Max spread (points) to allow entry
input bool     InpBlockEntryOnSpreadSpike = true;        // Block when spread > 1.25 × p90
input bool     InpBlockM15VsTrade     = true;            // Block long if M15 bearish, etc.
input bool     InpBlockH1VsTrade      = false;           // Same for H1
input int      InpRsiBlockLongAbove   = 75;              // 0=off
input int      InpRsiBlockShortBelow  = 25;              // 0=off
input bool     InpUseM1AlignFilter    = true;            // Block long if last N M1 bars all bearish
input int      InpM1AlignBars         = 3;
input bool     InpUseClosedM1RsiForEntry = true;
input int      InpRSIPeriod           = 7;               // M1 RSI
input bool     InpUseDomImbalance     = true;            // v1.11: include DOM imbalance in scoring (DISABLE on retail brokers w/ synthetic DOM)

// ─── Auto-Protect (every tick) ────────────────────────────────────
input group "═══ Auto-Protect (every tick) ═══"
input double   InpAPRunnerM5StrengthPct = 60.0;          // M5 trend % required to call it 'in a run'
input bool     InpAPAutoBankScalp     = true;            // Bank fast when not in a run
input double   InpAPScalpBankUsd      = 1.20;            // Bank if profit ≥ this $ and not running (raised: escape commission trap)
input bool     InpAPNoRedAfterGreen   = true;            // Close if max ≥ arm and now red
input double   InpAPGreenArmUsd       = 0.50;            // Green-armed level (raised: avoid arming on noise)
input bool     InpAPUniversalRetrace  = true;            // Retrace from peak (chop + run)
input double   InpAPRetraceChopUsd    = 0.40;            // Chop retrace $ from peak → bank (raised: let trade breathe)
input double   InpAPRetraceRunUsd     = 0.60;            // Run retrace $ from peak → bank (raised: let runners breathe)
input bool     InpAdaptiveRetrace     = true;            // v1.11: if too many ARTR exits today, auto-widen retrace tolerances
input int      InpAdaptiveRetraceTrigger = 3;            // # of ARTR exits today before widening kicks in
input double   InpAdaptiveRetraceMul  = 1.20;            // Multiplier applied to retrace thresholds when triggered (1.20 = +20% slack)
input bool     InpAPMicroTrail        = true;            // Lock slice of small greens with SL
input double   InpAPMicroLockFromUsd  = 0.80;            // First micro-lock arm (raised: don't lock sub-commission)
input double   InpAPWinnerGoneRedUsd  = 0.30;            // Close winner→loser at -this
input int      InpHardTimeStopSec     = 0;               // 0=off; if > 0, close after N sec idle (within ± InpHardTimeStopMinAbsUsd)
input double   InpHardTimeStopMinAbsUsd = 0.15;          // 'Idle' = |P/L| ≤ this $
input int      InpAPTrailIntervalSec  = 5;               // Periodic trail call (high-water style) interval

// ─── Profit / SL targets (defaults / floors) ──────────────────────
input group "═══ Profit / SL targets (defaults / floors) ═══"
input double   InpQuickProfitMin      = 1.50;            // Min target ($) (raised: ensure ≥ 1R after commission)
input double   InpQuickProfitGood     = 2.50;            // Good target ($)
input double   InpQuickProfitGreat    = 5.00;            // Great target ($)
input double   InpStopLossDollars     = 3.00;            // Fixed-$ SL fallback
input double   InpPartialPercent      = 60.0;            // Partial close %
input bool     InpOrderSLFromDynamicTargets = true;      // Use ATR-driven SL$ from g_Targets
input bool     InpCapOrderRiskToMinProfit = true;        // Cap risk to (minProfit × mult)
input double   InpMaxRiskVsMinTargetMult = 0.95;         // Max risk vs min target ratio
input double   InpMinProfitATRMult    = 0.8;             // T1 = ATR × this
input double   InpGoodProfitATRMult   = 1.5;             // T2 = ATR × this
input double   InpMaxProfitATRMult    = 2.5;             // T3 = ATR × this
input double   InpStopLossATRMult     = 1.2;             // SL distance = ATR × this

// ─── Intelligence ─────────────────────────────────────────────────
input group "═══ Intelligence ═══"
input int      InpTickBufferSize      = 200;
input int      InpATRPeriod           = 14;
input int      InpM15EMAPeriod        = 20;
input int      InpH1EMAPeriod         = 20;
input int      InpM5BarsForTrend      = 10;
input double   InpTrendStrength       = 0.5;
input int      InpMinTrendPoints      = 30;
input int      InpMomentumBars        = 3;

// ─── Risk Management ──────────────────────────────────────────────
input group "═══ Risk Management ═══"
input double   InpMaxDailyLoss        = 50.0;            // Daily loss halt ($)
input int      InpMaxConsecLosses     = 0;               // 0 = disabled
input int      InpCooldownAfterLoss   = 20;              // Sec
input int      InpCooldownAfterClose  = 20;              // Sec after any close
input bool     InpSuspendAfter2Losses = true;            // Per-setup: suspend a strategy after 2 consec losses on it
input int      InpSuspendDurationMin  = 45;              // Suspension duration (minutes) before that setup can fire again
input double   InpDailyProfitTarget   = 20.0;            // Target $ for the day (panel progress bar reference)

// ─── Sessions / News ──────────────────────────────────────────────
input group "═══ Sessions / News ═══"
input bool     InpUseSessionFilter    = true;
input int      InpLondonStart         = 8;               // London start (server hour)
input int      InpLondonEnd           = 16;
input int      InpNYStart             = 13;
input int      InpNYEnd               = 21;
input int      InpSessionOpenWindowMin = 15;             // Minutes after open considered 'session open window' (boost SPK/NEWS, penalize FLG/ABCD)
input bool     InpAvoidWeekendGap     = true;            // Friday after InpFridayCloseHour: stop entries (gap protection)
input int      InpFridayCloseHour     = 20;              // Server hour on Friday to stop new entries (default 20 = 1h before NY end)
input bool     InpBlockHighImpactNewEntries = true;      // Block within window of HIGH event
input int      InpHighImpactNewsBeforeMin = 15;          // Min before
input int      InpHighImpactNewsAfterMin  = 10;          // Min after
input int      InpNewsNearPrintThrottleSec = 60;

// ─── Close-send retries ───────────────────────────────────────────
input group "═══ Close-send retries ═══"
input int      InpCloseSendMaxRetries = 5;
input int      InpCloseSendRetryDelayMs = 280;

// ─── Logging ──────────────────────────────────────────────────────
input group "═══ Logging ═══"
input bool     InpEnableCsvLog        = true;
// Path is RELATIVE to <MT5 Data Folder>\MQL5\Files\
// Default writes to:  MQL5\Files\GOLD-HFT-SCALPER\Logs\Gold-HFT-Scalper-journal.csv
// MT5 auto-creates the GOLD-HFT-SCALPER\Logs subfolders on first write.
// Open via MT5 menu: File -> Open Data Folder -> MQL5\Files\GOLD-HFT-SCALPER\Logs\
input string   InpCsvLogFile          = "GOLD-HFT-SCALPER\\Logs\\Gold-HFT-Scalper-journal.csv";

// ─── Display ──────────────────────────────────────────────────────
input group "═══ Display ═══"
input bool     InpShowPanel           = true;
input int      InpPanelX              = 10;              // Panel X anchor (px from left)
input int      InpPanelY              = 25;              // Panel Y anchor (px from top)
input int      InpPanelWidth          = 380;             // Panel width (px) — wider = more diagnostic info
input int      InpPanelRedrawEveryN   = 5;               // Throttle ChartRedraw to every N panel updates (1 = every tick, expensive on live)
input color    ClrBullish             = C'50,205,50';
input color    ClrBearish             = C'220,50,50';
input color    ClrNeutral             = C'128,128,128';
input color    ClrWarning             = C'255,165,0';
input color    ClrAccent              = C'100,180,255';
input color    ClrGold                = C'255,200,50';

//+------------------------------------------------------------------+
//| CONSTANTS                                                        |
//+------------------------------------------------------------------+
#define MAGIC_NUMBER     998877
#define MAX_TICKS        500   // v1.11: bumped 200 -> 500 (NY-open Gold can fire 200 ticks/sec)
#define SPREAD_RING_SIZE 200
#define MAX_PROPOSALS    24

//+------------------------------------------------------------------+
//| ENUMS                                                            |
//+------------------------------------------------------------------+
enum VOLATILITY_REGIME
{
   VOL_QUIET,
   VOL_NORMAL,
   VOL_ELEVATED,
   VOL_EXPLOSIVE
};

enum EVENT_IMPACT
{
   IMPACT_LOW,
   IMPACT_MEDIUM,
   IMPACT_HIGH,
   IMPACT_HOLIDAY
};

enum SETUP_KIND
{
   SETUP_NONE = 0,
   SETUP_SPIKE,
   SETUP_FLAG,
   SETUP_ORB,
   SETUP_VWAP_RECLAIM,
   SETUP_VWAP_REJECT,
   SETUP_ABCD,
   SETUP_LEVEL_BOUNCE,
   SETUP_LEVEL_BREAK,
   SETUP_EMA_PULLBACK,
   SETUP_NEWS_CONT,
   SETUP_FADE
};

//+------------------------------------------------------------------+
//| STRUCTS                                                          |
//+------------------------------------------------------------------+
struct TickData
{
   double   bid;
   double   ask;
   datetime time;
   ulong    time_msc;
   double   bidChange;
   double   askChange;
   int      direction;
};

struct MicrostructureState
{
   double   tickVelocity;
   double   tickAcceleration;
   int      consecutiveUp;
   int      consecutiveDown;
   double   bidPressure;
   double   askPressure;
   double   imbalance;
   bool     momentumBurst;
   bool     exhaustion;
   string   regime;
};

struct KeyLevels
{
   double   nearestRoundAbove;
   double   nearestRoundBelow;
   double   distToRoundAbove;
   double   distToRoundBelow;
   double   sessionHigh;
   double   sessionLow;
   double   distToSessionHigh;
   double   distToSessionLow;
   double   recentPivotHigh;
   double   recentPivotLow;
   bool     nearMagnet;
   string   magnetType;
};

struct DynamicTargets
{
   double   minProfit;
   double   goodProfit;
   double   greatProfit;
   double   stopLoss;
   double   stopLossDollars;
   double   trailDistance;
   double   breakEvenTrigger;
};

struct EconomicEvent
{
   datetime    time;
   string      name;
   string      currency;
   EVENT_IMPACT impact;
   int         minutesUntil;
   bool        isGoldRelated;
};

struct NewsState
{
   bool        hasUpcoming;
   bool        inBlackout;
   int         minutesToNext;
   string      nextEventName;
   EVENT_IMPACT nextEventImpact;
   int         highImpactCount;
   string      warningMessage;
   bool        tradingRestricted;
};

// SETUP PROPOSAL — uniform output of every strategy generator
struct SetupProposal
{
   bool       valid;
   SETUP_KIND kind;
   string     name;          // e.g. "BULL_FLAG_M5"
   string     tag;           // short order-comment tag (≤4 chars), e.g. "BFG"
   bool       isBuy;
   double     score;         // final = raw + context bonus
   double     rawScore;      // before context bonus
   double     ctxScore;      // contextual bonus
   double     entryRef;      // intended entry price (mkt for now)
   double     stopLoss;      // SL price (strategy-aware)
   double     t1, t2, t3;    // ladder targets (price)
   double     riskUsd;       // computed at lot size from |entryRef-stopLoss|
   double     rrToT1;        // |t1-entryRef| / |entryRef-stopLoss|
   string     factors;       // human-readable score breakdown
};

//+------------------------------------------------------------------+
//| GLOBAL VARIABLES                                                 |
//+------------------------------------------------------------------+
string         g_Prefix = "GOLDHFT_";

// Indicators
int            g_HandleM15EMA;
int            g_HandleH1EMA;
int            g_HandleM5EMA;
int            g_HandleM1RSI;
int            g_HandleM1ATR;
int            g_HandleM5ATR;
int            g_HandleWPR;
int            g_HandleM5ADX;
int            g_HandleM5Bands;

// M5 enhanced analysis
double         g_M5EMAValue = 0;
int            g_M5EMABias = 0;
bool           g_M5MomentumBurst = false;
double         g_M5BiggestCandle = 0;

// Symbol info
int            g_Digits;
double         g_Point;
double         g_TickValue;
double         g_TickSize;

// Trading
CTrade         g_Trade;
datetime       g_LastEntryBarOpenTime = 0;
datetime       g_LastRankerCycle = 0;
datetime       g_LastNewsCheck = 0;
datetime       g_LastNewsThrottlePrint = 0;
string         g_LastNewsThrottleSig = "";
datetime       g_LastTrailCall = 0;

// Tick velocity engine
TickData       g_TickBuffer[];
int            g_TickCount = 0;
int            g_TickHead = 0;
MicrostructureState g_Micro;

// Key levels
KeyLevels      g_Levels;

// Volatility regime
VOLATILITY_REGIME g_VolRegime = VOL_NORMAL;
double         g_CurrentATR = 0;
double         g_M5ATR = 0;
double         g_AvgATR = 0;

// Dynamic targets
DynamicTargets g_Targets;

// News / calendar
NewsState      g_News;
EconomicEvent  g_UpcomingEvents[];
int            g_EventCount = 0;
datetime       g_LastHighImpactEventTime = 0;  // most-recent past HIGH event

// Analysis state
int            g_M15Bias = 0;
int            g_H1Bias = 0;
int            g_M5Trend = 0;
double         g_M5TrendPct = 0;
double         g_M5Movement = 0;
bool           g_M5LastClosedBull = false;
bool           g_M5LastClosedBear = false;
bool           g_M5MomentumOK = false;
int            g_M1Setup = 0;
double         g_M1RSI = 50;
double         g_M1RsiBar1 = 50;
double         g_M1ATR = 1.0;
double         g_WPR = -50;
string         g_Session = "";
bool           g_SessionOK = false;
double         g_Spread = 0;

// Intraday structure
double         g_SessionVWAP = 0;
bool           g_VWAPValid = false;
double         g_ORBHigh = 0;
double         g_ORBLow = 0;
bool           g_ORBValid = false;

// Position tracking
ulong          g_Ticket = 0;
bool           g_PartialTaken = false;
double         g_EntryPrice = 0;
double         g_MaxProfit = 0;
double         g_CurrentProfit = 0;
ENUM_POSITION_TYPE g_PosType;
datetime       g_EntryTime = 0;
SETUP_KIND     g_OpenSetupKind = SETUP_NONE;
string         g_OpenSetupName = "";
double         g_OpenSetupScore = 0;

// Risk
double         g_DailyPnL = 0;
int            g_ConsecLosses = 0;
datetime       g_LastLossTime = 0;
datetime       g_LastTradeCloseTime = 0;
double         g_LastTradeResult = 0;
datetime       g_DayStart = 0;
bool           g_Halted = false;
int            g_TodayTrades = 0;
int            g_TodayWins = 0;

// Stats
double         g_RecentResults[5];
int            g_RecentResultCount = 0;
int            g_RecentWins = 0;
int            g_RecentLosses = 0;
double         g_RecentM5Move = 0;
double         g_RecentM1Move = 0;

// MT5 extended intel
double         g_SpreadSamples[];
int            g_SpreadWriteIdx = 0;
int            g_SpreadSampleCount = 0;
double         g_SpreadMedian = 0;
double         g_SpreadP90 = 0;
double         g_SpreadMaxRecent = 0;
bool           g_SpreadSpike = false;
int            g_TicksLast10s = 0;
int            g_TicksLast60s = 0;

double         g_PDH = 0, g_PDL = 0, g_PWH = 0, g_PWL = 0;
double         g_DistPDH = 0, g_DistPDL = 0, g_DistPWH = 0, g_DistPWL = 0;

string         g_BrokerSessionSummary = "";
long           g_SymStopsLevel = 0;
long           g_SymFreezeLevel = 0;
string         g_SymTradeModeStr = "";
string         g_SymFillingStr = "";

bool           g_MarketBookSubscribed = false;
double         g_DomBidVol = 0;
double         g_DomAskVol = 0;
double         g_DomImbalancePct = 0;

bool           g_VolumeDataOK = false;
long           g_M1VolLast = 0;
long           g_M1VolSum20 = 0;

double         g_M5ADX = 0;
double         g_M5BBWidthPct = 0;

double         g_FreeMargin = 0;
double         g_MarginLevel = 0;
double         g_AccountEquity = 0;

double         g_LastEntryRequestPrice = 0;
double         g_LastSlippagePts = 0;

ulong          g_LogOpenPositionId = 0;
double         g_LogSnapshotScore  = 0;
string         g_LogSnapshotRegime = "";
string         g_LogSnapshotSetup  = "";

// Ranker telemetry — top 3 of last cycle for panel + journal
SetupProposal  g_LastTop[3];
int            g_LastTopCount = 0;
string         g_LastRankerVerdict = "";       // FIRED / BELOW_THRESH / GATED / NO_PROPOSAL
string         g_LastBlockReasonForTop[3] = {"", "", ""}; // why each top-3 was gated (full text, not truncated)
int            g_LastProposalCount = 0;        // how many proposals last cycle generated

// Session microstructure (set in CheckSession)
bool           g_InSessionOpenWindow_London = false;
bool           g_InSessionOpenWindow_NY     = false;
int            g_MinutesSinceLondonOpen     = -1;  // -1 = not in window
int            g_MinutesSinceNYOpen         = -1;
int            g_MinutesUntilSessionClose   = -1;  // for the active session

// Per-setup-kind suspension after consecutive losses
// Index by SETUP_KIND enum value
int            g_KindLossStreak[12];
datetime       g_KindSuspendUntil[12];

// v1.11: Last close tag is now a small FIFO queue so partial-close races
// (TP partial then full close in the same tick) each get the correct tag.
#define CLOSE_TAG_QUEUE_SIZE 8
string         g_CloseTagQueue[CLOSE_TAG_QUEUE_SIZE];
int            g_CloseTagHead = 0;     // next slot to push
int            g_CloseTagTail = 0;     // next slot to pop
// Backwards-compat alias kept so any external code that referenced the old
// single-string still compiles; internal code goes through Push/Pop.
string         g_LastCloseTag = "";

// v1.11: Cached news block (BlockNewEntriesDueToHighImpactNews now reads this
// instead of issuing CalendarValueHistory on every gate check). Populated by
// CheckEconomicCalendar once per minute.
bool           g_NewsHighImpactBlockActive = false;
string         g_NewsHighImpactBlockReason = "";

// v1.11: Panel redraw throttle counter (UpdatePanel only calls ChartRedraw
// every InpPanelRedrawEveryN ticks).
int            g_PanelRedrawCounter = 0;

// Exit-reason in-memory tally for panel (for today)
// Tags: ANRD, ARTR, ABNK, AGVE, AEMR, ACUT, ATIM, MPRT, SL, TP, MAN, OTH
int            g_ExitTallyCount = 0;
string         g_ExitTallyTags[12];
int            g_ExitTallyHits[12];

// Win/loss running averages for panel
double         g_TodayWinSum  = 0;
double         g_TodayLossSum = 0;
int            g_TodayLossCnt = 0;

//+------------------------------------------------------------------+
//| OnInit                                                           |
//+------------------------------------------------------------------+
int OnInit()
{
   g_Digits = (int)SymbolInfoInteger(_Symbol, SYMBOL_DIGITS);
   g_Point = SymbolInfoDouble(_Symbol, SYMBOL_POINT);
   g_TickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   g_TickSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   
   g_HandleM15EMA = iMA(_Symbol, PERIOD_M15, InpM15EMAPeriod, 0, MODE_EMA, PRICE_CLOSE);
   g_HandleH1EMA  = iMA(_Symbol, PERIOD_H1,  InpH1EMAPeriod,  0, MODE_EMA, PRICE_CLOSE);
   g_HandleM5EMA  = iMA(_Symbol, PERIOD_M5,  9,               0, MODE_EMA, PRICE_CLOSE);
   g_HandleM1RSI  = iRSI(_Symbol, PERIOD_M1, InpRSIPeriod, PRICE_CLOSE);
   g_HandleM1ATR  = iATR(_Symbol, PERIOD_M1, InpATRPeriod);
   g_HandleM5ATR  = iATR(_Symbol, PERIOD_M5, InpATRPeriod);
   g_HandleWPR    = iWPR(_Symbol, PERIOD_M1, 14);
   g_HandleM5ADX  = iADX(_Symbol, PERIOD_M5, 14);
   g_HandleM5Bands= iBands(_Symbol, PERIOD_M5, 20, 0, 2.0, PRICE_CLOSE);
   
   if(g_HandleM15EMA == INVALID_HANDLE || g_HandleH1EMA == INVALID_HANDLE || g_HandleM5EMA == INVALID_HANDLE ||
      g_HandleM1RSI == INVALID_HANDLE || g_HandleM1ATR == INVALID_HANDLE || g_HandleM5ATR == INVALID_HANDLE ||
      g_HandleWPR == INVALID_HANDLE || g_HandleM5ADX == INVALID_HANDLE || g_HandleM5Bands == INVALID_HANDLE)
   {
      Print("ERROR: Failed to create indicator handles");
      return INIT_FAILED;
   }
   
   ArrayResize(g_TickBuffer, MAX_TICKS);
   ArrayResize(g_SpreadSamples, SPREAD_RING_SIZE);
   g_TickCount = 0;
   g_TickHead = 0;
   g_SpreadWriteIdx = 0;
   g_SpreadSampleCount = 0;
   
   g_MarketBookSubscribed = MarketBookAdd(_Symbol);
   if(g_MarketBookSubscribed)
      Print("DOM subscribed: ", _Symbol);
   
   ZeroMemory(g_Micro);
   ZeroMemory(g_Levels);
   ZeroMemory(g_Targets);
   ZeroMemory(g_News);
   ArrayResize(g_UpcomingEvents, 0);
   g_EventCount = 0;
   g_LastNewsCheck = 0;
   g_LastTopCount = 0;
   g_LastRankerVerdict = "INIT";
   
   // Per-setup suspension state
   for(int kk = 0; kk < 12; kk++)
   {
      g_KindLossStreak[kk]   = 0;
      g_KindSuspendUntil[kk] = 0;
   }
   // Exit-tally
   g_ExitTallyCount = 0;
   for(int ee = 0; ee < 12; ee++) { g_ExitTallyTags[ee] = ""; g_ExitTallyHits[ee] = 0; }
   for(int bb = 0; bb < 3; bb++)  g_LastBlockReasonForTop[bb] = "";
   // v1.11: close-tag queue + news cache + panel redraw counter
   for(int qq = 0; qq < CLOSE_TAG_QUEUE_SIZE; qq++) g_CloseTagQueue[qq] = "";
   g_CloseTagHead = 0;
   g_CloseTagTail = 0;
   g_NewsHighImpactBlockActive = false;
   g_NewsHighImpactBlockReason = "";
   g_PanelRedrawCounter = 0;
   
   g_Trade.SetExpertMagicNumber(MAGIC_NUMBER);
   g_Trade.SetDeviationInPoints(10);
   
   long fillMode = SymbolInfoInteger(_Symbol, SYMBOL_FILLING_MODE);
   if((fillMode & SYMBOL_FILLING_FOK) == SYMBOL_FILLING_FOK)
      g_Trade.SetTypeFilling(ORDER_FILLING_FOK);
   else if((fillMode & SYMBOL_FILLING_IOC) == SYMBOL_FILLING_IOC)
      g_Trade.SetTypeFilling(ORDER_FILLING_IOC);
   else
      g_Trade.SetTypeFilling(ORDER_FILLING_RETURN);
   
   CheckNewDay();
   ScalperLogInit();
   
   if(InpShowPanel) DrawPanel();
   
   UpdateMT5ExtendedIntel();
   
   Print("═══════════════════════════════════════════════════════════════");
   Print("⚡ GOLD-HFT-SCALPER v1.11 — Pure Algorithmic Multi-Strategy");
   Print("   No AI, no HTTP. 9-strategy point-ranking engine.");
   Print("   MinSetupScore=", DoubleToString(InpMinSetupScore, 0),
         " | MinR:R=", DoubleToString(InpMinRRToFirstTarget, 2),
         " | EntryTF=", EntryBarTfLabel(),
         " | IntervalSec=", InpEntryIntervalSec);
   Print("═══════════════════════════════════════════════════════════════");
   
   RunStartupChecks();
   CheckEconomicCalendar();
   
   return INIT_SUCCEEDED;
}

//+------------------------------------------------------------------+
//| OnDeinit                                                         |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   if(g_MarketBookSubscribed)
   {
      MarketBookRelease(_Symbol);
      g_MarketBookSubscribed = false;
   }
   if(g_HandleM15EMA  != INVALID_HANDLE) IndicatorRelease(g_HandleM15EMA);
   if(g_HandleH1EMA   != INVALID_HANDLE) IndicatorRelease(g_HandleH1EMA);
   if(g_HandleM5EMA   != INVALID_HANDLE) IndicatorRelease(g_HandleM5EMA);
   if(g_HandleM1RSI   != INVALID_HANDLE) IndicatorRelease(g_HandleM1RSI);
   if(g_HandleM1ATR   != INVALID_HANDLE) IndicatorRelease(g_HandleM1ATR);
   if(g_HandleM5ATR   != INVALID_HANDLE) IndicatorRelease(g_HandleM5ATR);
   if(g_HandleWPR     != INVALID_HANDLE) IndicatorRelease(g_HandleWPR);
   if(g_HandleM5ADX   != INVALID_HANDLE) IndicatorRelease(g_HandleM5ADX);
   if(g_HandleM5Bands != INVALID_HANDLE) IndicatorRelease(g_HandleM5Bands);
   ObjectsDeleteAll(0, g_Prefix);
}

//+------------------------------------------------------------------+
//| TICK ENGINE                                                      |
//+------------------------------------------------------------------+
void RecordTick()
{
   MqlTick tick;
   if(!SymbolInfoTick(_Symbol, tick)) return;
   
   double prevBid = 0, prevAsk = 0;
   if(g_TickCount > 0)
   {
      int prevIdx = (g_TickHead - 1 + MAX_TICKS) % MAX_TICKS;
      prevBid = g_TickBuffer[prevIdx].bid;
      prevAsk = g_TickBuffer[prevIdx].ask;
   }
   
   g_TickBuffer[g_TickHead].bid = tick.bid;
   g_TickBuffer[g_TickHead].ask = tick.ask;
   g_TickBuffer[g_TickHead].time = tick.time;
   g_TickBuffer[g_TickHead].time_msc = tick.time_msc;
   g_TickBuffer[g_TickHead].bidChange = (prevBid > 0) ? (tick.bid - prevBid) : 0;
   g_TickBuffer[g_TickHead].askChange = (prevAsk > 0) ? (tick.ask - prevAsk) : 0;
   
   if(g_TickBuffer[g_TickHead].bidChange > 0)      g_TickBuffer[g_TickHead].direction = 1;
   else if(g_TickBuffer[g_TickHead].bidChange < 0) g_TickBuffer[g_TickHead].direction = -1;
   else                                            g_TickBuffer[g_TickHead].direction = 0;
   
   g_TickHead = (g_TickHead + 1) % MAX_TICKS;
   if(g_TickCount < MAX_TICKS) g_TickCount++;
   
   if(g_Point > 0 && ArraySize(g_SpreadSamples) >= SPREAD_RING_SIZE)
   {
      double sp = (tick.ask - tick.bid) / g_Point;
      g_SpreadSamples[g_SpreadWriteIdx] = sp;
      g_SpreadWriteIdx = (g_SpreadWriteIdx + 1) % SPREAD_RING_SIZE;
      if(g_SpreadSampleCount < SPREAD_RING_SIZE) g_SpreadSampleCount++;
   }
}

void AnalyzeMicrostructure()
{
   if(g_TickCount < 30) return;
   
   // v1.11: dynamic lookback. Base = 30 ticks, but if those 30 span <1 second
   // (typical at NY-open in Gold) we widen the window until we have at least
   // 1s of data or hit the cap. This stabilises the velocity/acceleration
   // math when ticks fire 100+/sec. Also constrained by MAX_TICKS-1.
   const double MIN_LOOKBACK_SEC = 1.0;
   const int    LOOKBACK_HARD_CAP = MathMin(MAX_TICKS - 1, 250);
   int lookback = MathMin(30, g_TickCount);
   int endIdx = (g_TickHead - 1 + MAX_TICKS) % MAX_TICKS;
   int startIdx = (g_TickHead - lookback + MAX_TICKS) % MAX_TICKS;
   ulong timeDiff = g_TickBuffer[endIdx].time_msc - g_TickBuffer[startIdx].time_msc;
   while((double)timeDiff / 1000.0 < MIN_LOOKBACK_SEC && lookback < g_TickCount && lookback < LOOKBACK_HARD_CAP)
   {
      lookback++;
      startIdx = (g_TickHead - lookback + MAX_TICKS) % MAX_TICKS;
      timeDiff = g_TickBuffer[endIdx].time_msc - g_TickBuffer[startIdx].time_msc;
   }
   
   double totalMove = 0;
   int upTicks = 0, downTicks = 0;
   double bidPressure = 0, askPressure = 0;
   
   if(timeDiff == 0) timeDiff = 1;
   double seconds = (double)timeDiff / 1000.0;
   
   for(int i = 0; i < lookback; i++)
   {
      int idx = (startIdx + i) % MAX_TICKS;
      totalMove += g_TickBuffer[idx].bidChange;
      
      if(g_TickBuffer[idx].direction > 0) upTicks++;
      else if(g_TickBuffer[idx].direction < 0) downTicks++;
      
      if(g_TickBuffer[idx].bidChange > 0) bidPressure += MathAbs(g_TickBuffer[idx].bidChange);
      if(g_TickBuffer[idx].bidChange < 0) askPressure += MathAbs(g_TickBuffer[idx].bidChange);
   }
   
   double prevVelocity = g_Micro.tickVelocity;
   if(seconds > 0 && g_Point > 0)
      g_Micro.tickVelocity = (totalMove / g_Point) / seconds;
   else
      g_Micro.tickVelocity = 0;
   
   g_Micro.tickAcceleration = g_Micro.tickVelocity - prevVelocity;
   
   g_Micro.consecutiveUp = 0;
   g_Micro.consecutiveDown = 0;
   for(int i = lookback - 1; i >= 0; i--)
   {
      int idx = (startIdx + i) % MAX_TICKS;
      if(g_TickBuffer[idx].direction > 0)
      {
         if(g_Micro.consecutiveDown == 0) g_Micro.consecutiveUp++;
         else break;
      }
      else if(g_TickBuffer[idx].direction < 0)
      {
         if(g_Micro.consecutiveUp == 0) g_Micro.consecutiveDown++;
         else break;
      }
      else break;
   }
   
   double totalPressure = bidPressure + askPressure;
   if(totalPressure > 0)
   {
      g_Micro.bidPressure = (bidPressure / totalPressure) * 100;
      g_Micro.askPressure = (askPressure / totalPressure) * 100;
   }
   
   int totalDirTicks = upTicks + downTicks;
   if(totalDirTicks > 0)
      g_Micro.imbalance = ((double)(upTicks - downTicks) / (double)totalDirTicks) * 100;
   else
      g_Micro.imbalance = 0;
   
   g_Micro.momentumBurst = (MathAbs(g_Micro.tickVelocity) > 50);
   g_Micro.exhaustion    = (MathAbs(g_Micro.tickVelocity) > 30 && g_Micro.tickAcceleration * g_Micro.tickVelocity < 0);
   
   switch(g_VolRegime)
   {
      case VOL_QUIET:     g_Micro.regime = "QUIET"; break;
      case VOL_NORMAL:    g_Micro.regime = "NORMAL"; break;
      case VOL_ELEVATED:  g_Micro.regime = "ELEVATED"; break;
      case VOL_EXPLOSIVE: g_Micro.regime = "EXPLOSIVE"; break;
   }
}

//+------------------------------------------------------------------+
//| Volatility regime + dynamic targets                              |
//+------------------------------------------------------------------+
void DetectVolatilityRegime()
{
   double atr[];
   ArraySetAsSeries(atr, true);
   if(CopyBuffer(g_HandleM1ATR, 0, 0, 20, atr) < 20) return;
   
   g_CurrentATR = atr[0];
   double sumATR = 0;
   for(int i = 0; i < 20; i++) sumATR += atr[i];
   g_AvgATR = sumATR / 20.0;
   
   double m5atr[];
   ArraySetAsSeries(m5atr, true);
   if(CopyBuffer(g_HandleM5ATR, 0, 0, 1, m5atr) >= 1)
      g_M5ATR = m5atr[0];
   
   double ratio = (g_AvgATR > 0) ? g_CurrentATR / g_AvgATR : 1.0;
   if(ratio < 0.6)       g_VolRegime = VOL_QUIET;
   else if(ratio < 1.2)  g_VolRegime = VOL_NORMAL;
   else if(ratio < 2.0)  g_VolRegime = VOL_ELEVATED;
   else                  g_VolRegime = VOL_EXPLOSIVE;
}

void CalculateDynamicTargets()
{
   DetectVolatilityRegime();
   
   if(g_Point <= 0 || g_TickSize <= 0)
   {
      g_Targets.minProfit = InpQuickProfitMin;
      g_Targets.goodProfit = InpQuickProfitGood;
      g_Targets.greatProfit = InpQuickProfitGreat;
      g_Targets.stopLossDollars = InpStopLossDollars;
      g_Targets.stopLoss = 100 * 0.01;
      g_Targets.breakEvenTrigger = g_Targets.minProfit * 0.6;
      return;
   }
   
   double atrPoints = g_CurrentATR / g_Point;
   if(atrPoints <= 0) atrPoints = 100;
   
   double dollarsPerPoint = g_TickValue * InpLotSize / g_TickSize;
   if(dollarsPerPoint <= 0) dollarsPerPoint = 0.01;
   
   double atrDollars = atrPoints * dollarsPerPoint;
   g_Targets.minProfit = atrDollars * InpMinProfitATRMult;
   g_Targets.goodProfit = atrDollars * InpGoodProfitATRMult;
   g_Targets.greatProfit = atrDollars * InpMaxProfitATRMult;
   
   switch(g_VolRegime)
   {
      case VOL_QUIET:
         g_Targets.minProfit *= 0.7;
         g_Targets.goodProfit *= 0.7;
         g_Targets.greatProfit *= 0.7;
         break;
      case VOL_ELEVATED:
         g_Targets.minProfit *= 1.3;
         g_Targets.goodProfit *= 1.3;
         g_Targets.greatProfit *= 1.3;
         break;
      case VOL_EXPLOSIVE:
         g_Targets.minProfit *= 1.8;
         g_Targets.goodProfit *= 1.8;
         g_Targets.greatProfit *= 2.0;
         break;
      default: break;
   }
   
   if(g_Targets.minProfit < 1.00) g_Targets.minProfit = 1.00;
   if(g_Targets.goodProfit < 2.00) g_Targets.goodProfit = 2.00;
   if(g_Targets.greatProfit < 4.00) g_Targets.greatProfit = 4.00;
   if(g_Targets.minProfit > 5.00) g_Targets.minProfit = 5.00;
   if(g_Targets.goodProfit > 10.00) g_Targets.goodProfit = 10.00;
   if(g_Targets.greatProfit > 20.00) g_Targets.greatProfit = 20.00;
   
   double slPoints = atrPoints * InpStopLossATRMult;
   double minSL = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * g_Point;
   if(slPoints * g_Point < minSL) slPoints = minSL / g_Point;
   
   g_Targets.stopLoss = slPoints * g_Point;
   g_Targets.stopLossDollars = slPoints * dollarsPerPoint;
   
   if(g_Targets.stopLossDollars > InpMaxRiskDollars)
   {
      g_Targets.stopLossDollars = InpMaxRiskDollars;
      g_Targets.stopLoss = (InpMaxRiskDollars / dollarsPerPoint) * g_Point;
   }
   
   g_Targets.trailDistance = g_CurrentATR * 0.8;
   g_Targets.breakEvenTrigger = g_Targets.minProfit * 0.6;
}

//+------------------------------------------------------------------+
//| Key levels                                                       |
//+------------------------------------------------------------------+
void DetectKeyLevels()
{
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double roundInterval = 10.0;  // gold $10 grid
   
   g_Levels.nearestRoundBelow = MathFloor(bid / roundInterval) * roundInterval;
   g_Levels.nearestRoundAbove = g_Levels.nearestRoundBelow + roundInterval;
   g_Levels.distToRoundBelow = (bid - g_Levels.nearestRoundBelow) / g_Point;
   g_Levels.distToRoundAbove = (g_Levels.nearestRoundAbove - bid) / g_Point;
   
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   datetime dayStart = StringToTime(IntegerToString(dt.year) + "." +
                                    IntegerToString(dt.mon) + "." +
                                    IntegerToString(dt.day));
   
   int barsToday = Bars(_Symbol, PERIOD_M1, dayStart, TimeCurrent());
   if(barsToday > 0)
   {
      double high[], low[];
      ArraySetAsSeries(high, true);
      ArraySetAsSeries(low, true);
      
      if(CopyHigh(_Symbol, PERIOD_M1, 0, barsToday, high) == barsToday &&
         CopyLow(_Symbol, PERIOD_M1, 0, barsToday, low) == barsToday)
      {
         g_Levels.sessionHigh = high[ArrayMaximum(high)];
         g_Levels.sessionLow = low[ArrayMinimum(low)];
         g_Levels.distToSessionHigh = (g_Levels.sessionHigh - bid) / g_Point;
         g_Levels.distToSessionLow = (bid - g_Levels.sessionLow) / g_Point;
      }
   }
   
   double h5[], l5[];
   ArraySetAsSeries(h5, true);
   ArraySetAsSeries(l5, true);
   if(CopyHigh(_Symbol, PERIOD_M5, 0, 20, h5) == 20 &&
      CopyLow(_Symbol, PERIOD_M5, 0, 20, l5) == 20)
   {
      g_Levels.recentPivotHigh = h5[0];
      g_Levels.recentPivotLow = l5[0];
      for(int i = 1; i < 20; i++)
      {
         if(h5[i] > g_Levels.recentPivotHigh) g_Levels.recentPivotHigh = h5[i];
         if(l5[i] < g_Levels.recentPivotLow) g_Levels.recentPivotLow = l5[i];
      }
   }
   
   g_Levels.nearMagnet = false;
   g_Levels.magnetType = "";
   double magnetThreshold = g_CurrentATR * 0.3;
   
   if(g_Levels.distToRoundAbove * g_Point < magnetThreshold ||
      g_Levels.distToRoundBelow * g_Point < magnetThreshold)
   {
      g_Levels.nearMagnet = true;
      g_Levels.magnetType = "ROUND";
   }
   else if(MathAbs(g_Levels.distToSessionHigh) * g_Point < magnetThreshold)
   {
      g_Levels.nearMagnet = true;
      g_Levels.magnetType = "SESSION_HIGH";
   }
   else if(MathAbs(g_Levels.distToSessionLow) * g_Point < magnetThreshold)
   {
      g_Levels.nearMagnet = true;
      g_Levels.magnetType = "SESSION_LOW";
   }
}

//+------------------------------------------------------------------+
//| Economic calendar                                                |
//+------------------------------------------------------------------+
void CheckEconomicCalendar()
{
   if(TimeCurrent() - g_LastNewsCheck < 60) return;
   g_LastNewsCheck = TimeCurrent();
   
   ZeroMemory(g_News);
   ArrayResize(g_UpcomingEvents, 0);
   g_EventCount = 0;
   
   // v1.11: reset news-block cache; will be re-evaluated below
   g_NewsHighImpactBlockActive = false;
   g_NewsHighImpactBlockReason = "";
   
   // v1.11: pull pad windows now so the loop can populate the cache without
   // a second CalendarValueHistory call later from BlockNewEntries...
   long padBefore = (long)InpHighImpactNewsBeforeMin * 60;
   long padAfter  = (long)InpHighImpactNewsAfterMin  * 60;
   
   datetime fromTime = TimeCurrent() - (datetime)MathMax((long)(30 * 60),  padAfter  + 300);
   datetime toTime   = TimeCurrent() + (datetime)MathMax((long)(2 * 3600), padBefore + 3600);
   
   MqlCalendarValue values[];
   int count = CalendarValueHistory(values, fromTime, toTime, NULL, NULL);
   if(count <= 0)
   {
      g_News.hasUpcoming = false;
      g_News.warningMessage = "No calendar data";
      return;
   }
   
   for(int i = 0; i < count; i++)
   {
      MqlCalendarEvent ev;
      MqlCalendarCountry country;
      if(!CalendarEventById(values[i].event_id, ev)) continue;
      if(!CalendarCountryById(ev.country_id, country)) continue;
      
      string curr = country.currency;
      bool isRelevant = (curr == "USD" || curr == "EUR" || curr == "CNY" || curr == "CHF");
      string lname = ev.name;
      StringToLower(lname);
      if(StringFind(lname, "gold") >= 0 || StringFind(lname, "inflation") >= 0 ||
         StringFind(lname, "cpi") >= 0 || StringFind(lname, "interest rate") >= 0 ||
         StringFind(lname, "fed") >= 0 || StringFind(lname, "fomc") >= 0 ||
         StringFind(lname, "nonfarm") >= 0 || StringFind(lname, "employment") >= 0 ||
         StringFind(lname, "gdp") >= 0 || StringFind(lname, "pmi") >= 0)
         isRelevant = true;
      if(!isRelevant) continue;
      
      EVENT_IMPACT impact = IMPACT_LOW;
      if(ev.importance == CALENDAR_IMPORTANCE_HIGH)         impact = IMPACT_HIGH;
      else if(ev.importance == CALENDAR_IMPORTANCE_MODERATE) impact = IMPACT_MEDIUM;
      
      // Track most recent past HIGH event for news-continuation strategy
      if(impact == IMPACT_HIGH && values[i].time <= TimeCurrent())
      {
         if(values[i].time > g_LastHighImpactEventTime)
            g_LastHighImpactEventTime = values[i].time;
      }
      
      // v1.11: populate cached news-block flag using same window logic the
      // legacy BlockNewEntriesDueToHighImpactNews used to compute on every call.
      if(impact == IMPACT_HIGH)
      {
         datetime et = values[i].time;
         long deltaSec;
         if(et >= TimeCurrent())
         {
            deltaSec = (long)(et - TimeCurrent());
            if(deltaSec <= padBefore && !g_NewsHighImpactBlockActive)
            {
               g_NewsHighImpactBlockActive = true;
               g_NewsHighImpactBlockReason = "HIGH event in " +
                                              IntegerToString((int)(deltaSec/60)) + "m: " + ev.name;
            }
         }
         else
         {
            deltaSec = (long)(TimeCurrent() - et);
            if(deltaSec <= padAfter && !g_NewsHighImpactBlockActive)
            {
               g_NewsHighImpactBlockActive = true;
               g_NewsHighImpactBlockReason = "HIGH event " +
                                              IntegerToString((int)(deltaSec/60)) + "m ago: " + ev.name;
            }
         }
      }
      
      // Only track future events for hasUpcoming/blackout
      if(values[i].time < TimeCurrent()) continue;
      if(g_EventCount >= 10) break;
      
      int minutesUntil = (int)((values[i].time - TimeCurrent()) / 60);
      if(minutesUntil < 0) minutesUntil = 0;
      
      ArrayResize(g_UpcomingEvents, g_EventCount + 1);
      g_UpcomingEvents[g_EventCount].time = values[i].time;
      g_UpcomingEvents[g_EventCount].name = ev.name;
      g_UpcomingEvents[g_EventCount].currency = curr;
      g_UpcomingEvents[g_EventCount].impact = impact;
      g_UpcomingEvents[g_EventCount].minutesUntil = minutesUntil;
      g_UpcomingEvents[g_EventCount].isGoldRelated = (curr == "USD");
      g_EventCount++;
      
      if(impact == IMPACT_HIGH)
         g_News.highImpactCount++;
   }
   
   if(g_EventCount > 0)
   {
      g_News.hasUpcoming = true;
      int nearestIdx = 0;
      int nearestMin = g_UpcomingEvents[0].minutesUntil;
      for(int i = 1; i < g_EventCount; i++)
      {
         if(g_UpcomingEvents[i].minutesUntil < nearestMin)
         {
            nearestMin = g_UpcomingEvents[i].minutesUntil;
            nearestIdx = i;
         }
      }
      g_News.minutesToNext = nearestMin;
      g_News.nextEventName = g_UpcomingEvents[nearestIdx].name;
      g_News.nextEventImpact = g_UpcomingEvents[nearestIdx].impact;
      
      for(int i = 0; i < g_EventCount; i++)
      {
         if(g_UpcomingEvents[i].impact == IMPACT_HIGH && g_UpcomingEvents[i].minutesUntil <= 5)
         {
            g_News.inBlackout = true;
            g_News.tradingRestricted = true;
            g_News.warningMessage = "HIGH-IMPACT EVENT IN " +
               IntegerToString(g_UpcomingEvents[i].minutesUntil) + " MIN: " + g_UpcomingEvents[i].name;
            break;
         }
      }
      
      if(!g_News.inBlackout && nearestMin <= 15)
      {
         if(g_News.nextEventImpact == IMPACT_HIGH)
         {
            g_News.warningMessage = "HIGH-IMPACT in " + IntegerToString(nearestMin) + "min: " + g_News.nextEventName;
            g_News.tradingRestricted = true;
         }
         else if(g_News.nextEventImpact == IMPACT_MEDIUM)
         {
            g_News.warningMessage = "Event in " + IntegerToString(nearestMin) + "min: " + g_News.nextEventName;
         }
      }
   }
   else
   {
      g_News.hasUpcoming = false;
      g_News.warningMessage = "Clear - no major events";
   }
}

double SpreadRoundTripUsdApprox()
{
   if(g_TickValue <= 0.0 || g_TickSize <= 0.0 || InpLotSize <= 0.0) return 0.0;
   double spreadPrice = g_Spread * g_Point;
   double ticks = spreadPrice / g_TickSize;
   double oneWay = ticks * g_TickValue * InpLotSize;
   return 2.0 * oneWay;
}

bool BlockNewEntriesDueToHighImpactNews()
{
   if(!InpBlockHighImpactNewEntries) return false;
   // v1.11: pure cache read — CheckEconomicCalendar populates the flag once
   // per minute. Pre-v1.11 this issued a CalendarValueHistory call per gate
   // check, costing up to 5 API calls per ranker cycle (1×top + 1×#2 fallback
   // + 3×StoreTopForPanel). On a live tick stream that adds up fast.
   return g_NewsHighImpactBlockActive;
}

// v1.11: legacy implementation kept commented out for reference / future revert.
bool BlockNewEntriesDueToHighImpactNews_LegacyDirect()
{
   if(!InpBlockHighImpactNewEntries) return false;
   
   long padBefore = (long)InpHighImpactNewsBeforeMin * 60;
   long padAfter  = (long)InpHighImpactNewsAfterMin  * 60;
   datetime fromT = TimeCurrent() - (datetime)(padAfter + 300);
   datetime toT   = TimeCurrent() + (datetime)(padBefore + 3600);
   
   MqlCalendarValue values[];
   int count = CalendarValueHistory(values, fromT, toT, NULL, NULL);
   if(count <= 0) return false;
   
   for(int i = 0; i < count; i++)
   {
      MqlCalendarEvent ev;
      MqlCalendarCountry country;
      if(!CalendarEventById(values[i].event_id, ev)) continue;
      if(!CalendarCountryById(ev.country_id, country)) continue;
      if(ev.importance != CALENDAR_IMPORTANCE_HIGH) continue;
      
      string curr = country.currency;
      bool isRelevant = (curr == "USD" || curr == "EUR" || curr == "CNY" || curr == "CHF");
      string lname = ev.name;
      StringToLower(lname);
      if(StringFind(lname, "gold") >= 0 || StringFind(lname, "inflation") >= 0 ||
         StringFind(lname, "cpi") >= 0 || StringFind(lname, "interest rate") >= 0 ||
         StringFind(lname, "fed") >= 0 || StringFind(lname, "fomc") >= 0 ||
         StringFind(lname, "nonfarm") >= 0 || StringFind(lname, "employment") >= 0 ||
         StringFind(lname, "gdp") >= 0 || StringFind(lname, "pmi") >= 0)
         isRelevant = true;
      if(!isRelevant) continue;
      
      datetime et = values[i].time;
      if(et > TimeCurrent())
      {
         if((long)(et - TimeCurrent()) <= padBefore) return true;
      }
      else
      {
         if((long)(TimeCurrent() - et) <= padAfter) return true;
      }
   }
   return false;
}

//+------------------------------------------------------------------+
//| MT5 extended intel (spread stats, D1/W1 levels, DOM, ADX, BB)    |
//+------------------------------------------------------------------+
double SpreadPercentile(const double frac)
{
   if(g_SpreadSampleCount < 3 || g_Point <= 0) return g_Spread;
   double tmp[];
   int n = g_SpreadSampleCount;
   ArrayResize(tmp, n);
   int start = (g_SpreadWriteIdx - n + SPREAD_RING_SIZE) % SPREAD_RING_SIZE;
   for(int i = 0; i < n; i++)
      tmp[i] = g_SpreadSamples[(start + i) % SPREAD_RING_SIZE];
   ArraySort(tmp);
   int idx = (int)MathFloor((n - 1) * frac);
   if(idx < 0) idx = 0;
   if(idx >= n) idx = n - 1;
   return tmp[idx];
}

void UpdateTickActivityRates()
{
   g_TicksLast10s = 0;
   g_TicksLast60s = 0;
   if(g_TickCount < 2) return;
   int endIdx = (g_TickHead - 1 + MAX_TICKS) % MAX_TICKS;
   ulong nowMsc = g_TickBuffer[endIdx].time_msc;
   for(int k = 0; k < g_TickCount; k++)
   {
      int idx = (endIdx - k + MAX_TICKS) % MAX_TICKS;
      ulong dt = nowMsc - g_TickBuffer[idx].time_msc;
      if(dt <= 10000) g_TicksLast10s++;
      if(dt <= 60000) g_TicksLast60s++;
      else break;
   }
}

void UpdateD1W1Levels()
{
   double h[], l[];
   ArraySetAsSeries(h, true);
   ArraySetAsSeries(l, true);
   g_PDH = 0; g_PDL = 0; g_PWH = 0; g_PWL = 0;
   if(CopyHigh(_Symbol, PERIOD_D1, 1, 1, h) == 1) g_PDH = h[0];
   if(CopyLow(_Symbol, PERIOD_D1, 1, 1, l) == 1) g_PDL = l[0];
   if(CopyHigh(_Symbol, PERIOD_W1, 1, 1, h) == 1) g_PWH = h[0];
   if(CopyLow(_Symbol, PERIOD_W1, 1, 1, l) == 1) g_PWL = l[0];
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   if(g_Point > 0)
   {
      g_DistPDH = (g_PDH > 0) ? (g_PDH - bid) / g_Point : 0;
      g_DistPDL = (g_PDL > 0) ? (bid - g_PDL) / g_Point : 0;
      g_DistPWH = (g_PWH > 0) ? (g_PWH - bid) / g_Point : 0;
      g_DistPWL = (g_PWL > 0) ? (bid - g_PWL) / g_Point : 0;
   }
}

string StrSymbolTradeMode(const long m)
{
   if(m == SYMBOL_TRADE_MODE_DISABLED)  return "DISABLED";
   if(m == SYMBOL_TRADE_MODE_FULL)      return "FULL";
   if(m == SYMBOL_TRADE_MODE_LONGONLY)  return "LONG_ONLY";
   if(m == SYMBOL_TRADE_MODE_SHORTONLY) return "SHORT_ONLY";
   if(m == SYMBOL_TRADE_MODE_CLOSEONLY) return "CLOSE_ONLY";
   return IntegerToString((int)m);
}

string StrFillingMode(const long fm)
{
   if((fm & SYMBOL_FILLING_FOK) == SYMBOL_FILLING_FOK) return "FOK";
   if((fm & SYMBOL_FILLING_IOC) == SYMBOL_FILLING_IOC) return "IOC";
   return "RETURN";
}

void UpdateBrokerSessionWindows()
{
   g_BrokerSessionSummary = "";
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   ENUM_DAY_OF_WEEK dow = (ENUM_DAY_OF_WEEK)dt.day_of_week;
   for(uint si = 0; si < 8; si++)
   {
      datetime from = 0, to = 0;
      if(!SymbolInfoSessionTrade(_Symbol, dow, si, from, to)) break;
      if(from == 0 && to == 0) continue;
      if(StringLen(g_BrokerSessionSummary) > 0) g_BrokerSessionSummary += " | ";
      g_BrokerSessionSummary += TimeToString(from, TIME_MINUTES) + "-" + TimeToString(to, TIME_MINUTES);
   }
   if(StringLen(g_BrokerSessionSummary) == 0)
      g_BrokerSessionSummary = "No session data";
}

void UpdateSymbolConstraints()
{
   g_SymStopsLevel  = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL);
   g_SymFreezeLevel = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_FREEZE_LEVEL);
   g_SymTradeModeStr = StrSymbolTradeMode(SymbolInfoInteger(_Symbol, SYMBOL_TRADE_MODE));
   g_SymFillingStr   = StrFillingMode(SymbolInfoInteger(_Symbol, SYMBOL_FILLING_MODE));
}

void UpdateM1VolumeStats()
{
   g_VolumeDataOK = false;
   g_M1VolLast = 0;
   g_M1VolSum20 = 0;
   long vol[];
   ArraySetAsSeries(vol, true);
   if(CopyTickVolume(_Symbol, PERIOD_M1, 0, 20, vol) != 20) return;
   g_VolumeDataOK = true;
   g_M1VolLast = vol[0];
   for(int i = 0; i < 20; i++) g_M1VolSum20 += vol[i];
}

void UpdateDomImbalance()
{
   g_DomBidVol = 0;
   g_DomAskVol = 0;
   g_DomImbalancePct = 0;
   if(!g_MarketBookSubscribed) return;
   MqlBookInfo book[];
   if(!MarketBookGet(_Symbol, book) || ArraySize(book) == 0) return;
   for(int i = 0; i < ArraySize(book); i++)
   {
      double v = book[i].volume_real;
      if(v <= 0) v = (double)book[i].volume;
      if(book[i].type == BOOK_TYPE_BUY)       g_DomBidVol += v;
      else if(book[i].type == BOOK_TYPE_SELL) g_DomAskVol += v;
   }
   double tot = g_DomBidVol + g_DomAskVol;
   if(tot > 0) g_DomImbalancePct = (g_DomBidVol - g_DomAskVol) / tot * 100.0;
}

void UpdateRegimeIndicators()
{
   g_M5ADX = 0;
   g_M5BBWidthPct = 0;
   double adx[];
   ArraySetAsSeries(adx, true);
   if(CopyBuffer(g_HandleM5ADX, 0, 0, 1, adx) >= 1) g_M5ADX = adx[0];
   double bu[], bl[], bm[];
   ArraySetAsSeries(bu, true);
   ArraySetAsSeries(bl, true);
   ArraySetAsSeries(bm, true);
   if(CopyBuffer(g_HandleM5Bands, 0, 0, 1, bu) >= 1 &&
      CopyBuffer(g_HandleM5Bands, 1, 0, 1, bl) >= 1 &&
      CopyBuffer(g_HandleM5Bands, 2, 0, 1, bm) >= 1 && bm[0] != 0)
   {
      g_M5BBWidthPct = (bu[0] - bl[0]) / bm[0] * 100.0;
   }
}

void UpdateAccountSnapshot()
{
   g_FreeMargin    = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
   g_MarginLevel   = AccountInfoDouble(ACCOUNT_MARGIN_LEVEL);
   g_AccountEquity = AccountInfoDouble(ACCOUNT_EQUITY);
}

void UpdateSpreadDistribution()
{
   if(g_SpreadSampleCount < 5)
   {
      g_SpreadMedian    = g_Spread;
      g_SpreadP90       = g_Spread;
      g_SpreadMaxRecent = g_Spread;
      g_SpreadSpike     = false;
      return;
   }
   g_SpreadMedian = SpreadPercentile(0.50);
   g_SpreadP90    = SpreadPercentile(0.90);
   double tmp[];
   int n = g_SpreadSampleCount;
   ArrayResize(tmp, n);
   int start = (g_SpreadWriteIdx - n + SPREAD_RING_SIZE) % SPREAD_RING_SIZE;
   for(int i = 0; i < n; i++) tmp[i] = g_SpreadSamples[(start + i) % SPREAD_RING_SIZE];
   ArraySort(tmp);
   g_SpreadMaxRecent = tmp[n - 1];
   g_SpreadSpike = (g_Spread > g_SpreadP90 * 1.25 && g_SpreadP90 > 0);
}

void UpdateMT5ExtendedIntel()
{
   UpdateSpreadDistribution();
   UpdateTickActivityRates();
   UpdateD1W1Levels();
   UpdateBrokerSessionWindows();
   UpdateSymbolConstraints();
   UpdateM1VolumeStats();
   UpdateDomImbalance();
   UpdateRegimeIndicators();
   UpdateAccountSnapshot();
}

//+------------------------------------------------------------------+
//| Market data (RSI/ATR/WPR + spread)                               |
//+------------------------------------------------------------------+
void UpdateMarketData()
{
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   g_Spread = (ask - bid) / g_Point;
   
   double rsi[], atr[], wpr[];
   ArraySetAsSeries(rsi, true);
   ArraySetAsSeries(atr, true);
   ArraySetAsSeries(wpr, true);
   
   if(CopyBuffer(g_HandleM1RSI, 0, 0, 1, rsi) >= 1) g_M1RSI = rsi[0];
   if(CopyBuffer(g_HandleM1RSI, 0, 1, 1, rsi) >= 1) g_M1RsiBar1 = rsi[0]; else g_M1RsiBar1 = g_M1RSI;
   if(CopyBuffer(g_HandleM1ATR, 0, 0, 1, atr) >= 1) g_M1ATR = atr[0];
   if(CopyBuffer(g_HandleWPR,   0, 0, 1, wpr) >= 1) g_WPR = wpr[0];
}

double RsiForEntryModel()
{
   if(!InpUseClosedM1RsiForEntry) return g_M1RSI;
   return g_M1RsiBar1;
}

ENUM_TIMEFRAMES EffectiveEntryBarTF()
{
   if(InpEntryNewBarTimeframe == PERIOD_CURRENT) return (ENUM_TIMEFRAMES)Period();
   return InpEntryNewBarTimeframe;
}

string EntryBarTfLabel()
{
   ENUM_TIMEFRAMES t = EffectiveEntryBarTF();
   if(t == PERIOD_M1)  return "M1";
   if(t == PERIOD_M5)  return "M5";
   if(t == PERIOD_M15) return "M15";
   if(t == PERIOD_M30) return "M30";
   if(t == PERIOD_H1)  return "H1";
   return IntegerToString((int)t);
}

//+------------------------------------------------------------------+
//| HIGHER-TF BIAS + M5 ANALYSIS                                     |
//+------------------------------------------------------------------+
void AnalyzeM15Bias()
{
   double ema[];
   ArraySetAsSeries(ema, true);
   if(CopyBuffer(g_HandleM15EMA, 0, 0, 1, ema) < 1) { g_M15Bias = 0; return; }
   double price = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double buffer = ema[0] * 0.001;
   if(price > ema[0] + buffer)      g_M15Bias = 1;
   else if(price < ema[0] - buffer) g_M15Bias = -1;
   else                             g_M15Bias = 0;
}

void AnalyzeH1Bias()
{
   double ema[];
   ArraySetAsSeries(ema, true);
   if(g_HandleH1EMA == INVALID_HANDLE || CopyBuffer(g_HandleH1EMA, 0, 0, 1, ema) < 1)
   { g_H1Bias = 0; return; }
   double price = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double buffer = ema[0] * 0.001;
   if(price > ema[0] + buffer)      g_H1Bias = 1;
   else if(price < ema[0] - buffer) g_H1Bias = -1;
   else                             g_H1Bias = 0;
}

void AnalyzeM5Trend()
{
   double open[], close[], high[], low[];
   ArraySetAsSeries(open, true);
   ArraySetAsSeries(close, true);
   ArraySetAsSeries(high, true);
   ArraySetAsSeries(low, true);
   
   if(CopyOpen(_Symbol, PERIOD_M5, 1, InpM5BarsForTrend, open) != InpM5BarsForTrend ||
      CopyClose(_Symbol, PERIOD_M5, 1, InpM5BarsForTrend, close) != InpM5BarsForTrend ||
      CopyHigh(_Symbol, PERIOD_M5, 1, InpM5BarsForTrend, high) != InpM5BarsForTrend ||
      CopyLow(_Symbol, PERIOD_M5, 1, InpM5BarsForTrend, low) != InpM5BarsForTrend)
   {
      g_M5Trend = 0; g_M5TrendPct = 0; g_M5Movement = 0;
      return;
   }
   
   int bulls = 0, bears = 0;
   double avgCandleSize = 0;
   g_M5BiggestCandle = 0;
   for(int i = 0; i < InpM5BarsForTrend; i++)
   {
      if(close[i] > open[i]) bulls++;
      else if(close[i] < open[i]) bears++;
      double cs = (high[i] - low[i]) / g_Point;
      avgCandleSize += cs;
      if(cs > g_M5BiggestCandle) g_M5BiggestCandle = cs;
   }
   avgCandleSize /= InpM5BarsForTrend;
   double lastCandleSize = (high[0] - low[0]) / g_Point;
   g_M5MomentumBurst = (lastCandleSize >= avgCandleSize * 2.0);
   g_M5Movement = (close[0] - open[InpM5BarsForTrend-1]) / g_Point;
   
   double total = (double)InpM5BarsForTrend;
   double bullPct = (double)bulls / total;
   double bearPct = (double)bears / total;
   g_M5TrendPct = MathMax(bullPct, bearPct) * 100;
   
   double ema[];
   ArraySetAsSeries(ema, true);
   if(CopyBuffer(g_HandleM5EMA, 0, 0, 1, ema) >= 1)
   {
      g_M5EMAValue = ema[0];
      double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
      double buffer = g_CurrentATR * 0.1;
      if(bid > g_M5EMAValue + buffer)      g_M5EMABias = 1;
      else if(bid < g_M5EMAValue - buffer) g_M5EMABias = -1;
      else                                 g_M5EMABias = 0;
   }
   
   g_M5LastClosedBull = (close[0] > open[0]);
   g_M5LastClosedBear = (close[0] < open[0]);
   
   if(bullPct >= InpTrendStrength && g_M5Movement >= InpMinTrendPoints)
      g_M5Trend = 1;
   else if(bearPct >= InpTrendStrength && g_M5Movement <= -InpMinTrendPoints)
      g_M5Trend = -1;
   else
      g_M5Trend = 0;
}

void AnalyzeM5Momentum()
{
   double open[], close[];
   ArraySetAsSeries(open, true);
   ArraySetAsSeries(close, true);
   if(CopyOpen(_Symbol, PERIOD_M5, 1, InpMomentumBars, open) != InpMomentumBars ||
      CopyClose(_Symbol, PERIOD_M5, 1, InpMomentumBars, close) != InpMomentumBars)
   {
      g_M5MomentumOK = false;
      return;
   }
   int confirming = 0;
   for(int i = 0; i < InpMomentumBars; i++)
   {
      if(g_M5Trend == 1 && close[i] > open[i]) confirming++;
      else if(g_M5Trend == -1 && close[i] < open[i]) confirming++;
   }
   g_M5MomentumOK = (confirming >= InpMomentumBars - 1);
}

void AnalyzeM1Setup()
{
   g_M1Setup = 0;
   if(g_M5Trend == 0) return;
   double open[], close[];
   ArraySetAsSeries(open, true);
   ArraySetAsSeries(close, true);
   if(CopyOpen(_Symbol, PERIOD_M1, 1, 5, open) != 5 ||
      CopyClose(_Symbol, PERIOD_M1, 1, 5, close) != 5)
      return;
   int pullback = 0;
   for(int i = 0; i < 5; i++)
   {
      if(g_M5Trend == 1 && close[i] < open[i]) pullback++;
      else if(g_M5Trend == -1 && close[i] > open[i]) pullback++;
      else break;
   }
   bool resumption = (g_M5Trend == 1 && close[0] > open[0]) || (g_M5Trend == -1 && close[0] < open[0]);
   if(pullback >= 2 && resumption) g_M1Setup = 2;
   else if(pullback >= 1) g_M1Setup = 1;
}

void CheckSession()
{
   // reset session-window globals every call
   g_InSessionOpenWindow_London = false;
   g_InSessionOpenWindow_NY     = false;
   g_MinutesSinceLondonOpen     = -1;
   g_MinutesSinceNYOpen         = -1;
   g_MinutesUntilSessionClose   = -1;
   
   if(!InpUseSessionFilter) { g_SessionOK = true; g_Session = "ANY"; return; }
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   int hour = dt.hour;
   int minute = dt.min;
   g_SessionOK = false;
   g_Session = "CLOSED";
   if(hour >= InpLondonStart && hour < InpLondonEnd)
   {
      g_Session = "LONDON";
      g_SessionOK = true;
      g_MinutesUntilSessionClose = (InpLondonEnd - hour) * 60 - minute;
   }
   if(hour >= InpNYStart && hour < InpNYEnd)
   {
      if(g_Session == "LONDON") g_Session = "OVERLAP";
      else g_Session = "NY";
      g_SessionOK = true;
      int nyClose = (InpNYEnd - hour) * 60 - minute;
      if(g_MinutesUntilSessionClose < 0 || nyClose > g_MinutesUntilSessionClose)
         g_MinutesUntilSessionClose = nyClose;
   }
   
   // Session-open window: first N minutes after each open
   int winMin = InpSessionOpenWindowMin;
   if(winMin > 0)
   {
      int minsSinceLondon = (hour - InpLondonStart) * 60 + minute;
      if(minsSinceLondon >= 0 && minsSinceLondon < winMin)
      {
         g_InSessionOpenWindow_London = true;
         g_MinutesSinceLondonOpen = minsSinceLondon;
      }
      int minsSinceNY = (hour - InpNYStart) * 60 + minute;
      if(minsSinceNY >= 0 && minsSinceNY < winMin)
      {
         g_InSessionOpenWindow_NY = true;
         g_MinutesSinceNYOpen = minsSinceNY;
      }
   }
   
   // v1.11: weekend-gap protection. Gold can gap $10-$20 on Sunday open and a
   // resting SL won't protect against the gapped fill price. After Friday
   // InpFridayCloseHour we stop new entries (existing positions still managed
   // by AutoProtect, which usually flattens by NY close).
   if(InpAvoidWeekendGap && dt.day_of_week == 5 && hour >= InpFridayCloseHour)
   {
      g_SessionOK = false;
      g_Session   = "WEEKEND_CLOSE";
   }
}

void AnalyzeRecentMoves()
{
   double o5[], c5[];
   ArraySetAsSeries(o5, true);
   ArraySetAsSeries(c5, true);
   if(CopyOpen(_Symbol, PERIOD_M5, 1, 3, o5) == 3 &&
      CopyClose(_Symbol, PERIOD_M5, 1, 3, c5) == 3)
      g_RecentM5Move = (c5[0] - o5[2]) / g_Point;
   double o1[], c1[];
   ArraySetAsSeries(o1, true);
   ArraySetAsSeries(c1, true);
   if(CopyOpen(_Symbol, PERIOD_M1, 1, 5, o1) == 5 &&
      CopyClose(_Symbol, PERIOD_M1, 1, 5, c1) == 5)
      g_RecentM1Move = (c1[0] - o1[4]) / g_Point;
}

void RunAnalysis()
{
   AnalyzeM15Bias();
   AnalyzeH1Bias();
   AnalyzeM5Trend();
   AnalyzeM5Momentum();
   AnalyzeM1Setup();
   CheckSession();
   AnalyzeRecentMoves();
}

//+------------------------------------------------------------------+
//| VWAP / ORB                                                       |
//+------------------------------------------------------------------+
datetime GetVWAPORBAnchorTime()
{
   MqlDateTime t;
   TimeToStruct(TimeCurrent(), t);
   if(InpVWAPAnchorMode == 0)
   {
      t.hour = 0; t.min = 0; t.sec = 0;
      return StructToTime(t);
   }
   t.hour = InpLondonStart; t.min = 0; t.sec = 0;
   datetime tLo = StructToTime(t);
   if(TimeCurrent() < tLo) tLo -= 86400;
   return tLo;
}

void UpdateVWAPAndORB()
{
   g_VWAPValid = false;
   g_ORBValid = false;
   g_SessionVWAP = 0;
   g_ORBHigh = 0;
   g_ORBLow = 0;
   
   datetime anchor = GetVWAPORBAnchorTime();
   int s = iBarShift(_Symbol, PERIOD_M1, anchor, false);
   if(s < 0) return;
   
   double sumPV = 0, sumV = 0;
   for(int k = 0; k <= s; k++)
   {
      double h = iHigh(_Symbol, PERIOD_M1, k);
      double l = iLow(_Symbol, PERIOD_M1, k);
      double c = iClose(_Symbol, PERIOD_M1, k);
      long   v = iTickVolume(_Symbol, PERIOD_M1, k);
      if(v < 0) v = 0;
      double tp = (h + l + c) / 3.0;
      sumPV += tp * (double)v;
      sumV  += (double)v;
   }
   if(sumV > 0)
   {
      g_SessionVWAP = sumPV / sumV;
      g_VWAPValid = true;
   }
   
   int nOrb = InpORBFirstMinutes;
   if(nOrb < 1) nOrb = 15;
   double hi = -1.0e100, lo = 1.0e100;
   for(int k = 0; k < nOrb; k++)
   {
      int sh = s - k;
      if(sh < 0) break;
      double h = iHigh(_Symbol, PERIOD_M1, sh);
      double l = iLow(_Symbol, PERIOD_M1, sh);
      if(h > hi) hi = h;
      if(l < lo) lo = l;
   }
   if(hi > lo)
   {
      g_ORBHigh = hi;
      g_ORBLow  = lo;
      g_ORBValid = true;
   }
}

//+------------------------------------------------------------------+
//| BAR HELPERS                                                      |
//+------------------------------------------------------------------+
bool Bar1BullishTF(const ENUM_TIMEFRAMES tf)
{
   if(Bars(_Symbol, tf) < 3) return true;
   double o = iOpen(_Symbol, tf, 1);
   double c = iClose(_Symbol, tf, 1);
   return c > o;
}

bool Bar1BearishTF(const ENUM_TIMEFRAMES tf)
{
   if(Bars(_Symbol, tf) < 3) return true;
   double o = iOpen(_Symbol, tf, 1);
   double c = iClose(_Symbol, tf, 1);
   return c < o;
}

bool M1LastNAllBearish()
{
   int n = InpM1AlignBars;
   if(n < 1) return false;
   if(n > 20) n = 20;
   for(int i = 1; i <= n; i++)
   {
      double o = iOpen(_Symbol, PERIOD_M1, i);
      double c = iClose(_Symbol, PERIOD_M1, i);
      if(c >= o) return false;
   }
   return true;
}

bool M1LastNAllBullish()
{
   int n = InpM1AlignBars;
   if(n < 1) return false;
   if(n > 20) n = 20;
   for(int i = 1; i <= n; i++)
   {
      double o = iOpen(_Symbol, PERIOD_M1, i);
      double c = iClose(_Symbol, PERIOD_M1, i);
      if(c <= o) return false;
   }
   return true;
}

int GetEffectiveM5Direction()
{
   if(g_M5Trend != 0) return g_M5Trend;
   if(!g_M5MomentumBurst) return 0;
   if(g_M5EMABias != 0) return g_M5EMABias;
   if(g_M5LastClosedBull && !g_M5LastClosedBear) return 1;
   if(g_M5LastClosedBear && !g_M5LastClosedBull) return -1;
   return 0;
}

string RegimeLabelForAI()
{
   if(g_News.inBlackout)            return "NEWS_BLACKOUT";
   if(g_VolRegime == VOL_EXPLOSIVE) return "VOL_EXPLOSIVE";
   if(g_M5Trend == 0)
   {
      if(g_M5MomentumBurst) return "IMPULSE_BURST";
      return "RANGE_CHOP";
   }
   if(g_M5ADX >= 25.0)              return "TREND_ADX";
   if(g_VolRegime == VOL_QUIET)     return "RANGE_QUIET";
   return "TREND_SOFT";
}

//+------------------------------------------------------------------+
//| CSV JOURNAL                                                      |
//+------------------------------------------------------------------+
string ScalperEscapeField(const string s)
{
   string x = s;
   StringReplace(x, ";", ",");
   StringReplace(x, "\n", " ");
   StringReplace(x, "\r", "");
   if(StringLen(x) > 320) x = StringSubstr(x, 0, 320);
   return x;
}

void ScalperLogInit()
{
   if(!InpEnableCsvLog) return;
   // Writes to <Terminal Data>\MQL5\Files\<InpCsvLogFile>
   // MT5 auto-creates intermediate directories on first FileOpen with FILE_WRITE.
   if(FileIsExist(InpCsvLogFile)) return;
   int h = FileOpen(InpCsvLogFile, FILE_WRITE|FILE_TXT|FILE_ANSI);
   if(h == INVALID_HANDLE)
   {
      Print("CSV log: cannot create ", InpCsvLogFile, " err ", GetLastError());
      return;
   }
   FileWriteString(h, "utc_ts;event;symbol;regime;score;spread_pts;primary;secondary;detail\n");
   FileClose(h);
   Print("CSV journal: MQL5\\Files\\", InpCsvLogFile);
}

void ScalperLog(const string evt, const string primary, const string secondary, const string detail)
{
   if(!InpEnableCsvLog) return;
   string ts = TimeToString(TimeGMT(), TIME_DATE|TIME_SECONDS);
   string line = ts + ";" + evt + ";" + _Symbol + ";" + RegimeLabelForAI() + ";" +
      DoubleToString(g_LogSnapshotScore, 2) + ";" + DoubleToString(g_Spread, 1) + ";" +
      ScalperEscapeField(primary) + ";" + ScalperEscapeField(secondary) + ";" + ScalperEscapeField(detail) + "\n";
   int h = FileOpen(InpCsvLogFile, FILE_READ|FILE_WRITE|FILE_TXT|FILE_ANSI|FILE_SHARE_READ|FILE_SHARE_WRITE);
   if(h == INVALID_HANDLE) return;
   FileSeek(h, 0, SEEK_END);
   FileWriteString(h, line);
   FileClose(h);
}

//+------------------------------------------------------------------+
//| SETUP NAME / KIND HELPERS                                        |
//+------------------------------------------------------------------+
string SetupKindName(SETUP_KIND k)
{
   switch(k)
   {
      case SETUP_SPIKE:         return "SPIKE";
      case SETUP_FLAG:          return "FLAG";
      case SETUP_ORB:           return "ORB";
      case SETUP_VWAP_RECLAIM:  return "VWAP_RECLAIM";
      case SETUP_VWAP_REJECT:   return "VWAP_REJECT";
      case SETUP_ABCD:          return "ABCD";
      case SETUP_LEVEL_BOUNCE:  return "LEVEL_BOUNCE";
      case SETUP_LEVEL_BREAK:   return "LEVEL_BREAK";
      case SETUP_EMA_PULLBACK:  return "EMA_PULLBACK";
      case SETUP_NEWS_CONT:     return "NEWS_CONT";
      case SETUP_FADE:          return "FADE";
      default:                  return "NONE";
   }
}

string SetupKindTag(SETUP_KIND k)
{
   switch(k)
   {
      case SETUP_SPIKE:         return "SPK";
      case SETUP_FLAG:          return "FLG";
      case SETUP_ORB:           return "ORB";
      case SETUP_VWAP_RECLAIM:  return "VWR";
      case SETUP_VWAP_REJECT:   return "VWJ";
      case SETUP_ABCD:          return "ABC";
      case SETUP_LEVEL_BOUNCE:  return "LBN";
      case SETUP_LEVEL_BREAK:   return "LBR";
      case SETUP_EMA_PULLBACK:  return "EMA";
      case SETUP_NEWS_CONT:     return "NWC";
      case SETUP_FADE:          return "FAD";
      default:                  return "NON";
   }
}

void ProposalReset(SetupProposal &p, SETUP_KIND k)
{
   p.valid    = false;
   p.kind     = k;
   p.name     = SetupKindName(k);
   p.tag      = SetupKindTag(k);
   p.isBuy    = false;
   p.score    = 0;
   p.rawScore = 0;
   p.ctxScore = 0;
   p.entryRef = 0;
   p.stopLoss = 0;
   p.t1 = 0; p.t2 = 0; p.t3 = 0;
   p.riskUsd = 0;
   p.rrToT1  = 0;
   p.factors = "";
}

double DollarsPerPriceUnit()
{
   if(g_TickValue <= 0 || InpLotSize <= 0 || g_TickSize <= 0) return 0;
   return (g_TickValue * InpLotSize) / g_TickSize;
}

// Compute riskUsd, rrToT1, and ladder T2/T3 from entryRef/stopLoss/t1
void ProposalFinalizeRR(SetupProposal &p)
{
   if(p.entryRef <= 0 || p.stopLoss <= 0) { p.valid = false; return; }
   double dpp = DollarsPerPriceUnit();
   if(dpp <= 0) { p.valid = false; return; }
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   if(slDist <= 0) { p.valid = false; return; }
   p.riskUsd = slDist * dpp;
   if(p.t1 > 0)
   {
      double tpDist = MathAbs(p.t1 - p.entryRef);
      p.rrToT1 = (slDist > 0) ? tpDist / slDist : 0;
   }
   // Auto-fill T2/T3 from dynamic targets if missing
   if(g_Targets.minProfit > 0 && (p.t2 <= 0 || p.t3 <= 0))
   {
      double t2DistPrice = (g_Targets.goodProfit / dpp);
      double t3DistPrice = (g_Targets.greatProfit / dpp);
      if(p.isBuy)
      {
         if(p.t2 <= 0) p.t2 = NormalizeDouble(p.entryRef + t2DistPrice, g_Digits);
         if(p.t3 <= 0) p.t3 = NormalizeDouble(p.entryRef + t3DistPrice, g_Digits);
      }
      else
      {
         if(p.t2 <= 0) p.t2 = NormalizeDouble(p.entryRef - t2DistPrice, g_Digits);
         if(p.t3 <= 0) p.t3 = NormalizeDouble(p.entryRef - t3DistPrice, g_Digits);
      }
   }
}

//+------------------------------------------------------------------+
//| Risk cap (USD) currently in force for proposal pre-flight.       |
//| Mirrors the math used by PassesGlobalHardGates so pre-execution  |
//| clamping and the gate agree on the same number.                  |
//+------------------------------------------------------------------+
double CurrentRiskCapUsd()
{
   double cap = InpMaxRiskDollars;
   if(g_Targets.stopLossDollars > 0)
      cap = MathMin(cap, g_Targets.stopLossDollars);
   if(InpCapOrderRiskToMinProfit && g_Targets.minProfit > 0)
   {
      double cap2 = g_Targets.minProfit * InpMaxRiskVsMinTargetMult;
      if(cap2 > 0) cap = MathMin(cap, cap2);
   }
   if(cap < 0.5) cap = 0.5;  // sanity floor
   return cap;
}

//+------------------------------------------------------------------+
//| ClampProposalSlToRiskCap                                         |
//|   If a strategy placed its SL beyond what the dollar-risk cap    |
//|   allows AT InpLotSize, pull SL closer to entry to fit the cap   |
//|   and rescale T1 to preserve the original reward-to-risk.        |
//|   Marks the proposal invalid if the post-clamp SL would sit      |
//|   inside the broker's minimum stop level.                        |
//+------------------------------------------------------------------+
void ClampProposalSlToRiskCap(SetupProposal &p)
{
   if(!p.valid) return;
   if(p.entryRef <= 0 || p.stopLoss <= 0) return;
   double dpp = DollarsPerPriceUnit();
   if(dpp <= 0) return;
   double capUsd = CurrentRiskCapUsd();
   double maxSlDist = capUsd / dpp;        // max SL distance in price units, at InpLotSize
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   if(slDist <= maxSlDist) return;          // already within cap — nothing to do
   
   // Preserve original R:R when shrinking SL
   double origRR = (p.t1 > 0 ? MathAbs(p.t1 - p.entryRef) / slDist : 1.5);
   if(origRR < 1.0) origRR = 1.0;
   // v1.11: ceiling depends on setup kind. Tight-SL momentum setups (SPIKE,
   // NEWS_CONT, FADE) genuinely produce 5R-8R to T1 — capping them to 4R was
   // leaving real money on the table. Allow up to 8R for those; keep 4R for
   // pattern-based setups where 4R is already aspirational.
   double rrCeiling = (p.kind == SETUP_SPIKE || p.kind == SETUP_NEWS_CONT || p.kind == SETUP_FADE)
                      ? 8.0 : 4.0;
   if(origRR > rrCeiling) origRR = rrCeiling;
   
   double newSlDist = maxSlDist;
   double newSl = p.isBuy
      ? NormalizeDouble(p.entryRef - newSlDist, g_Digits)
      : NormalizeDouble(p.entryRef + newSlDist, g_Digits);
   
   // Respect broker minimum stop distance — if too tight, drop the proposal
   double minStop = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * g_Point;
   if(minStop <= 0) minStop = 10 * g_Point;
   if(MathAbs(p.entryRef - newSl) < minStop)
   {
      p.valid = false;
      p.factors += "[clamp invalid: SL would be inside broker minStop]\n";
      return;
   }
   
   double oldSl = p.stopLoss;
   p.stopLoss = newSl;
   double newT1Dist = newSlDist * origRR;
   p.t1 = p.isBuy
      ? NormalizeDouble(p.entryRef + newT1Dist, g_Digits)
      : NormalizeDouble(p.entryRef - newT1Dist, g_Digits);
   // Reset t2/t3 — let ProposalFinalizeRR re-derive from dynamic targets
   p.t2 = 0;
   p.t3 = 0;
   p.factors += "[SL clamped to risk cap: " + DoubleToString(oldSl, g_Digits) +
                " -> " + DoubleToString(newSl, g_Digits) +
                " (cap $" + DoubleToString(capUsd, 2) + ")]\n";
}

//+------------------------------------------------------------------+
//| AsymmetricMultiplier                                             |
//|   Score-band-based lot multiplier so weaker setups (60-74) trade |
//|   smaller and only A+ setups (>=85) get the full size.           |
//+------------------------------------------------------------------+
double AsymmetricMultiplier(double score)
{
   if(!InpAsymSize) return 1.0;
   if(score >= 85.0) return InpAsymMul_85_plus;
   if(score >= 75.0) return InpAsymMul_75_85;
   return InpAsymMul_60_75;
}

//+------------------------------------------------------------------+
//| SetupCategoryWeight                                              |
//|   Prior probability multiplier — rare/high-conviction setups get |
//|   a small boost, frequent/moderate setups a small penalty, so    |
//|   the ranker doesn't trivially default to the most-frequent kind.|
//+------------------------------------------------------------------+
double SetupCategoryWeight(SETUP_KIND k)
{
   if(k == SETUP_SPIKE || k == SETUP_NEWS_CONT) return 1.12;     // rare, high conviction
   if(k == SETUP_FLAG  || k == SETUP_EMA_PULLBACK || k == SETUP_ORB) return 0.95;  // frequent, moderate
   return 1.00;
}

//+------------------------------------------------------------------+
//| EffectiveRiskUsd                                                 |
//|   Risk this proposal would actually deliver at the planned lot   |
//|   (regime + asymmetric multipliers, BEFORE auto-shrink-to-cap).  |
//+------------------------------------------------------------------+
double EffectiveRiskUsd(const SetupProposal &p)
{
   if(p.entryRef <= 0 || p.stopLoss <= 0 || g_TickValue <= 0 || g_TickSize <= 0)
      return p.riskUsd;
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tradeLot = LotForRegime() * AsymmetricMultiplier(p.score);
   if(tradeLot <= 0) tradeLot = InpLotSize;
   return slDist * (g_TickValue * tradeLot / g_TickSize);
}

//+------------------------------------------------------------------+
//| RiskExceedsCapAtMinLot                                           |
//|   True iff even at the broker's minimum tradeable lot the dollar |
//|   risk would still exceed the cap. The gate uses this — anything |
//|   above min lot can be salvaged by ExecuteProposal's auto-shrink.|
//+------------------------------------------------------------------+
bool RiskExceedsCapAtMinLot(const SetupProposal &p, double capUsd)
{
   if(p.entryRef <= 0 || p.stopLoss <= 0 || g_TickValue <= 0 || g_TickSize <= 0) return false;
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double minLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   if(minLot <= 0) minLot = 0.01;
   double riskAtMin = slDist * (g_TickValue * minLot / g_TickSize);
   return (riskAtMin > capUsd * 1.10);
}

//+------------------------------------------------------------------+
//| CONTEXT BONUS — universal point system applied per proposal      |
//|   Strategy raw score + contextual confluence                     |
//+------------------------------------------------------------------+
void ApplyContextScore(SetupProposal &p)
{
   if(!p.valid) return;
   double bonus = 0;
   string f = "";
   int dir = p.isBuy ? 1 : -1;
   
   // HTF confluence
   if(g_M15Bias == dir)        { bonus += 12; f += "+12 M15 align\n"; }
   else if(g_M15Bias == -dir)  { bonus -= 15; f += "-15 M15 against\n"; }
   if(g_H1Bias == dir)         { bonus += 8;  f += "+8 H1 align\n"; }
   else if(g_H1Bias == -dir)   { bonus -= 10; f += "-10 H1 against\n"; }
   if(g_M15Bias == dir && g_H1Bias == dir) { bonus += 5; f += "+5 M15+H1 align bonus\n"; }
   
   // ADX context
   if(g_M5ADX >= 28)      { bonus += 10; f += "+10 ADX strong (" + DoubleToString(g_M5ADX, 1) + ")\n"; }
   else if(g_M5ADX >= 20) { bonus += 5;  f += "+5 ADX moderate (" + DoubleToString(g_M5ADX, 1) + ")\n"; }
   else if(g_M5ADX < 18 && p.kind != SETUP_FADE && p.kind != SETUP_VWAP_RECLAIM)
                          { bonus -= 6;  f += "-6 ADX chop (" + DoubleToString(g_M5ADX, 1) + ")\n"; }
   
   // Microstructure
   if((dir > 0 && g_Micro.imbalance > 20) || (dir < 0 && g_Micro.imbalance < -20))
                          { bonus += 12; f += "+12 OF imbalance with dir\n"; }
   else if((dir > 0 && g_Micro.imbalance < -20) || (dir < 0 && g_Micro.imbalance > 20))
                          { bonus -= 10; f += "-10 OF imbalance against\n"; }
   if((dir > 0 && g_Micro.tickVelocity > 8) || (dir < 0 && g_Micro.tickVelocity < -8))
                          { bonus += 8;  f += "+8 tick velocity with dir\n"; }
   
   // Volatility regime
   if(g_VolRegime == VOL_NORMAL || g_VolRegime == VOL_ELEVATED)
                          { bonus += 8;  f += "+8 vol normal/elevated\n"; }
   else if(g_VolRegime == VOL_EXPLOSIVE)
   {
      if(p.kind == SETUP_SPIKE || p.kind == SETUP_NEWS_CONT)
                          { bonus += 12; f += "+12 vol explosive (spike/news)\n"; }
      else                { bonus -= 4;  f += "-4 vol explosive (chop risk)\n"; }
   }
   else if(g_VolRegime == VOL_QUIET && p.kind != SETUP_VWAP_RECLAIM && p.kind != SETUP_FADE)
                          { bonus -= 10; f += "-10 vol quiet\n"; }
   
   // VWAP context
   if(g_VWAPValid)
   {
      double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
      if((dir > 0 && bid > g_SessionVWAP) || (dir < 0 && bid < g_SessionVWAP))
                          { bonus += 6;  f += "+6 VWAP side OK\n"; }
      else                { bonus -= 4;  f += "-4 VWAP side wrong\n"; }
   }
   
   // Spread quality (binary tight/spike)
   if(g_Spread <= 25)        { bonus += 5;  f += "+5 tight spread\n"; }
   if(g_SpreadSpike)         { bonus -= 15; f += "-15 spread spike\n"; }
   
   // Spread cost vs target distance — penalize if spread eats too much of T1 distance
   if(p.t1 > 0 && p.entryRef > 0 && g_Point > 0)
   {
      double tDistPts = MathAbs(p.t1 - p.entryRef) / g_Point;
      if(tDistPts > 0)
      {
         double costFrac = g_Spread / tDistPts;
         if(costFrac > 0.25)        { bonus -= 12; f += "-12 spread cost extreme (" + DoubleToString(costFrac*100, 0) + "%T1)\n"; }
         else if(costFrac > 0.15)   { bonus -=  7; f += "-7 spread cost high (" + DoubleToString(costFrac*100, 0) + "%T1)\n"; }
         else if(costFrac > 0.08)   { bonus -=  3; f += "-3 spread cost moderate (" + DoubleToString(costFrac*100, 0) + "%T1)\n"; }
      }
   }
   
   // DOM order-flow imbalance — bid/ask depth aligned with direction
   // v1.11: gated by InpUseDomImbalance. Most retail brokers serve a synthetic
   // DOM derived from their B-book pool, which makes this signal unreliable
   // (constant ~50/50 or jittery ±90 swings). DISABLE on retail accounts that
   // don't pass through real exchange depth.
   if(InpUseDomImbalance && g_MarketBookSubscribed)
   {
      if(MathAbs(g_DomImbalancePct) >= 60)
      {
         bool aligned = (dir > 0 && g_DomImbalancePct > 0) || (dir < 0 && g_DomImbalancePct < 0);
         if(aligned)               { bonus += 8;  f += "+8 DOM imbalance aligned (" + DoubleToString(g_DomImbalancePct, 0) + "%)\n"; }
         else                      { bonus -= 6;  f += "-6 DOM imbalance against (" + DoubleToString(g_DomImbalancePct, 0) + "%)\n"; }
      }
      else if(MathAbs(g_DomImbalancePct) >= 40)
      {
         bool aligned = (dir > 0 && g_DomImbalancePct > 0) || (dir < 0 && g_DomImbalancePct < 0);
         if(aligned)               { bonus += 3;  f += "+3 DOM imbalance soft-aligned\n"; }
         else                      { bonus -= 2;  f += "-2 DOM imbalance soft-against\n"; }
      }
   }
   
   // Session-open microstructure window — boost rare/explosive setups, penalize trend-following
   if(g_InSessionOpenWindow_London || g_InSessionOpenWindow_NY)
   {
      string winLbl = g_InSessionOpenWindow_NY ? "NY-OPEN" : "LON-OPEN";
      if(p.kind == SETUP_SPIKE || p.kind == SETUP_NEWS_CONT)
                                { bonus += 6;  f += "+6 " + winLbl + " window (boost SPK/NEWS)\n"; }
      else if(p.kind == SETUP_FLAG || p.kind == SETUP_ABCD)
                                { bonus -= 4;  f += "-4 " + winLbl + " window (penalty FLG/ABCD)\n"; }
   }
   
   // Magnet proximity penalty (target obstruction)
   if(g_Levels.nearMagnet && p.kind != SETUP_LEVEL_BOUNCE && p.kind != SETUP_LEVEL_BREAK && p.kind != SETUP_FADE)
                          { bonus -= 10; f += "-10 magnet obstacle (" + g_Levels.magnetType + ")\n"; }
   
   // News context
   if(g_News.inBlackout && p.kind != SETUP_NEWS_CONT && p.kind != SETUP_SPIKE)
                          { bonus -= 30; f += "-30 news blackout (non-news setup)\n"; }
   else if(!g_News.hasUpcoming || g_News.minutesToNext > 60)
                          { bonus += 3;  f += "+3 calendar clear\n"; }
   
   // Recent-loss penalty (gentle cooldown effect)
   if(g_RecentLosses >= 2) { bonus -= 6;  f += "-6 recent losses(2+)\n"; }
   if(g_ConsecLosses >= 2) { bonus -= 8;  f += "-8 consec losses(2+)\n"; }
   
   p.ctxScore = bonus;
   
   // Apply category prior weight (rare/high-conviction get small boost, frequent/moderate get small penalty)
   double catW = SetupCategoryWeight(p.kind);
   p.score = (p.rawScore + bonus) * catW;
   if(catW != 1.0)
      f += (catW > 1.0 ? "+" : "") + DoubleToString((catW - 1.0) * 100, 1) + "% category prior weight\n";
   
   p.factors += f;
}

//+------------------------------------------------------------------+
//| ─── SETUP: SPIKE / MOMENTUM CAPTURE ─────────────────────────────│
//+------------------------------------------------------------------+
SetupProposal Setup_Spike()
{
   SetupProposal p;
   ProposalReset(p, SETUP_SPIKE);
   if(!InpEnableSpike) return p;
   if(g_M1ATR <= 0 || g_Point <= 0) return p;
   
   // Spread-blowout guard: if current spread is meaningfully above its rolling p90,
   // the spike has already happened and we'd be entering into the slippage explosion.
   if(g_SpreadP90 > 0 && g_Spread > g_SpreadP90 * InpSpikeMaxSpreadP90Mul) return p;
   
   // Last completed M1 bar
   double o = iOpen(_Symbol, PERIOD_M1, 1);
   double c = iClose(_Symbol, PERIOD_M1, 1);
   double h = iHigh(_Symbol, PERIOD_M1, 1);
   double l = iLow(_Symbol, PERIOD_M1, 1);
   double bodySize = MathAbs(c - o);
   double atrBodyThresh = g_M1ATR * InpSpikeBodyAtrMult;
   if(bodySize < atrBodyThresh) return p;
   
   bool bull = (c > o);
   bool bear = (c < o);
   if(!bull && !bear) return p;
   
   // Microstructure imbalance must agree
   if(bull && g_Micro.imbalance < InpSpikeMinImbalance) return p;
   if(bear && g_Micro.imbalance > -InpSpikeMinImbalance) return p;
   if(MathAbs(g_Micro.tickVelocity) < InpSpikeMinTickVel) return p;
   if(bull && g_Micro.tickVelocity < 0) return p;
   if(bear && g_Micro.tickVelocity > 0) return p;
   
   // Build proposal
   p.isBuy = bull;
   p.entryRef = bull ? SymbolInfoDouble(_Symbol, SYMBOL_ASK) : SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double bodyMid = (o + c) / 2.0;
   double slPad = g_M1ATR * InpSpikeSlAtrMult;
   if(bull)
   {
      p.stopLoss = NormalizeDouble(MathMin(bodyMid, l) - slPad, g_Digits);
   }
   else
   {
      p.stopLoss = NormalizeDouble(MathMax(bodyMid, h) + slPad, g_Digits);
   }
   
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = MathMax(slDist * 1.5, g_M1ATR * 1.5);
   p.t1 = bull ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   // Raw score: base + body strength bonus
   double bodyMult = bodySize / g_M1ATR;
   double bodyBonus = MathMin(15.0, (bodyMult - InpSpikeBodyAtrMult) * 8.0);
   if(g_M5MomentumBurst) bodyBonus += 6;
   if((bull && g_M5LastClosedBull) || (bear && g_M5LastClosedBear)) bodyBonus += 4;
   p.rawScore = InpSpikeRawScore + bodyBonus;
   p.factors = "Spike body=" + DoubleToString(bodyMult, 2) + "xATR; imb=" + DoubleToString(g_Micro.imbalance, 0) +
               "%; vel=" + DoubleToString(g_Micro.tickVelocity, 1) + "pts/s\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: BULL/BEAR FLAG (M5 continuation) ─────────────────────│
//+------------------------------------------------------------------+
SetupProposal Setup_BullBearFlag()
{
   SetupProposal p;
   ProposalReset(p, SETUP_FLAG);
   if(!InpEnableFlag) return p;
   if(g_M5Trend == 0 || g_M5TrendPct < InpFlagMinM5TrendPct) return p;
   if(g_M5ATR <= 0) return p;
   
   bool bullishTrend = (g_M5Trend > 0);
   
   // Look at last few completed M5 bars for pullback
   double opens[], closes[], highs[], lows[];
   ArraySetAsSeries(opens, true);
   ArraySetAsSeries(closes, true);
   ArraySetAsSeries(highs, true);
   ArraySetAsSeries(lows, true);
   int need = InpFlagMaxPullbackBars + 2;
   if(CopyOpen(_Symbol, PERIOD_M5, 1, need, opens) != need ||
      CopyClose(_Symbol, PERIOD_M5, 1, need, closes) != need ||
      CopyHigh(_Symbol, PERIOD_M5, 1, need, highs) != need ||
      CopyLow(_Symbol, PERIOD_M5, 1, need, lows) != need)
      return p;
   
   // Count consecutive counter-color bars (1..maxPullback) that hold the EMA9
   int pullbacks = 0;
   double pbHigh = -1.0e100, pbLow = 1.0e100;
   for(int i = 1; i <= InpFlagMaxPullbackBars; i++)  // bars 1..N from last closed
   {
      bool counter = bullishTrend ? (closes[i-1] < opens[i-1]) : (closes[i-1] > opens[i-1]);
      if(!counter) break;
      pullbacks++;
      if(highs[i-1] > pbHigh) pbHigh = highs[i-1];
      if(lows[i-1] < pbLow)   pbLow  = lows[i-1];
   }
   if(pullbacks < InpFlagMinPullbackBars) return p;
   
   // EMA9 must hold (in a bull flag, pullback lows must be ≥ EMA9 - small buffer)
   double ema9 = g_M5EMAValue;
   if(ema9 <= 0) return p;
   double emaBuf = g_M5ATR * 0.2;
   if(bullishTrend && pbLow < ema9 - emaBuf) return p;
   if(!bullishTrend && pbHigh > ema9 + emaBuf) return p;
   
   // Trigger: last M5 bar must break the pullback range in trend direction
   double last0Open = iOpen(_Symbol, PERIOD_M5, 0);
   double last0Close = iClose(_Symbol, PERIOD_M5, 0);
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   bool trigger = bullishTrend ? (bid > pbHigh + g_Point) : (ask < pbLow - g_Point);
   if(!trigger) return p;
   
   p.isBuy = bullishTrend;
   p.entryRef = bullishTrend ? ask : bid;
   double slPad = g_M5ATR * InpFlagSlAtrMult;
   p.stopLoss = bullishTrend ? NormalizeDouble(pbLow - slPad, g_Digits)
                              : NormalizeDouble(pbHigh + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = slDist * 1.6;
   p.t1 = bullishTrend ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                        : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   double trendBonus = MathMin(12.0, (g_M5TrendPct - InpFlagMinM5TrendPct) * 0.4);
   if(g_M5MomentumOK) trendBonus += 5;
   if(pullbacks <= 2) trendBonus += 4; // shallow flag = clean
   p.rawScore = InpFlagRawScore + trendBonus;
   p.factors = "Flag pullbacks=" + IntegerToString(pullbacks) +
               "; trendPct=" + DoubleToString(g_M5TrendPct, 0) + "%; ema9=" + DoubleToString(ema9, g_Digits) + "\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: OPENING RANGE BREAKOUT (London / NY anchor) ─────────│
//+------------------------------------------------------------------+
SetupProposal Setup_ORB()
{
   SetupProposal p;
   ProposalReset(p, SETUP_ORB);
   if(!InpEnableORB) return p;
   if(!g_ORBValid) return p;
   if(g_M5ATR <= 0) return p;
   
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   bool brokeUp   = (bid > g_ORBHigh + g_Point);
   bool brokeDown = (ask < g_ORBLow - g_Point);
   if(!brokeUp && !brokeDown) return p;
   
   // Make sure we are AFTER the ORB window (anchor + N minutes)
   datetime anchor = GetVWAPORBAnchorTime();
   if(TimeCurrent() < anchor + InpORBFirstMinutes * 60) return p;
   
   bool isBuy = brokeUp;
   
   // Optional retest: latest M1 bar must have touched ORB line and rejected
   if(InpORBRequireRetest)
   {
      double l1 = iLow(_Symbol, PERIOD_M1, 1);
      double h1 = iHigh(_Symbol, PERIOD_M1, 1);
      double c1 = iClose(_Symbol, PERIOD_M1, 1);
      bool retestOK = false;
      if(isBuy && l1 <= g_ORBHigh + g_M1ATR * 0.2 && c1 > g_ORBHigh) retestOK = true;
      if(!isBuy && h1 >= g_ORBLow - g_M1ATR * 0.2 && c1 < g_ORBLow) retestOK = true;
      if(!retestOK) return p;
   }
   
   p.isBuy = isBuy;
   p.entryRef = isBuy ? ask : bid;
   double slPad = g_M5ATR * InpORBSlAtrMult;
   p.stopLoss = isBuy ? NormalizeDouble(g_ORBLow - slPad, g_Digits)
                       : NormalizeDouble(g_ORBHigh + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double rangeSize = g_ORBHigh - g_ORBLow;
   double tDist = MathMax(slDist * 1.5, rangeSize);
   p.t1 = isBuy ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                 : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   double penetrationPts = isBuy ? (bid - g_ORBHigh) / g_Point : (g_ORBLow - ask) / g_Point;
   double penBonus = MathMin(10.0, penetrationPts * 0.2);
   if(g_VWAPValid)
   {
      bool vwapAlign = isBuy ? (bid > g_SessionVWAP) : (bid < g_SessionVWAP);
      if(vwapAlign) penBonus += 6;
   }
   p.rawScore = InpORBRawScore + penBonus;
   p.factors = "ORB break " + (isBuy ? "UP" : "DN") + " by " + DoubleToString(penetrationPts, 0) +
               " pts; range=" + DoubleToString(rangeSize / g_Point, 0) + " pts\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: VWAP RECLAIM ─────────────────────────────────────────│
//+------------------------------------------------------------------+
SetupProposal Setup_VWAPReclaim()
{
   SetupProposal p;
   ProposalReset(p, SETUP_VWAP_RECLAIM);
   if(!InpEnableVWAPReclaim) return p;
   if(!g_VWAPValid) return p;
   if(g_M5ATR <= 0) return p;
   
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double l1 = iLow(_Symbol, PERIOD_M1, 1);
   double h1 = iHigh(_Symbol, PERIOD_M1, 1);
   double c1 = iClose(_Symbol, PERIOD_M1, 1);
   double minDist = InpVWAPMinDistPts * g_Point;
   
   // Bullish reclaim: recent low was below VWAP and now closed back above
   bool bullReclaim = (l1 < g_SessionVWAP - minDist) && (c1 > g_SessionVWAP) && (bid > g_SessionVWAP);
   // Bearish reclaim: recent high was above VWAP and now closed back below
   bool bearReclaim = (h1 > g_SessionVWAP + minDist) && (c1 < g_SessionVWAP) && (bid < g_SessionVWAP);
   if(!bullReclaim && !bearReclaim) return p;
   
   p.isBuy = bullReclaim;
   p.entryRef = bullReclaim ? ask : bid;
   double slPad = g_M5ATR * InpVWAPSlAtrMult;
   p.stopLoss = bullReclaim ? NormalizeDouble(l1 - slPad, g_Digits)
                              : NormalizeDouble(h1 + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = slDist * 1.5;
   p.t1 = bullReclaim ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                       : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   double dist = MathAbs(bid - g_SessionVWAP) / g_Point;
   double scoreBonus = MathMin(8.0, dist * 0.1);
   p.rawScore = InpVWAPRawScore + scoreBonus;
   p.factors = "VWAP reclaim " + (bullReclaim ? "UP" : "DN") + " | dist=" + DoubleToString(dist, 0) + "pts\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: VWAP REJECT ──────────────────────────────────────────│
//|   Price tested VWAP from one side and got rejected (continuation)│
//+------------------------------------------------------------------+
SetupProposal Setup_VWAPReject()
{
   SetupProposal p;
   ProposalReset(p, SETUP_VWAP_REJECT);
   if(!InpEnableVWAPReject) return p;
   if(!g_VWAPValid) return p;
   if(g_M5ATR <= 0) return p;
   
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double l1 = iLow(_Symbol, PERIOD_M1, 1);
   double h1 = iHigh(_Symbol, PERIOD_M1, 1);
   double o1 = iOpen(_Symbol, PERIOD_M1, 1);
   double c1 = iClose(_Symbol, PERIOD_M1, 1);
   double touchTol = g_M1ATR * 0.3;
   
   // Rejection from below (price was below, touched VWAP, stayed below = bearish continuation)
   bool bearReject = (h1 >= g_SessionVWAP - touchTol && h1 <= g_SessionVWAP + touchTol)
                     && (c1 < o1) && (bid < g_SessionVWAP) && g_M5Trend < 0;
   // Rejection from above (bullish continuation below VWAP doesn't make sense; mirror requires uptrend)
   bool bullReject = (l1 >= g_SessionVWAP - touchTol && l1 <= g_SessionVWAP + touchTol)
                     && (c1 > o1) && (bid > g_SessionVWAP) && g_M5Trend > 0;
   if(!bullReject && !bearReject) return p;
   
   p.isBuy = bullReject;
   p.entryRef = bullReject ? ask : bid;
   double slPad = g_M5ATR * InpVWAPSlAtrMult;
   p.stopLoss = bullReject ? NormalizeDouble(g_SessionVWAP - slPad, g_Digits)
                            : NormalizeDouble(g_SessionVWAP + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = slDist * 1.4;
   p.t1 = bullReject ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                      : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   p.rawScore = InpVWAPRawScore + 5.0;  // small bonus for trend alignment
   p.factors = "VWAP reject " + (bullReject ? "BULL" : "BEAR") + " (continuation in M5 trend)\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: ABCD PATTERN ─────────────────────────────────────────│
//|   A→B impulse, B→C pullback (30..70%), C→D continuation trigger  │
//+------------------------------------------------------------------+
SetupProposal Setup_ABCD()
{
   SetupProposal p;
   ProposalReset(p, SETUP_ABCD);
   if(!InpEnableABCD) return p;
   if(g_M5ATR <= 0) return p;
   // ABCD pattern detection is noisy in chop — require an actual M5 trend (ADX)
   if(g_M5ADX < InpABCDMinADX) return p;
   
   // We use M5 bars. Window: last 10 closed bars.
   int need = 10;
   double highs[], lows[], opens[], closes[];
   ArraySetAsSeries(highs, true);
   ArraySetAsSeries(lows, true);
   ArraySetAsSeries(opens, true);
   ArraySetAsSeries(closes, true);
   if(CopyHigh(_Symbol, PERIOD_M5, 1, need, highs) != need ||
      CopyLow(_Symbol, PERIOD_M5, 1, need, lows) != need ||
      CopyOpen(_Symbol, PERIOD_M5, 1, need, opens) != need ||
      CopyClose(_Symbol, PERIOD_M5, 1, need, closes) != need)
      return p;
   
   // Bullish ABCD: find lowest low (A) in older half, highest high (B) after it,
   //   then current low (C) between, and last bar closing back up (D trigger)
   int aIdxMin = -1; double aLow = 1.0e100;
   for(int i = 4; i < need; i++) { if(lows[i] < aLow) { aLow = lows[i]; aIdxMin = i; } }
   int bIdxMax = -1; double bHigh = -1.0e100;
   for(int i = 1; i < (aIdxMin >= 0 ? aIdxMin : 1); i++) { if(highs[i] > bHigh) { bHigh = highs[i]; bIdxMax = i; } }
   
   // Bullish ABCD validity
   bool bullValid = false;
   double cLow = 0;
   if(aIdxMin > 0 && bIdxMax > 0 && aIdxMin > bIdxMax)
   {
      // C is the lowest low between B and now
      cLow = 1.0e100;
      for(int i = 0; i < bIdxMax; i++) { if(lows[i] < cLow) cLow = lows[i]; }
      double impulse = (bHigh - aLow) / g_Point;
      double pullback = (bHigh - cLow);
      double pbRatio = (bHigh > aLow) ? pullback / (bHigh - aLow) : 0;
      bool last0Bull = (iClose(_Symbol, PERIOD_M5, 0) > iOpen(_Symbol, PERIOD_M5, 0));
      bool brokeC = (iClose(_Symbol, PERIOD_M5, 0) > iHigh(_Symbol, PERIOD_M5, 1));
      if(impulse >= InpABCDImpulseMinPts && pbRatio >= InpABCDPullbackMin && pbRatio <= InpABCDPullbackMax &&
         (last0Bull || brokeC))
         bullValid = true;
   }
   
   // Bearish ABCD: highest high (A), lowest low (B), highest current (C), then resume down
   int aHighIdx = -1; double aHigh = -1.0e100;
   for(int i = 4; i < need; i++) { if(highs[i] > aHigh) { aHigh = highs[i]; aHighIdx = i; } }
   int bLowIdx = -1; double bLow = 1.0e100;
   for(int i = 1; i < (aHighIdx >= 0 ? aHighIdx : 1); i++) { if(lows[i] < bLow) { bLow = lows[i]; bLowIdx = i; } }
   bool bearValid = false;
   double cHigh = 0;
   if(aHighIdx > 0 && bLowIdx > 0 && aHighIdx > bLowIdx)
   {
      cHigh = -1.0e100;
      for(int i = 0; i < bLowIdx; i++) { if(highs[i] > cHigh) cHigh = highs[i]; }
      double impulse = (aHigh - bLow) / g_Point;
      double pullback = (cHigh - bLow);
      double pbRatio = (aHigh > bLow) ? pullback / (aHigh - bLow) : 0;
      bool last0Bear = (iClose(_Symbol, PERIOD_M5, 0) < iOpen(_Symbol, PERIOD_M5, 0));
      bool brokeC = (iClose(_Symbol, PERIOD_M5, 0) < iLow(_Symbol, PERIOD_M5, 1));
      if(impulse >= InpABCDImpulseMinPts && pbRatio >= InpABCDPullbackMin && pbRatio <= InpABCDPullbackMax &&
         (last0Bear || brokeC))
         bearValid = true;
   }
   
   if(!bullValid && !bearValid) return p;
   
   // If both pop up (rare), pick the one matching M5 trend
   bool isBuy;
   if(bullValid && bearValid) isBuy = (g_M5Trend >= 0);
   else                       isBuy = bullValid;
   
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   p.isBuy = isBuy;
   p.entryRef = isBuy ? ask : bid;
   double slPad = g_M5ATR * InpABCDSlAtrMult;
   if(isBuy) p.stopLoss = NormalizeDouble(cLow - slPad, g_Digits);
   else      p.stopLoss = NormalizeDouble(cHigh + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = slDist * 1.6;
   p.t1 = isBuy ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                 : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   p.rawScore = InpABCDRawScore;
   p.factors = "ABCD " + (isBuy ? "BULL" : "BEAR") + " (impulse + 30-70% pullback + resume)\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: KEY-LEVEL BOUNCE ─────────────────────────────────────│
//+------------------------------------------------------------------+
SetupProposal Setup_LevelBounce()
{
   SetupProposal p;
   ProposalReset(p, SETUP_LEVEL_BOUNCE);
   if(!InpEnableLevelBounce) return p;
   if(g_M5ATR <= 0) return p;
   // Require either a confirmed magnet level OR an actual trend; loose bounce in chop = noise
   if(!g_Levels.nearMagnet && g_M5ADX < InpLevelMinADX) return p;
   
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double maxDist = InpLevelMaxDistPts * g_Point;
   
   // Pick the closest support / resistance candidate
   //   Supports below: roundBelow, sessionLow, PDL, PWL, recentPivotLow
   //   Resistances above: roundAbove, sessionHigh, PDH, PWH, recentPivotHigh
   double supportLevels[5], resistLevels[5];
   string  supLabels[5], resLabels[5];
   supportLevels[0] = g_Levels.nearestRoundBelow; supLabels[0] = "ROUND_LOW";
   supportLevels[1] = g_Levels.sessionLow;        supLabels[1] = "SESS_LOW";
   supportLevels[2] = g_PDL;                      supLabels[2] = "PDL";
   supportLevels[3] = g_PWL;                      supLabels[3] = "PWL";
   supportLevels[4] = g_Levels.recentPivotLow;    supLabels[4] = "PIVOT_LOW";
   resistLevels[0] = g_Levels.nearestRoundAbove;  resLabels[0] = "ROUND_HIGH";
   resistLevels[1] = g_Levels.sessionHigh;        resLabels[1] = "SESS_HIGH";
   resistLevels[2] = g_PDH;                       resLabels[2] = "PDH";
   resistLevels[3] = g_PWH;                       resLabels[3] = "PWH";
   resistLevels[4] = g_Levels.recentPivotHigh;    resLabels[4] = "PIVOT_HIGH";
   
   double bestSup = 0; string supLab = "";
   double bestRes = 0; string resLab = "";
   double bestSupDist = 1.0e100, bestResDist = 1.0e100;
   for(int i = 0; i < 5; i++)
   {
      if(supportLevels[i] > 0 && supportLevels[i] < bid)
      {
         double d = bid - supportLevels[i];
         if(d <= maxDist && d < bestSupDist) { bestSup = supportLevels[i]; bestSupDist = d; supLab = supLabels[i]; }
      }
      if(resistLevels[i] > 0 && resistLevels[i] > bid)
      {
         double d = resistLevels[i] - bid;
         if(d <= maxDist && d < bestResDist) { bestRes = resistLevels[i]; bestResDist = d; resLab = resLabels[i]; }
      }
   }
   
   // Bullish bounce: latest M1 wicked into support and closed above
   double l1 = iLow(_Symbol, PERIOD_M1, 1);
   double h1 = iHigh(_Symbol, PERIOD_M1, 1);
   double c1 = iClose(_Symbol, PERIOD_M1, 1);
   double o1 = iOpen(_Symbol, PERIOD_M1, 1);
   bool bullBounce = (bestSup > 0 && l1 <= bestSup + maxDist && c1 > bestSup && c1 > o1);
   bool bearBounce = (bestRes > 0 && h1 >= bestRes - maxDist && c1 < bestRes && c1 < o1);
   if(!bullBounce && !bearBounce) return p;
   
   bool isBuy;
   string label;
   double levelPx;
   if(bullBounce && bearBounce)
   {
      // Pick the one with the longer rejection wick
      double bullWick = c1 - l1;
      double bearWick = h1 - c1;
      if(bullWick > bearWick) { isBuy = true;  label = supLab; levelPx = bestSup; }
      else                    { isBuy = false; label = resLab; levelPx = bestRes; }
   }
   else if(bullBounce) { isBuy = true;  label = supLab; levelPx = bestSup; }
   else                { isBuy = false; label = resLab; levelPx = bestRes; }
   
   p.isBuy = isBuy;
   p.entryRef = isBuy ? ask : bid;
   double slPad = g_M5ATR * InpLevelSlAtrMult;
   p.stopLoss = isBuy ? NormalizeDouble(levelPx - slPad, g_Digits)
                       : NormalizeDouble(levelPx + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = slDist * 1.7;
   p.t1 = isBuy ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                 : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   double wickPts = isBuy ? (c1 - l1) / g_Point : (h1 - c1) / g_Point;
   double wickBonus = MathMin(10.0, wickPts * 0.08);
   p.rawScore = InpLevelRawScore + wickBonus + 4.0;  // bounce inherently counter-intuitive — small base bonus
   p.factors = "Bounce " + (isBuy ? "off " : "from ") + label + " @ " + DoubleToString(levelPx, g_Digits) +
               " | wick=" + DoubleToString(wickPts, 0) + "pts\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: KEY-LEVEL BREAK ──────────────────────────────────────│
//+------------------------------------------------------------------+
SetupProposal Setup_LevelBreak()
{
   SetupProposal p;
   ProposalReset(p, SETUP_LEVEL_BREAK);
   if(!InpEnableLevelBreak) return p;
   if(g_M5ATR <= 0 || g_M1ATR <= 0) return p;
   // Require either a confirmed magnet level OR an actual trend; level breaks in chop = fakeouts
   if(!g_Levels.nearMagnet && g_M5ADX < InpLevelMinADX) return p;
   
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   
   // Candidate levels (above for bullish break, below for bearish)
   double upLevels[5];   string upLab[5];
   double dnLevels[5];   string dnLab[5];
   upLevels[0] = g_Levels.nearestRoundAbove;  upLab[0] = "ROUND";
   upLevels[1] = g_Levels.sessionHigh;        upLab[1] = "SESS_H";
   upLevels[2] = g_PDH;                       upLab[2] = "PDH";
   upLevels[3] = g_PWH;                       upLab[3] = "PWH";
   upLevels[4] = g_Levels.recentPivotHigh;    upLab[4] = "PIVOT_H";
   dnLevels[0] = g_Levels.nearestRoundBelow;  dnLab[0] = "ROUND";
   dnLevels[1] = g_Levels.sessionLow;         dnLab[1] = "SESS_L";
   dnLevels[2] = g_PDL;                       dnLab[2] = "PDL";
   dnLevels[3] = g_PWL;                       dnLab[3] = "PWL";
   dnLevels[4] = g_Levels.recentPivotLow;     dnLab[4] = "PIVOT_L";
   
   double c1 = iClose(_Symbol, PERIOD_M1, 1);
   double o1 = iOpen(_Symbol, PERIOD_M1, 1);
   double bodySize = MathAbs(c1 - o1);
   bool atrExpand = (bodySize >= g_M1ATR * 0.8);
   
   // Bullish break: bid clearly above one of the upLevels and last bar bullish + ATR-expansion
   double pickLevel = 0; string pickLab = ""; bool isBuy = false; double penPts = 0;
   for(int i = 0; i < 5; i++)
   {
      double L = upLevels[i];
      if(L <= 0) continue;
      double pen = (bid - L) / g_Point;
      if(pen > 0 && pen < InpLevelMaxDistPts && pen > penPts && c1 > L && c1 > o1 && atrExpand)
      { pickLevel = L; pickLab = upLab[i]; isBuy = true; penPts = pen; }
   }
   for(int i = 0; i < 5; i++)
   {
      double L = dnLevels[i];
      if(L <= 0) continue;
      double pen = (L - ask) / g_Point;
      if(pen > 0 && pen < InpLevelMaxDistPts && pen > penPts && c1 < L && c1 < o1 && atrExpand)
      { pickLevel = L; pickLab = dnLab[i]; isBuy = false; penPts = pen; }
   }
   if(pickLevel <= 0) return p;
   
   p.isBuy = isBuy;
   p.entryRef = isBuy ? ask : bid;
   double slPad = g_M5ATR * InpLevelSlAtrMult;
   p.stopLoss = isBuy ? NormalizeDouble(pickLevel - slPad, g_Digits)
                       : NormalizeDouble(pickLevel + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = slDist * 1.5;
   p.t1 = isBuy ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                 : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   double scoreBonus = MathMin(10.0, penPts * 0.25);
   p.rawScore = InpLevelRawScore + scoreBonus + 6.0;  // breaks are higher conviction than bounces
   p.factors = "Break " + (isBuy ? "above " : "below ") + pickLab + " @ " + DoubleToString(pickLevel, g_Digits) +
               " | pen=" + DoubleToString(penPts, 0) + "pts (ATR expansion)\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: EMA PULLBACK (M5 EMA9 / M15 EMA20) ───────────────────│
//+------------------------------------------------------------------+
SetupProposal Setup_EMAPullback()
{
   SetupProposal p;
   ProposalReset(p, SETUP_EMA_PULLBACK);
   if(!InpEnableEMAPullback) return p;
   
   ENUM_TIMEFRAMES tf = InpEMAPullbackTF;
   int handle = (tf == PERIOD_M15) ? g_HandleM15EMA : g_HandleM5EMA;
   double atrUse = (tf == PERIOD_M15) ? g_M5ATR * 1.5 : g_M5ATR;
   if(atrUse <= 0) return p;
   
   double ema[];
   ArraySetAsSeries(ema, true);
   if(CopyBuffer(handle, 0, 0, 1, ema) < 1) return p;
   double emaVal = ema[0];
   if(emaVal <= 0) return p;
   
   // Need an established trend
   if(g_M5Trend == 0) return p;
   bool isBuy = (g_M5Trend > 0);
   
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   
   // Last completed bar must have touched (or come within InpEMAPullbackMaxPts of) the EMA, then closed in trend dir
   double l1 = iLow(_Symbol, tf, 1);
   double h1 = iHigh(_Symbol, tf, 1);
   double c1 = iClose(_Symbol, tf, 1);
   double o1 = iOpen(_Symbol, tf, 1);
   double touchTol = InpEMAPullbackMaxPts * g_Point;
   
   bool touched = false;
   if(isBuy && l1 <= emaVal + touchTol) touched = true;
   if(!isBuy && h1 >= emaVal - touchTol) touched = true;
   if(!touched) return p;
   bool resumption = isBuy ? (c1 > o1) : (c1 < o1);
   if(!resumption) return p;
   
   p.isBuy = isBuy;
   p.entryRef = isBuy ? ask : bid;
   double slPad = atrUse * InpEMAPullbackSlAtrMult;
   p.stopLoss = isBuy ? NormalizeDouble(MathMin(emaVal, l1) - slPad, g_Digits)
                       : NormalizeDouble(MathMax(emaVal, h1) + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = slDist * 1.6;
   p.t1 = isBuy ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                 : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   p.rawScore = InpEMAPullbackRawScore + (g_M5MomentumOK ? 5.0 : 0.0);
   p.factors = "EMA pullback " + (tf == PERIOD_M15 ? "M15-20" : "M5-9") +
               " @ " + DoubleToString(emaVal, g_Digits) + " (" + (isBuy ? "long" : "short") + ")\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: POST-NEWS CONTINUATION ───────────────────────────────│
//+------------------------------------------------------------------+
SetupProposal Setup_NewsContinuation()
{
   SetupProposal p;
   ProposalReset(p, SETUP_NEWS_CONT);
   if(!InpEnableNewsCont) return p;
   if(g_LastHighImpactEventTime <= 0) return p;
   
   long secSince = (long)(TimeCurrent() - g_LastHighImpactEventTime);
   if(secSince < 0 || secSince > InpNewsContMaxSecAfter) return p;
   if(g_M1ATR <= 0) return p;
   
   // Last completed M1 must be a strong body in the direction we want to continue
   double o1 = iOpen(_Symbol, PERIOD_M1, 1);
   double c1 = iClose(_Symbol, PERIOD_M1, 1);
   double h1 = iHigh(_Symbol, PERIOD_M1, 1);
   double l1 = iLow(_Symbol, PERIOD_M1, 1);
   double bodyAbs = MathAbs(c1 - o1);
   if(bodyAbs < g_M1ATR * InpNewsContBodyAtrMult) return p;
   
   bool bull = (c1 > o1);
   bool bear = (c1 < o1);
   if(!bull && !bear) return p;
   
   p.isBuy = bull;
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   p.entryRef = bull ? ask : bid;
   double slPad = g_M1ATR * InpNewsContSlAtrMult;
   p.stopLoss = bull ? NormalizeDouble(l1 - slPad, g_Digits)
                      : NormalizeDouble(h1 + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = MathMax(slDist * 1.8, g_M1ATR * 2.5);
   p.t1 = bull ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   double freshnessBonus = MathMax(0.0, 10.0 - (secSince / 18.0));  // ~10 if t<18s, ~0 if t>=180s
   p.rawScore = InpNewsContRawScore + freshnessBonus;
   p.factors = "News-cont " + (bull ? "BULL" : "BEAR") + " | " + IntegerToString((int)secSince) +
               "s post HIGH event; body=" + DoubleToString(bodyAbs / g_M1ATR, 2) + "xATR\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| ─── SETUP: FADE (counter-trend at extreme + magnet) ─────────────│
//+------------------------------------------------------------------+
SetupProposal Setup_Fade()
{
   SetupProposal p;
   ProposalReset(p, SETUP_FADE);
   if(!InpEnableFade) return p;
   if(g_M1ATR <= 0) return p;
   
   double rsi = RsiForEntryModel();
   bool longFade  = (rsi <= InpFadeRsiLongCap);    // oversold → buy
   bool shortFade = (rsi >= InpFadeRsiShortFloor); // overbought → sell
   if(!longFade && !shortFade) return p;
   
   if(InpFadeRequireMagnet && !g_Levels.nearMagnet) return p;
   if(InpFadeRequireExhaustion && !g_Micro.exhaustion) return p;
   
   bool isBuy = longFade;
   if(longFade && shortFade) return p;  // contradictory; skip
   
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   p.isBuy = isBuy;
   p.entryRef = isBuy ? ask : bid;
   double slPad = g_M1ATR * InpFadeSlAtrMult;
   double l1 = iLow(_Symbol, PERIOD_M1, 1);
   double h1 = iHigh(_Symbol, PERIOD_M1, 1);
   p.stopLoss = isBuy ? NormalizeDouble(MathMin(l1, bid) - slPad, g_Digits)
                       : NormalizeDouble(MathMax(h1, bid) + slPad, g_Digits);
   double slDist = MathAbs(p.entryRef - p.stopLoss);
   double tDist = slDist * 1.3;  // tight target — fades are quick
   p.t1 = isBuy ? NormalizeDouble(p.entryRef + tDist, g_Digits)
                 : NormalizeDouble(p.entryRef - tDist, g_Digits);
   
   double extremeBonus = isBuy ? (InpFadeRsiLongCap - rsi) * 0.4 : (rsi - InpFadeRsiShortFloor) * 0.4;
   if(extremeBonus < 0) extremeBonus = 0;
   p.rawScore = InpFadeRawScore + extremeBonus;
   p.factors = "Fade " + (isBuy ? "LONG" : "SHORT") + " | RSI=" + DoubleToString(rsi, 1) +
               " | magnet=" + (g_Levels.nearMagnet ? g_Levels.magnetType : "no") +
               " | exh=" + (g_Micro.exhaustion ? "Y" : "N") + "\n";
   p.valid = true;
   ClampProposalSlToRiskCap(p);
   ProposalFinalizeRR(p);
   ApplyContextScore(p);
   return p;
}

//+------------------------------------------------------------------+
//| GLOBAL HARD GATES (after ranker has chosen the winner)           |
//+------------------------------------------------------------------+
bool PassesGlobalHardGates(SetupProposal &p, string &skipReason)
{
   if(!p.valid) { skipReason = "invalid proposal"; return false; }
   
   if(g_Spread > (double)InpEntryMaxSpreadPts)
   { skipReason = "spread " + DoubleToString(g_Spread, 1) + " > max " + IntegerToString(InpEntryMaxSpreadPts); return false; }
   if(InpBlockEntryOnSpreadSpike && g_SpreadSpike)
   { skipReason = "spread spike vs p90"; return false; }
   
   if(g_Halted) { skipReason = "halted"; return false; }
   if(g_DailyPnL <= -InpMaxDailyLoss) { skipReason = "daily loss limit"; return false; }
   if(InpMaxConsecLosses > 0 && g_ConsecLosses >= InpMaxConsecLosses)
   { skipReason = "consec losses cap"; return false; }
   if(g_ConsecLosses > 0 && TimeCurrent() - g_LastLossTime < InpCooldownAfterLoss)
   { skipReason = "cooldown after loss"; return false; }
   if(InpCooldownAfterClose > 0 && g_LastTradeCloseTime > 0 &&
      TimeCurrent() - g_LastTradeCloseTime < InpCooldownAfterClose)
   { skipReason = "cooldown after close"; return false; }
   if(!g_SessionOK) { skipReason = "session closed"; return false; }
   
   // News block (entry-window padding)
   if(BlockNewEntriesDueToHighImpactNews() && p.kind != SETUP_NEWS_CONT)
   { skipReason = "high-impact news window"; return false; }
   
   // RSI caps
   double rsi = RsiForEntryModel();
   if(p.isBuy && InpRsiBlockLongAbove > 0 && rsi >= (double)InpRsiBlockLongAbove && p.kind != SETUP_FADE)
   { skipReason = "RSI " + DoubleToString(rsi, 1) + " >= long cap"; return false; }
   if(!p.isBuy && InpRsiBlockShortBelow > 0 && rsi <= (double)InpRsiBlockShortBelow && p.kind != SETUP_FADE)
   { skipReason = "RSI " + DoubleToString(rsi, 1) + " <= short floor"; return false; }
   
   // M15 / H1 against trade
   bool spikeOrNews = (p.kind == SETUP_SPIKE || p.kind == SETUP_NEWS_CONT);
   if(InpBlockM15VsTrade && !spikeOrNews && p.kind != SETUP_FADE)
   {
      if(p.isBuy && g_M15Bias < 0) { skipReason = "M15 bearish (block long)"; return false; }
      if(!p.isBuy && g_M15Bias > 0) { skipReason = "M15 bullish (block short)"; return false; }
   }
   if(InpBlockH1VsTrade && !spikeOrNews && p.kind != SETUP_FADE)
   {
      if(p.isBuy && g_H1Bias < 0) { skipReason = "H1 bearish (block long)"; return false; }
      if(!p.isBuy && g_H1Bias > 0) { skipReason = "H1 bullish (block short)"; return false; }
   }
   
   // M1 align (skip for spike, fade, news where turns are expected)
   if(InpUseM1AlignFilter && !spikeOrNews && p.kind != SETUP_FADE)
   {
      if(p.isBuy && M1LastNAllBearish())
      { skipReason = "M1 last " + IntegerToString(InpM1AlignBars) + " all bearish"; return false; }
      if(!p.isBuy && M1LastNAllBullish())
      { skipReason = "M1 last " + IntegerToString(InpM1AlignBars) + " all bullish"; return false; }
   }
   
   // R:R floor
   if(p.rrToT1 < InpMinRRToFirstTarget)
   { skipReason = "R:R " + DoubleToString(p.rrToT1, 2) + " < min " + DoubleToString(InpMinRRToFirstTarget, 2); return false; }
   
   // $ risk cap — only reject if even broker-min lot can't fit; otherwise let
   // ExecuteProposal's auto-shrink handle it. This prevents the gate from
   // blocking salvageable setups whose SL placement is technically wider
   // than InpLotSize × cap but which can be shrunk to a tradable lot.
   double allowed = CurrentRiskCapUsd();
   if(RiskExceedsCapAtMinLot(p, allowed))
   {
      double slDistTmp = MathAbs(p.entryRef - p.stopLoss);
      double minLot    = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
      if(minLot <= 0) minLot = 0.01;
      double riskAtMin = slDistTmp * (g_TickValue * minLot / MathMax(g_TickSize, 0.0000001));
      skipReason = "risk $" + DoubleToString(riskAtMin, 2) + " > cap $" + DoubleToString(allowed, 2) +
                   " (even at min lot)";
      return false;
   }
   
   skipReason = "";
   return true;
}

//+------------------------------------------------------------------+
//| Order comment builder                                            |
//+------------------------------------------------------------------+
string BuildEntryOrderComment(const string tag, const int score, const bool isBuy, const string detail)
{
   int maxLen = InpOrderCommentMaxLen;
   if(maxLen < 12) maxLen = 12;
   if(maxLen > 63) maxLen = 63;
   
   string prefix = tag + "-" + IntegerToString(score) + "-" + (isBuy ? "B" : "S") + ":";
   if(StringLen(prefix) > maxLen) return StringSubstr(prefix, 0, maxLen);
   
   string d = detail;
   StringReplace(d, "\n", " ");
   StringReplace(d, "\r", " ");
   StringReplace(d, "|", " ");
   StringReplace(d, "\"", "'");
   int rep = 1;
   while(rep > 0) rep = StringReplace(d, "  ", " ");
   StringTrimLeft(d);
   StringTrimRight(d);
   
   int room = maxLen - StringLen(prefix);
   if(room < 1) return StringSubstr(prefix, 0, maxLen);
   if(StringLen(d) > room)
   {
      if(room <= 3) d = "";
      else d = StringSubstr(d, 0, room - 3) + "...";
   }
   return prefix + d;
}

//+------------------------------------------------------------------+
//| Risk helpers                                                     |
//+------------------------------------------------------------------+
double OrderRiskDollars()
{
   double dpp = DollarsPerPriceUnit();
   if(dpp <= 0) return MathMax(0.5, InpStopLossDollars);
   double risk = InpStopLossDollars;
   if(InpOrderSLFromDynamicTargets && g_Targets.stopLossDollars > 0)
      risk = g_Targets.stopLossDollars;
   if(risk <= 0) risk = InpStopLossDollars;
   risk = MathMin(risk, InpMaxRiskDollars);
   if(InpCapOrderRiskToMinProfit && g_Targets.minProfit > 0)
   {
      double cap = g_Targets.minProfit * InpMaxRiskVsMinTargetMult;
      if(cap > 0) risk = MathMin(risk, cap);
   }
   if(risk < 0.01) risk = InpStopLossDollars;
   return risk;
}

//+------------------------------------------------------------------+
//| LOT SIZE BY VOLATILITY REGIME                                    |
//| Scales InpLotSize by per-regime multipliers, then clamps to      |
//| broker volume min/max/step. Guarantees a tradable volume.        |
//+------------------------------------------------------------------+
double LotForRegime()
{
   double mul = 1.0;
   switch(g_VolRegime)
   {
      case VOL_QUIET:     mul = InpLotMulQuiet;     break;
      case VOL_NORMAL:    mul = InpLotMulNormal;    break;
      case VOL_ELEVATED:  mul = InpLotMulElevated;  break;
      case VOL_EXPLOSIVE: mul = InpLotMulExplosive; break;
      default:            mul = 1.0;                break;
   }
   double lot = InpLotSize * mul;
   double vmin  = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double vmax  = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   double vstep = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   if(vstep > 0) lot = MathFloor(lot / vstep + 0.0000001) * vstep;
   if(vmin > 0 && lot < vmin) lot = vmin;
   if(vmax > 0 && lot > vmax) lot = vmax;
   return NormalizeDouble(lot, 2);
}

//+------------------------------------------------------------------+
//| EXECUTE PROPOSAL — opens position with strategy-aware SL         |
//+------------------------------------------------------------------+
bool ExecuteProposal(SetupProposal &p)
{
   if(!p.valid) return false;
   
   double dpp = DollarsPerPriceUnit();
   if(dpp <= 0) { Print("✖ ExecuteProposal: dollarsPerPriceUnit invalid"); return false; }
   
   // Always cap actual SL distance to OrderRiskDollars()
   double allowedRiskUsd = OrderRiskDollars();
   double allowedDist = allowedRiskUsd / dpp;
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double price = p.isBuy ? ask : bid;
   double sl = p.stopLoss;
   double slDist = MathAbs(price - sl);
   if(slDist > allowedDist)
   {
      sl = p.isBuy ? NormalizeDouble(price - allowedDist, g_Digits)
                   : NormalizeDouble(price + allowedDist, g_Digits);
      slDist = allowedDist;
   }
   
   double minStop = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * g_Point;
   if(minStop > 0 && slDist < minStop)
   {
      sl = p.isBuy ? NormalizeDouble(price - minStop, g_Digits)
                   : NormalizeDouble(price + minStop, g_Digits);
      slDist = minStop;
   }
   
   // Volatility-regime-scaled lot, then asymmetric (score-band) multiplier
   double regimeLot = LotForRegime();
   double asymMul   = AsymmetricMultiplier(p.score);
   double tradeLot  = regimeLot * asymMul;
   
   // Snap to broker volume step before computing risk
   double vmin  = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double vmax  = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   double vstep = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   if(vstep > 0) tradeLot = MathFloor(tradeLot / vstep + 0.0000001) * vstep;
   if(vmin > 0 && tradeLot < vmin) tradeLot = vmin;
   if(vmax > 0 && tradeLot > vmax) tradeLot = vmax;
   tradeLot = NormalizeDouble(tradeLot, 2);
   
   // Risk at the planned lot
   double riskPerLotUsd = slDist * (g_TickValue / MathMax(g_TickSize, 0.0000001));
   double riskUsdAtSl   = riskPerLotUsd * tradeLot;
   double capUsd        = CurrentRiskCapUsd();
   bool   shrunk        = false;
   double prevLot       = tradeLot;
   
   if(riskUsdAtSl > capUsd * 1.05)
   {
      // Auto-shrink lot to fit cap (use 0.98×cap so we land safely under tolerance)
      double targetLot = (capUsd * 0.98) / MathMax(riskPerLotUsd, 0.0000001);
      if(vstep > 0) targetLot = MathFloor(targetLot / vstep + 0.0000001) * vstep;
      targetLot = NormalizeDouble(targetLot, 2);
      
      if(vmin > 0 && targetLot < vmin)
      {
         // Even broker-min lot is over the cap — abort
         Print("✖ ExecuteProposal: risk $", DoubleToString(riskPerLotUsd * vmin, 2),
               " > cap $", DoubleToString(capUsd, 2), " even at broker min lot ", DoubleToString(vmin, 2));
         g_LastRankerVerdict = "REJECTED " + p.name + " (risk too high even at min lot)";
         return false;
      }
      tradeLot = targetLot;
      riskUsdAtSl = riskPerLotUsd * tradeLot;
      shrunk = true;
   }
   
   string ordCom = BuildEntryOrderComment(p.tag, (int)MathRound(p.score), p.isBuy, p.factors);
   
   Print("═══════════════════════════════════════════════════════════════");
   Print("⚡ HFT FIRE: ", p.name, " score=", DoubleToString(p.score, 1), " (raw=", DoubleToString(p.rawScore, 1),
         " ctx=", DoubleToString(p.ctxScore, 1), ")");
   Print("   ", (p.isBuy ? "BUY" : "SELL"), " ", DoubleToString(tradeLot, 2), " lots @ ", DoubleToString(price, g_Digits),
         " | SL ", DoubleToString(sl, g_Digits), " | risk ~$", DoubleToString(riskUsdAtSl, 2),
         " (cap $", DoubleToString(capUsd, 2), ")",
         " | T1=", DoubleToString(p.t1, g_Digits), " | R:R=", DoubleToString(p.rrToT1, 2),
         " | regime=", RegimeLabelForAI());
   if(shrunk)
      Print("   ⚖ Lot auto-shrunk: ", DoubleToString(prevLot, 2), " -> ", DoubleToString(tradeLot, 2),
            " (regime=", DoubleToString(regimeLot, 2), " asym=", DoubleToString(asymMul, 2), ")");
   else
      Print("   Sizing: regime=", DoubleToString(regimeLot, 2), " x asym=", DoubleToString(asymMul, 2),
            " = ", DoubleToString(tradeLot, 2));
   Print("   Comment: ", ordCom);
   
   g_LastEntryRequestPrice = price;
   g_LastSlippagePts = 0;
   bool ok = false;
   
   if(p.isBuy)
      ok = g_Trade.Buy(tradeLot, _Symbol, price, sl, 0, ordCom);
   else
      ok = g_Trade.Sell(tradeLot, _Symbol, price, sl, 0, ordCom);
   
   if(ok && g_Trade.ResultRetcode() == TRADE_RETCODE_DONE)
   {
      g_Ticket = g_Trade.ResultOrder();
      g_PartialTaken = false;
      g_EntryPrice = price;
      g_MaxProfit = 0;
      g_EntryTime = TimeCurrent();
      g_TodayTrades++;
      g_OpenSetupKind  = p.kind;
      g_OpenSetupName  = p.name;
      g_OpenSetupScore = p.score;
      g_LogSnapshotScore = p.score;
      g_LogSnapshotRegime = RegimeLabelForAI();
      g_LogSnapshotSetup  = p.name;
      g_LogOpenPositionId = 0;
      for(int pi = PositionsTotal() - 1; pi >= 0; pi--)
      {
         ulong ptk = PositionGetTicket(pi);
         if(ptk == 0) continue;
         if(!PositionSelectByTicket(ptk)) continue;
         if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;
         if((ulong)PositionGetInteger(POSITION_MAGIC) != MAGIC_NUMBER) continue;
         g_LogOpenPositionId = (ulong)PositionGetInteger(POSITION_IDENTIFIER);
         break;
      }
      Print("   ✅ EXECUTED ticket=", g_Ticket);
      ScalperLog("ENTRY_FILL", (p.isBuy ? "BUY" : "SELL"), p.name,
                 "score=" + DoubleToString(p.score, 2) + ";rr=" + DoubleToString(p.rrToT1, 2) +
                 ";riskUsd=" + DoubleToString(riskUsdAtSl, 2) +
                 ";order=" + IntegerToString((long)g_Ticket) +
                 ";posId=" + IntegerToString((long)g_LogOpenPositionId));
      Print("═══════════════════════════════════════════════════════════════");
      if(InpShowPanel) { UpdatePanel(); ForceChartRedraw(); }  // v1.11: instant panel refresh on entry
      return true;
   }
   
   Print("   ❌ FAILED: ret=", g_Trade.ResultRetcode(), " ", g_Trade.ResultComment());
   Print("═══════════════════════════════════════════════════════════════");
   return false;
}

//+------------------------------------------------------------------+
//| RANKER — collect, sort, gate, fire                               |
//+------------------------------------------------------------------+
// Returns true if this setup kind is currently in suspension cooldown
bool IsKindSuspended(SETUP_KIND k)
{
   int idx = (int)k;
   if(idx < 0 || idx >= 12) return false;
   return (g_KindSuspendUntil[idx] > TimeCurrent());
}

// Append a proposal if valid AND its setup kind is not currently suspended
void TryAppendProposal(SetupProposal &q, SetupProposal &arr[], int &n)
{
   if(!q.valid) return;
   if(IsKindSuspended(q.kind)) return;
   if(n >= MAX_PROPOSALS) return;
   arr[n++] = q;
}

int CollectProposals(SetupProposal &arr[])
{
   int n = 0;
   ArrayResize(arr, MAX_PROPOSALS);
   SetupProposal q;
   q = Setup_Spike();             TryAppendProposal(q, arr, n);
   q = Setup_BullBearFlag();      TryAppendProposal(q, arr, n);
   q = Setup_ORB();               TryAppendProposal(q, arr, n);
   q = Setup_VWAPReclaim();       TryAppendProposal(q, arr, n);
   q = Setup_VWAPReject();        TryAppendProposal(q, arr, n);
   q = Setup_ABCD();              TryAppendProposal(q, arr, n);
   q = Setup_LevelBounce();       TryAppendProposal(q, arr, n);
   q = Setup_LevelBreak();        TryAppendProposal(q, arr, n);
   q = Setup_EMAPullback();       TryAppendProposal(q, arr, n);
   q = Setup_NewsContinuation();  TryAppendProposal(q, arr, n);
   q = Setup_Fade();              TryAppendProposal(q, arr, n);
   ArrayResize(arr, n);
   return n;
}

void SortProposalsDesc(SetupProposal &arr[])
{
   int n = ArraySize(arr);
   for(int i = 1; i < n; i++)
   {
      SetupProposal key = arr[i];
      int j = i - 1;
      while(j >= 0 && arr[j].score < key.score)
      {
         arr[j + 1] = arr[j];
         j--;
      }
      arr[j + 1] = key;
   }
}

void StoreTopForPanel(SetupProposal &arr[])
{
   g_LastTopCount = 0;
   int n = ArraySize(arr);
   int k = MathMin(3, n);
   for(int i = 0; i < k; i++)
   {
      g_LastTop[i] = arr[i];
      g_LastTopCount++;
      // Pre-compute block reason for each top so the panel can show 'why' a
      // proposal would not fire even though it appears in the top 3
      string skip = "";
      if(arr[i].score < InpMinSetupScore)
         skip = "below threshold (" + DoubleToString(arr[i].score, 1) + " < " +
                DoubleToString(InpMinSetupScore, 1) + ")";
      else if(!PassesGlobalHardGates(arr[i], skip))
         ; // skip already populated by PassesGlobalHardGates
      else
         skip = "PASS";
      g_LastBlockReasonForTop[i] = skip;
   }
   for(int i = k; i < 3; i++) g_LastBlockReasonForTop[i] = "";
}

void RunRanker()
{
   SetupProposal arr[];
   int n = CollectProposals(arr);
   g_LastProposalCount = n;
   if(n == 0)
   {
      g_LastTopCount = 0;
      g_LastRankerVerdict = "NO_PROPOSAL";
      for(int i = 0; i < 3; i++) g_LastBlockReasonForTop[i] = "";
      if(InpVerboseSkips) Print("⏭ Ranker: no setup proposals this cycle");
      return;
   }
   SortProposalsDesc(arr);
   StoreTopForPanel(arr);
   
   // Log top-N to journal
   string topLine = "";
   int logN = MathMin(InpRankerLogTopN, n);
   for(int i = 0; i < logN; i++)
   {
      topLine += arr[i].name + "=" + DoubleToString(arr[i].score, 1) +
                 "[rr=" + DoubleToString(arr[i].rrToT1, 2) + "] ";
   }
   ScalperLog("RANKER", IntegerToString(n) + " proposals", "", topLine);
   if(InpVerboseSkips)
      Print("🎯 Ranker: ", n, " proposal(s). Top: ", topLine);
   
   // Fire the top proposal if it clears the score threshold + gates
   if(arr[0].score < InpMinSetupScore)
   {
      g_LastRankerVerdict = "BELOW_THRESH (" + DoubleToString(arr[0].score, 1) + " < " +
                            DoubleToString(InpMinSetupScore, 1) + ")";
      if(InpVerboseSkips)
         Print("⏭ Top score ", DoubleToString(arr[0].score, 1), " < min ",
               DoubleToString(InpMinSetupScore, 1));
      return;
   }
   
   // Try fire — if top fails gates, try next (limit 2)
   int tryUpTo = MathMin(2, n);
   for(int i = 0; i < tryUpTo; i++)
   {
      string skip = "";
      if(arr[i].score < InpMinSetupScore) break;
      if(PassesGlobalHardGates(arr[i], skip))
      {
         g_LastRankerVerdict = "FIRED " + arr[i].name;
         ExecuteProposal(arr[i]);
         return;
      }
      else
      {
         g_LastRankerVerdict = "GATED " + arr[i].name + " (" + skip + ")";
         if(InpVerboseSkips)
            Print("⛔ ", arr[i].name, " score=", DoubleToString(arr[i].score, 1), " blocked: ", skip);
      }
   }
}

//+------------------------------------------------------------------+
//| POSITION MANAGEMENT — check, autoprotect, trail, partial         |
//+------------------------------------------------------------------+
bool CheckPosition()
{
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket <= 0) continue;
      if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC) != MAGIC_NUMBER) continue;
      g_Ticket = ticket;
      g_CurrentProfit = PositionGetDouble(POSITION_PROFIT);
      g_PosType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
      if(g_CurrentProfit > g_MaxProfit) g_MaxProfit = g_CurrentProfit;
      return true;
   }
   if(g_Ticket > 0)
   {
      g_Ticket = 0;
      g_PartialTaken = false;
      g_EntryPrice = 0;
      g_MaxProfit = 0;
      g_CurrentProfit = 0;
      g_EntryTime = 0;
      g_OpenSetupKind = SETUP_NONE;
      g_OpenSetupName = "";
      g_OpenSetupScore = 0;
   }
   return false;
}

// v1.11: Close-tag FIFO queue helpers.
// TradeCloseWithComment pushes; OnTradeTransaction pops. Handles partial-close
// races where two deals materialise in the same tick (e.g. partial then runner).
void PushCloseTag(const string tag)
{
   if(StringLen(tag) == 0) return;
   g_CloseTagQueue[g_CloseTagHead % CLOSE_TAG_QUEUE_SIZE] = tag;
   g_CloseTagHead++;
   // also keep g_LastCloseTag in sync for any legacy reader
   g_LastCloseTag = tag;
}

string PopCloseTag()
{
   if(g_CloseTagTail >= g_CloseTagHead) return "";  // queue empty
   string t = g_CloseTagQueue[g_CloseTagTail % CLOSE_TAG_QUEUE_SIZE];
   g_CloseTagQueue[g_CloseTagTail % CLOSE_TAG_QUEUE_SIZE] = "";
   g_CloseTagTail++;
   return t;
}

// Bump the in-memory exit-reason tally for today (used by panel)
void BumpExitTally(const string tag)
{
   if(StringLen(tag) == 0) return;
   for(int i = 0; i < g_ExitTallyCount; i++)
   {
      if(g_ExitTallyTags[i] == tag) { g_ExitTallyHits[i]++; return; }
   }
   if(g_ExitTallyCount < 12)
   {
      g_ExitTallyTags[g_ExitTallyCount] = tag;
      g_ExitTallyHits[g_ExitTallyCount] = 1;
      g_ExitTallyCount++;
   }
}

// Derive a 3-4 char exit-reason tag from the close-tag queue (FIFO order, so
// partials and runners each get their own tag) or by parsing the broker's
// deal comment for SL/TP markers.
string DeriveExitReasonTag(const string brokerComment)
{
   string queued = PopCloseTag();
   if(StringLen(queued) > 0) return queued;
   string c = brokerComment;
   StringToLower(c);
   if(StringFind(c, "[sl") >= 0 || StringFind(c, "stop loss") >= 0 || StringFind(c, "[so]") >= 0) return "SL";
   if(StringFind(c, "[tp") >= 0 || StringFind(c, "take profit") >= 0) return "TP";
   if(StringFind(c, "mprt") >= 0) return "MPRT";
   if(StringLen(brokerComment) == 0) return "MAN";
   return "OTH";
}

bool TradeCloseWithComment(ulong ticket, double volume, const string tag, const string detail)
{
   // v1.11: Push our close-tag onto the FIFO queue so OnTradeTransaction can
   // pop it back even when partials + full close materialise in the same tick.
   PushCloseTag(tag);
   if(!PositionSelectByTicket(ticket)) return false;
   string sym = PositionGetString(POSITION_SYMBOL);
   double posVol = PositionGetDouble(POSITION_VOLUME);
   if(volume <= 0 || volume > posVol + 1e-8) volume = posVol;
   double vmin = SymbolInfoDouble(sym, SYMBOL_VOLUME_MIN);
   double vstep = SymbolInfoDouble(sym, SYMBOL_VOLUME_STEP);
   if(vstep > 0) volume = MathFloor(volume / vstep + 1e-8) * vstep;
   volume = NormalizeDouble(volume, 8);
   if(volume < vmin) volume = vmin;
   if(volume > posVol) volume = posVol;
   
   ENUM_POSITION_TYPE ptype = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
   ulong posMagic = (ulong)PositionGetInteger(POSITION_MAGIC);
   
   MqlTradeRequest req; MqlTradeResult res;
   ZeroMemory(req); ZeroMemory(res);
   req.action = TRADE_ACTION_DEAL;
   req.position = ticket;
   req.symbol = sym;
   req.volume = volume;
   req.deviation = 10;
   req.magic = posMagic;
   req.comment = BuildEntryOrderComment(tag, 0, ptype == POSITION_TYPE_BUY, detail);
   
   long fm = SymbolInfoInteger(sym, SYMBOL_FILLING_MODE);
   if((fm & SYMBOL_FILLING_FOK) == SYMBOL_FILLING_FOK) req.type_filling = ORDER_FILLING_FOK;
   else if((fm & SYMBOL_FILLING_IOC) == SYMBOL_FILLING_IOC) req.type_filling = ORDER_FILLING_IOC;
   else req.type_filling = ORDER_FILLING_RETURN;
   
   int maxTry = InpCloseSendMaxRetries;  if(maxTry < 1) maxTry = 1;
   int delayMs = InpCloseSendRetryDelayMs; if(delayMs < 0) delayMs = 0;
   
   for(int attempt = 0; attempt < maxTry; attempt++)
   {
      if(attempt > 0) Sleep(delayMs);
      if(!PositionSelectByTicket(ticket)) return false;
      posVol = PositionGetDouble(POSITION_VOLUME);
      if(volume > posVol + 1e-8) volume = posVol;
      if(volume < vmin - 1e-12) return false;
      ZeroMemory(res);
      if(ptype == POSITION_TYPE_BUY)
      {
         req.type = ORDER_TYPE_SELL;
         req.price = SymbolInfoDouble(sym, SYMBOL_BID);
      }
      else
      {
         req.type = ORDER_TYPE_BUY;
         req.price = SymbolInfoDouble(sym, SYMBOL_ASK);
      }
      ResetLastError();
      bool sent = OrderSend(req, res);
      uint err = GetLastError();
      if(sent && (res.retcode == TRADE_RETCODE_DONE || res.retcode == TRADE_RETCODE_PLACED))
         return true;
      Print("⚠️ TradeCloseWithComment attempt ", attempt + 1, "/", maxTry,
            " ret ", res.retcode, " err ", err, " ", res.comment);
   }
   return false;
}

void ClosePartial()
{
   if(!PositionSelectByTicket(g_Ticket)) return;
   double volume = PositionGetDouble(POSITION_VOLUME);
   double partVol = NormalizeDouble(volume * (InpPartialPercent / 100.0), 2);
   double minLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double step = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   if(partVol < minLot) partVol = minLot;
   if(step > 0) partVol = MathFloor(partVol / step) * step;
   if(partVol >= volume) return;
   
   bool partOk = TradeCloseWithComment(g_Ticket, partVol, "MPRT", "partial " + DoubleToString(InpPartialPercent, 0) + "%");
   if(!partOk) partOk = g_Trade.PositionClosePartial(g_Ticket, partVol);
   if(partOk)
   {
      g_PartialTaken = true;
      Print("✅ Partial close ", partVol, " lots @ $", DoubleToString(g_CurrentProfit, 2));
      Sleep(100);
      if(PositionSelectByTicket(g_Ticket))
      {
         double sl = PositionGetDouble(POSITION_SL);
         double newSL = g_EntryPrice + (g_PosType == POSITION_TYPE_BUY ? 1 : -1) * 5 * g_Point;
         newSL = NormalizeDouble(newSL, g_Digits);
         if((g_PosType == POSITION_TYPE_BUY && newSL > sl) ||
            (g_PosType == POSITION_TYPE_SELL && newSL < sl))
         {
            g_Trade.PositionModify(g_Ticket, newSL, 0);
            Print("🔒 SL → BE+");
         }
      }
   }
}

void TrailStop()
{
   if(!PositionSelectByTicket(g_Ticket)) return;
   if(g_MaxProfit <= 0) return;
   double sl = PositionGetDouble(POSITION_SL);
   double minStop = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * g_Point;
   if(minStop <= 0) minStop = 10 * g_Point;
   
   double protectPct = 0.70;  // lock 70% of MAX
   double lockProfit = g_MaxProfit * protectPct;
   double lots = PositionGetDouble(POSITION_VOLUME);
   double tickVal = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   if(lots <= 0 || tickVal <= 0 || tickSize <= 0) return;
   double pricePerDollar = tickSize / (lots * tickVal);
   double lockDistance = lockProfit * pricePerDollar;
   
   double newSL = 0;
   if(g_PosType == POSITION_TYPE_BUY)
   {
      double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
      newSL = NormalizeDouble(g_EntryPrice + lockDistance, g_Digits);
      if(newSL > g_EntryPrice && newSL > sl && newSL < bid - minStop)
      {
         g_Trade.PositionModify(g_Ticket, newSL, 0);
         Print("📈 TRAIL lock 70% of $", DoubleToString(g_MaxProfit, 2), " → SL ", newSL);
      }
   }
   else
   {
      double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
      newSL = NormalizeDouble(g_EntryPrice - lockDistance, g_Digits);
      if(newSL < g_EntryPrice && (newSL < sl || sl == 0) && newSL > ask + minStop)
      {
         g_Trade.PositionModify(g_Ticket, newSL, 0);
         Print("📉 TRAIL lock 70% of $", DoubleToString(g_MaxProfit, 2), " → SL ", newSL);
      }
   }
}

bool AP_IsRunnerM5()
{
   bool aligned = (g_PosType == POSITION_TYPE_BUY && g_M5Trend > 0) ||
                  (g_PosType == POSITION_TYPE_SELL && g_M5Trend < 0);
   return (aligned && g_M5MomentumOK && g_M5TrendPct >= InpAPRunnerM5StrengthPct);
}

//+------------------------------------------------------------------+
//| AUTO PROTECT — every-tick rules (NORED, retrace, lock, gave-back)|
//+------------------------------------------------------------------+
void AutoProtectProfit()
{
   if(g_Ticket == 0) return;
   if(!PositionSelectByTicket(g_Ticket)) return;
   
   double sl = PositionGetDouble(POSITION_SL);
   double minStop = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * g_Point;
   if(minStop <= 0) minStop = 10 * g_Point;
   
   // Actual lot of THIS position (may differ from InpLotSize due to vol-regime scaling).
   // Used so lock-distance formulas convert $ → price correctly for this trade.
   double posLot = PositionGetDouble(POSITION_VOLUME);
   if(posLot <= 0) posLot = InpLotSize;
   
   bool inRun = AP_IsRunnerM5();
   int duration = (int)(TimeCurrent() - g_EntryTime);
   
   // Hard time-stop (optional, off by default) — true HFT punishment for idle trades
   if(InpHardTimeStopSec > 0 && duration > InpHardTimeStopSec &&
      MathAbs(g_CurrentProfit) <= InpHardTimeStopMinAbsUsd && g_MaxProfit < InpAPGreenArmUsd)
   {
      Print("⏱ A-TIME: idle ", duration, "s |P/L|=$", DoubleToString(MathAbs(g_CurrentProfit), 2), " — close");
      if(!TradeCloseWithComment(g_Ticket, 0, "ATIM", "hard time stop"))
         g_Trade.PositionClose(g_Ticket);
      return;
   }
   
   // RULE -2: NORED — was green, now red
   if(InpAPNoRedAfterGreen && g_MaxProfit >= InpAPGreenArmUsd && g_CurrentProfit < 0.0)
   {
      Print("🛑 A-NORED: max $", DoubleToString(g_MaxProfit, 2), " | now $", DoubleToString(g_CurrentProfit, 2));
      if(!TradeCloseWithComment(g_Ticket, 0, "ANRD", "no red after green"))
         g_Trade.PositionClose(g_Ticket);
      return;
   }
   
   // RULE -1: Retrace from peak (run + chop)
   if(InpAPUniversalRetrace && g_MaxProfit >= InpAPGreenArmUsd && g_CurrentProfit > 0.0)
   {
      double retraceL = inRun ? InpAPRetraceRunUsd : InpAPRetraceChopUsd;
      if(retraceL < 0.05) retraceL = 0.20;
      // v1.11: adaptive widening — if today's ARTR exits exceed the trigger,
      // the retrace threshold is too tight for current conditions. Widen by
      // InpAdaptiveRetraceMul to give runners more breathing room. Same-day
      // evidence loop, so it self-corrects when the regime calms down.
      if(InpAdaptiveRetrace)
      {
         int artrToday = 0;
         for(int eq = 0; eq < g_ExitTallyCount; eq++)
            if(g_ExitTallyTags[eq] == "ARTR") { artrToday = g_ExitTallyHits[eq]; break; }
         if(artrToday >= InpAdaptiveRetraceTrigger)
            retraceL *= InpAdaptiveRetraceMul;
      }
      if(g_CurrentProfit < g_MaxProfit - retraceL)
      {
         Print("✅ A-RETR: take $", DoubleToString(g_CurrentProfit, 2), " (max $", DoubleToString(g_MaxProfit, 2),
               (inRun ? " RUN" : " chop"), " thresh=$", DoubleToString(retraceL, 2), ")");
         if(!TradeCloseWithComment(g_Ticket, 0, "ARTR", "retrace from peak"))
            g_Trade.PositionClose(g_Ticket);
         return;
      }
   }
   
   // Auto-bank if not in a run
   if(InpAPAutoBankScalp && !inRun && g_CurrentProfit >= InpAPScalpBankUsd)
   {
      Print("✅ A-BANK: not running — $", DoubleToString(g_CurrentProfit, 2));
      if(!TradeCloseWithComment(g_Ticket, 0, "ABNK", "not runner bank"))
         g_Trade.PositionClose(g_Ticket);
      return;
   }
   
   // RULE 1: Breakeven (in chop, earlier)
   if(!g_PartialTaken && (g_MaxProfit >= g_Targets.breakEvenTrigger || (!inRun && g_MaxProfit >= 0.38)))
   {
      double beStop = 0;
      if(g_PosType == POSITION_TYPE_BUY)
      {
         beStop = NormalizeDouble(g_EntryPrice + 5 * g_Point, g_Digits);
         double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
         if(sl < g_EntryPrice && beStop < bid - minStop)
         {
            if(g_Trade.PositionModify(g_Ticket, beStop, 0)) Print("🔒 BE → ", beStop);
         }
      }
      else
      {
         beStop = NormalizeDouble(g_EntryPrice - 5 * g_Point, g_Digits);
         double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
         if((sl > g_EntryPrice || sl == 0) && beStop > ask + minStop)
         {
            if(g_Trade.PositionModify(g_Ticket, beStop, 0)) Print("🔒 BE → ", beStop);
         }
      }
   }
   
   // RULE 1.5: Micro-lock ladder
   if(InpAPMicroTrail && g_TickValue > 0 && posLot > 0 && g_TickSize > 0 &&
      g_MaxProfit < g_Targets.minProfit * 0.95)
   {
      double mLock = 0;
      if(g_MaxProfit >= 1.10)      mLock = inRun ? 0.48 : 0.58;
      else if(g_MaxProfit >= 0.88) mLock = inRun ? 0.34 : 0.45;
      else if(g_MaxProfit >= 0.72) mLock = inRun ? 0.36 : 0.48;
      else if(g_MaxProfit >= 0.64) mLock = inRun ? 0.22 : 0.32;
      else if(g_MaxProfit >= InpAPMicroLockFromUsd) mLock = inRun ? 0.12 : 0.18;
      if(mLock > 0.05 && mLock < g_MaxProfit - 0.04)
      {
         double lockDistM = mLock / (g_TickValue * posLot / g_TickSize);
         double nsl = 0;
         if(g_PosType == POSITION_TYPE_BUY)
         {
            nsl = NormalizeDouble(g_EntryPrice + lockDistM, g_Digits);
            double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
            if(nsl > sl && nsl < bid - minStop)
            {
               if(g_Trade.PositionModify(g_Ticket, nsl, 0))
                  Print("💰 MICRO-LOCK $", DoubleToString(mLock, 2), (inRun ? " run" : " chop"), " → ", nsl);
            }
         }
         else
         {
            nsl = NormalizeDouble(g_EntryPrice - lockDistM, g_Digits);
            double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
            if((nsl < sl || sl == 0) && nsl > ask + minStop)
            {
               if(g_Trade.PositionModify(g_Ticket, nsl, 0))
                  Print("💰 MICRO-LOCK $", DoubleToString(mLock, 2), (inRun ? " run" : " chop"), " → ", nsl);
            }
         }
      }
   }
   
   // RULE 2: Profit-lock ladder
   double lockProfit = 0;
   if(g_MaxProfit >= g_Targets.greatProfit) lockProfit = g_Targets.goodProfit;
   else if(g_MaxProfit >= g_Targets.goodProfit) lockProfit = g_Targets.minProfit;
   else if(g_MaxProfit >= g_Targets.minProfit) lockProfit = g_Targets.minProfit * 0.5;
   if(lockProfit > 0 && g_TickValue > 0 && posLot > 0 && g_TickSize > 0)
   {
      double lockDist = lockProfit / (g_TickValue * posLot / g_TickSize);
      double newSL = 0;
      if(g_PosType == POSITION_TYPE_BUY)
      {
         newSL = NormalizeDouble(g_EntryPrice + lockDist, g_Digits);
         double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
         if(newSL > sl && newSL < bid - minStop)
         {
            if(g_Trade.PositionModify(g_Ticket, newSL, 0))
               Print("💰 LADDER lock $", DoubleToString(lockProfit, 2), " → ", newSL);
         }
      }
      else
      {
         newSL = NormalizeDouble(g_EntryPrice - lockDist, g_Digits);
         double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
         if((newSL < sl || sl == 0) && newSL > ask + minStop)
         {
            if(g_Trade.PositionModify(g_Ticket, newSL, 0))
               Print("💰 LADDER lock $", DoubleToString(lockProfit, 2), " → ", newSL);
         }
      }
   }
   
   // RULE 3: Gave-back protection
   if(g_MaxProfit >= 4.00 && g_CurrentProfit < g_MaxProfit - 2.00 && g_CurrentProfit > 0)
   {
      Print("⚠ GAVE BACK $2+ from $4+ peak → bank $", DoubleToString(g_CurrentProfit, 2));
      if(!TradeCloseWithComment(g_Ticket, 0, "AGVE", "gave back profit"))
         g_Trade.PositionClose(g_Ticket);
      return;
   }
   double redLine = (InpAPWinnerGoneRedUsd > 0 ? -InpAPWinnerGoneRedUsd : -0.30);
   if((g_MaxProfit >= 2.50 && g_CurrentProfit <= redLine) ||
      (!inRun && g_MaxProfit >= 0.50 && g_MaxProfit < 2.3 && g_CurrentProfit <= redLine))
   {
      Print("🛑 WINNER→RED max $", DoubleToString(g_MaxProfit, 2), " now $", DoubleToString(g_CurrentProfit, 2));
      if(!TradeCloseWithComment(g_Ticket, 0, "AEMR", "winner to loser"))
         g_Trade.PositionClose(g_Ticket);
      return;
   }
   if(g_MaxProfit >= 2.00 && g_CurrentProfit <= -0.75)
   {
      Print("🛑 GOOD START GONE BAD: $", DoubleToString(g_CurrentProfit, 2));
      if(!TradeCloseWithComment(g_Ticket, 0, "ACUT", "good start gone bad"))
         g_Trade.PositionClose(g_Ticket);
      return;
   }
}

//+------------------------------------------------------------------+
//| OnTradeTransaction — track entry slippage + exit P/L stats       |
//+------------------------------------------------------------------+
void OnTradeTransaction(const MqlTradeTransaction& trans,
                        const MqlTradeRequest& request,
                        const MqlTradeResult& result)
{
   if(trans.type != TRADE_TRANSACTION_DEAL_ADD) return;
   ulong deal = trans.deal;
   if(deal == 0 || !HistoryDealSelect(deal)) return;
   if(HistoryDealGetString(deal, DEAL_SYMBOL) != _Symbol) return;
   if(HistoryDealGetInteger(deal, DEAL_MAGIC) != MAGIC_NUMBER) return;
   
   ENUM_DEAL_ENTRY dEntry = (ENUM_DEAL_ENTRY)HistoryDealGetInteger(deal, DEAL_ENTRY);
   if(dEntry == DEAL_ENTRY_IN)
   {
      double fillPrice = HistoryDealGetDouble(deal, DEAL_PRICE);
      if(g_LastEntryRequestPrice > 0 && g_Point > 0)
      {
         ENUM_DEAL_TYPE dType = (ENUM_DEAL_TYPE)HistoryDealGetInteger(deal, DEAL_TYPE);
         if(dType == DEAL_TYPE_BUY)
            g_LastSlippagePts = (fillPrice - g_LastEntryRequestPrice) / g_Point;
         else if(dType == DEAL_TYPE_SELL)
            g_LastSlippagePts = (g_LastEntryRequestPrice - fillPrice) / g_Point;
         Print("📌 Slippage: ", DoubleToString(g_LastSlippagePts, 1), " pts");
      }
      return;
   }
   if(dEntry != DEAL_ENTRY_OUT) return;
   
   double profit = HistoryDealGetDouble(deal, DEAL_PROFIT);
   profit += HistoryDealGetDouble(deal, DEAL_COMMISSION);
   profit += HistoryDealGetDouble(deal, DEAL_SWAP);
   
   g_DailyPnL += profit;
   g_LastTradeResult = profit;
   g_LastTradeCloseTime = TimeCurrent();
   
   ulong pid = (ulong)HistoryDealGetInteger(deal, DEAL_POSITION_ID);
   string dc = HistoryDealGetString(deal, DEAL_COMMENT);
   string evt = (profit >= 0 ? "EXIT_WIN" : "EXIT_LOSS");
   string exitTag = DeriveExitReasonTag(dc);
   BumpExitTally(exitTag);
   if(InpShowPanel) { UpdatePanel(); ForceChartRedraw(); }  // v1.11: instant panel refresh on close
   string snap = "open_reg=" + g_LogSnapshotRegime + ";open_setup=" + g_LogSnapshotSetup +
                 ";open_score=" + DoubleToString(g_LogSnapshotScore, 2) +
                 ";exit_reason=" + exitTag;
   ScalperLog(evt, DoubleToString(profit, 2), IntegerToString((long)pid), ScalperEscapeField(dc) + " | " + snap);
   
   for(int i = 4; i > 0; i--) g_RecentResults[i] = g_RecentResults[i-1];
   g_RecentResults[0] = profit;
   if(g_RecentResultCount < 5) g_RecentResultCount++;
   
   g_RecentWins = 0; g_RecentLosses = 0;
   for(int i = 0; i < g_RecentResultCount; i++)
   {
      if(g_RecentResults[i] >= 0) g_RecentWins++;
      else g_RecentLosses++;
   }
   
   // Per-setup-kind suspension tracking + win/loss running averages
   int kindIdx = (int)g_OpenSetupKind;
   if(kindIdx >= 0 && kindIdx < 12)
   {
      if(profit < 0)
      {
         g_KindLossStreak[kindIdx]++;
         if(InpSuspendAfter2Losses && g_KindLossStreak[kindIdx] >= 2)
         {
            g_KindSuspendUntil[kindIdx] = TimeCurrent() + InpSuspendDurationMin * 60;
            g_KindLossStreak[kindIdx] = 0;
            Print("⏸ SUSPEND ", g_OpenSetupName, " for ", InpSuspendDurationMin, "min (2 consec losses on this kind)");
         }
      }
      else
      {
         g_KindLossStreak[kindIdx] = 0;
      }
   }
   
   if(profit < 0)
   {
      g_ConsecLosses++;
      g_LastLossTime = TimeCurrent();
      g_TodayLossSum += profit;
      g_TodayLossCnt++;
      Print("❌ LOSS $", DoubleToString(profit, 2), " | day=$", DoubleToString(g_DailyPnL, 2),
            " | consec=", g_ConsecLosses, " | recent ", g_RecentWins, "/", g_RecentResultCount);
   }
   else
   {
      g_ConsecLosses = 0;
      g_TodayWins++;
      g_TodayWinSum += profit;
      Print("✅ WIN $", DoubleToString(profit, 2), " | day=$", DoubleToString(g_DailyPnL, 2),
            " | today ", g_TodayWins, "/", g_TodayTrades);
   }
}

int CountPositions()
{
   int count = 0;
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket > 0 && PositionGetString(POSITION_SYMBOL) == _Symbol &&
         PositionGetInteger(POSITION_MAGIC) == MAGIC_NUMBER)
         count++;
   }
   return count;
}

void CheckNewDay()
{
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   datetime today = StringToTime(IntegerToString(dt.year) + "." +
                                 IntegerToString(dt.mon) + "." +
                                 IntegerToString(dt.day));
   if(today != g_DayStart)
   {
      Print("📅 NEW DAY");
      if(g_DayStart > 0)
         Print("   Yesterday: ", g_TodayWins, "/", g_TodayTrades, " | P/L $", DoubleToString(g_DailyPnL, 2));
      g_DailyPnL = 0;
      g_ConsecLosses = 0;
      g_Halted = false;
      g_DayStart = today;
      g_TodayTrades = 0;
      g_TodayWins = 0;
      g_TodayWinSum  = 0;
      g_TodayLossSum = 0;
      g_TodayLossCnt = 0;
      g_RecentResultCount = 0;
      g_RecentWins = 0;
      g_RecentLosses = 0;
      g_LastTradeCloseTime = 0;
      g_LastTradeResult = 0;
      g_ExitTallyCount = 0;
      for(int i = 0; i < 5; i++) g_RecentResults[i] = 0;
      for(int i = 0; i < 12; i++) { g_ExitTallyTags[i] = ""; g_ExitTallyHits[i] = 0; }
   }
}

//+------------------------------------------------------------------+
//| Startup checks                                                   |
//+------------------------------------------------------------------+
void RunStartupChecks()
{
   Print("");
   Print("📋 STARTUP CHECKS");
   Print("   Symbol=", _Symbol, " digits=", g_Digits, " point=", DoubleToString(g_Point, g_Digits),
         " tickValue=$", DoubleToString(g_TickValue, 4));
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   g_Spread = (ask - bid) / g_Point;
   Print("   Price ", DoubleToString(bid, g_Digits), "/", DoubleToString(ask, g_Digits),
         " spread ", DoubleToString(g_Spread, 0), " pts");
   bool autoTrading = (bool)TerminalInfoInteger(TERMINAL_TRADE_ALLOWED);
   bool eaTrading   = (bool)MQLInfoInteger(MQL_TRADE_ALLOWED);
   Print("   AutoTrade=", (autoTrading ? "Y" : "N"), " EATrade=", (eaTrading ? "Y" : "N"));
   CheckSession();
   Print("   Session=", g_Session, " OK=", (g_SessionOK ? "Y" : "N"));
   RunAnalysis();
   Print("   M15=", (g_M15Bias > 0 ? "BULL" : (g_M15Bias < 0 ? "BEAR" : "FLAT")),
         " H1=", (g_H1Bias > 0 ? "BULL" : (g_H1Bias < 0 ? "BEAR" : "FLAT")),
         " M5=", (g_M5Trend > 0 ? "UP" : (g_M5Trend < 0 ? "DN" : "—")),
         " (", DoubleToString(g_M5TrendPct, 0), "%)");
   Print("   Enabled setups: ",
         (InpEnableSpike ? "SPK " : ""),
         (InpEnableFlag ? "FLG " : ""),
         (InpEnableORB ? "ORB " : ""),
         (InpEnableVWAPReclaim ? "VWR " : ""),
         (InpEnableVWAPReject ? "VWJ " : ""),
         (InpEnableABCD ? "ABC " : ""),
         (InpEnableLevelBounce ? "LBN " : ""),
         (InpEnableLevelBreak ? "LBR " : ""),
         (InpEnableEMAPullback ? "EMA " : ""),
         (InpEnableNewsCont ? "NWC " : ""),
         (InpEnableFade ? "FAD " : ""));
   Print("");
}

//+------------------------------------------------------------------+
//| OnTick                                                           |
//+------------------------------------------------------------------+
void OnTick()
{
   // Always: tick engine + microstructure
   RecordTick();
   AnalyzeMicrostructure();
   CheckNewDay();
   UpdateMarketData();
   
   // Slow refresh (every 5 ticks)
   static int tickCounter = 0;
   tickCounter++;
   if(tickCounter >= 5)
   {
      tickCounter = 0;
      CalculateDynamicTargets();
      DetectKeyLevels();
      CheckEconomicCalendar();
      UpdateMT5ExtendedIntel();
   }
   
   if(InpShowPanel) UpdatePanel();
   
   bool hasPosition = CheckPosition();
   if(hasPosition)
   {
      // 1) Tick-rate auto-protect
      AutoProtectProfit();
      // 2) Periodic trail (high-water 70%)
      if(InpAPTrailIntervalSec > 0 && TimeCurrent() - g_LastTrailCall >= InpAPTrailIntervalSec)
      {
         TrailStop();
         g_LastTrailCall = TimeCurrent();
      }
      return;
   }
   
   // No position → ranker cycle
   ENUM_TIMEFRAMES entryTf = EffectiveEntryBarTF();
   datetime currentBar = iTime(_Symbol, entryTf, 0);
   bool newBar = (currentBar != g_LastEntryBarOpenTime);
   if(newBar)
   {
      g_LastEntryBarOpenTime = currentBar;
      RunAnalysis();
   }
   
   bool intervalElapsed = (TimeCurrent() - g_LastRankerCycle >= InpEntryIntervalSec);
   bool doCycle = intervalElapsed;
   if(InpEntryOnNewBarOnly) doCycle = newBar && intervalElapsed;
   if(!doCycle) return;
   
   // Don't even bother if max positions reached or halted
   if(CountPositions() >= InpMaxPositions) return;
   
   // Refresh everything the setups need
   UpdateMT5ExtendedIntel();
   RunAnalysis();
   UpdateVWAPAndORB();
   CalculateDynamicTargets();
   DetectKeyLevels();
   
   // Throttle news prints (info only)
   if(g_News.inBlackout || g_News.tradingRestricted)
   {
      string sig = g_News.warningMessage + "|" + (g_News.inBlackout ? "B" : "R");
      bool sigChanged = (sig != g_LastNewsThrottleSig);
      int thr = InpNewsNearPrintThrottleSec;
      if(thr < 5) thr = 5;
      if(sigChanged || (TimeCurrent() - g_LastNewsThrottlePrint) >= thr)
      {
         Print("📰 ", g_News.warningMessage);
         g_LastNewsThrottlePrint = TimeCurrent();
         g_LastNewsThrottleSig = sig;
      }
   }
   
   RunRanker();
   g_LastRankerCycle = TimeCurrent();
}

//+------------------------------------------------------------------+
//| PANEL — Draw                                                     |
//+------------------------------------------------------------------+
void CreateRect(string n, int x, int y, int w, int h, color c)
{
   string name = g_Prefix + n;
   ObjectCreate(0, name, OBJ_RECTANGLE_LABEL, 0, 0, 0);
   ObjectSetInteger(0, name, OBJPROP_XDISTANCE, x);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE, y);
   ObjectSetInteger(0, name, OBJPROP_XSIZE, w);
   ObjectSetInteger(0, name, OBJPROP_YSIZE, h);
   ObjectSetInteger(0, name, OBJPROP_BGCOLOR, c);
   ObjectSetInteger(0, name, OBJPROP_BORDER_TYPE, BORDER_FLAT);
   ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
}

void CreateLabel(string n, string t, int x, int y, color c, bool b, int s)
{
   string name = g_Prefix + n;
   ObjectCreate(0, name, OBJ_LABEL, 0, 0, 0);
   ObjectSetInteger(0, name, OBJPROP_XDISTANCE, x);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE, y);
   ObjectSetString(0, name, OBJPROP_TEXT, t);
   ObjectSetInteger(0, name, OBJPROP_COLOR, c);
   ObjectSetInteger(0, name, OBJPROP_FONTSIZE, s);
   ObjectSetString(0, name, OBJPROP_FONT, b ? "Consolas Bold" : "Consolas");
   ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
}

void SetLabel(string n, string t, color c)
{
   string name = g_Prefix + n;
   if(ObjectFind(0, name) >= 0)
   {
      ObjectSetString(0, name, OBJPROP_TEXT, t);
      ObjectSetInteger(0, name, OBJPROP_COLOR, c);
   }
}

// Progress bar — two stacked rectangles. Filled portion uses the first color,
// remaining uses the second. Width pct is clamped to [0..1].
void CreateProgressBar(string n, int x, int y, int w, int h, double pct, color fill, color bg)
{
   if(pct < 0) pct = 0;
   if(pct > 1) pct = 1;
   int fillW = (int)MathRound(w * pct);
   CreateRect(n + "_bg",   x,        y, w,     h, bg);
   CreateRect(n + "_fill", x,        y, fillW, h, fill);
}

void SetProgressBar(string n, int w, double pct, color fill, color bg)
{
   if(pct < 0) pct = 0;
   if(pct > 1) pct = 1;
   int fillW = (int)MathRound(w * pct);
   string fName = g_Prefix + n + "_fill";
   string bName = g_Prefix + n + "_bg";
   if(ObjectFind(0, fName) >= 0)
   {
      ObjectSetInteger(0, fName, OBJPROP_XSIZE, fillW);
      ObjectSetInteger(0, fName, OBJPROP_BGCOLOR, fill);
   }
   if(ObjectFind(0, bName) >= 0)
      ObjectSetInteger(0, bName, OBJPROP_BGCOLOR, bg);
}

void DrawPanel()
{
   // Wipe leftover panel objects (covers v1.00 -> v1.10 layout migration too)
   ObjectsDeleteAll(0, g_Prefix);
   
   int x = InpPanelX;
   int y = InpPanelY;
   int w = InpPanelWidth;
   int innerW = w - 14;
   
   // Total height computed from sections (~840px)
   int hHeader  = 50;
   int hTrade   = 138;
   int hRanker  = 162;
   int hMarket  = 168;
   int hToday   = 122;
   int hSusp    = 80;
   int hRisk    = 70;
   int hFooter  = 26;
   int gap      = 5;
   int totalH   = hHeader + gap + hTrade + gap + hRanker + gap + hMarket + gap +
                  hToday + gap + hSusp + gap + hRisk + gap + hFooter + 10;
   
   CreateRect("Border", x-2, y-2, w+4, totalH+4, C'40,45,55');
   CreateRect("BG",     x,   y,   w,   totalH,   C'18,20,26');
   
   int col1   = x + 12;
   int col2   = x + 138;
   int colVal = x + w - 12;     // right-justified-ish anchor (we still left-anchor; just for layout)
   int ly     = y;
   
   // ─── HEADER ───────────────────────────────────────────────────────
   CreateRect("Header", x, ly, w, hHeader, C'30,38,52');
   CreateLabel("Title", "GOLD-HFT-SCALPER",      col1, ly + 8,  ClrGold, true, 13);
   CreateLabel("Sub",   "v1.10 - 9-strategy ranker", col1, ly + 28, C'160,170,190', false, 9);
   ly += hHeader + gap;
   
   // ─── TRADE ────────────────────────────────────────────────────────
   CreateRect("TradeBox",  x+5, ly,    innerW, hTrade,         C'28,34,46');
   CreateRect("TradeBar",  x+5, ly,    innerW, 22,             C'40,55,75');
   CreateLabel("TradeTtl", "▼ TRADE",  col1,   ly + 4,         ClrAccent, true, 10);
   int tly = ly + 28;
   CreateLabel("L_TStat", "Status:",   col1, tly, C'120,130,150', false, 9);
   CreateLabel("V_TStat", "— NO TRADE", col2, tly, ClrNeutral, true, 10);
   tly += 18;
   CreateLabel("L_TSet",  "Setup:",    col1, tly, C'120,130,150', false, 9);
   CreateLabel("V_TSet",  "—",          col2, tly, ClrAccent, true, 10);
   tly += 18;
   CreateLabel("L_TEnt",  "Entry/SL:", col1, tly, C'120,130,150', false, 9);
   CreateLabel("V_TEnt",  "— / —",      col2, tly, clrWhite, false, 9);
   tly += 16;
   CreateLabel("L_TT1",   "T1/RR:",    col1, tly, C'120,130,150', false, 9);
   CreateLabel("V_TT1",   "— / —",      col2, tly, clrWhite, false, 9);
   tly += 18;
   CreateLabel("L_TPnL",  "P/L:",      col1, tly, C'120,130,150', false, 9);
   CreateLabel("V_TPnL",  "$0.00",     col2, tly, ClrNeutral, true, 13);
   tly += 22;
   CreateLabel("L_TMax",  "Best/Age:", col1, tly, C'120,130,150', false, 9);
   CreateLabel("V_TMax",  "$0.00 / 0s", col2, tly, ClrNeutral, false, 9);
   ly += hTrade + gap;
   
   // ─── RANKER ───────────────────────────────────────────────────────
   CreateRect("RnkBox",   x+5, ly,    innerW, hRanker,         C'28,34,46');
   CreateRect("RnkBar",   x+5, ly,    innerW, 22,              C'52,42,72');
   CreateLabel("RnkTtl",  "▼ RANKER", col1,   ly + 4,          C'180,160,255', true, 10);
   int rly = ly + 28;
   CreateLabel("L_Vrd",   "Verdict:", col1, rly, C'120,130,150', false, 9);
   CreateLabel("V_Vrd",   "—",        col2, rly, ClrNeutral, true, 9);
   rly += 16;
   CreateLabel("L_Cyc",   "Next/Cnt:", col1, rly, C'120,130,150', false, 9);
   CreateLabel("V_Cyc",   "—",         col2, rly, C'180,180,200', false, 9);
   rly += 18;
   // Top 3 — each gets a name+score row and a status (PASS/block reason) row
   for(int i = 0; i < 3; i++)
   {
      string si = IntegerToString(i+1);
      CreateLabel("V_Top"+si,    "—", col1,    rly, C'200,210,225', true, 9);
      rly += 14;
      CreateLabel("V_TopBlk"+si, "",  col1+10, rly, C'140,150,170', false, 8);
      rly += 16;
   }
   ly += hRanker + gap;
   
   // ─── MARKET ───────────────────────────────────────────────────────
   CreateRect("MktBox",   x+5, ly,    innerW, hMarket,         C'28,34,46');
   CreateRect("MktBar",   x+5, ly,    innerW, 22,              C'42,62,52');
   CreateLabel("MktTtl",  "▼ MARKET", col1,   ly + 4,          C'150,220,180', true, 10);
   int mly = ly + 28;
   CreateLabel("L_M15",   "M15/H1:",   col1, mly, C'120,130,150', false, 9);
   CreateLabel("V_M15",   "—",         col2, mly, ClrNeutral, true, 10);
   mly += 18;
   CreateLabel("L_M5T",   "M5/ADX:",   col1, mly, C'120,130,150', false, 9);
   CreateLabel("V_M5T",   "—",         col2, mly, ClrNeutral, true, 10);
   mly += 18;
   CreateLabel("L_Reg",   "Regime:",   col1, mly, C'120,130,150', false, 9);
   CreateLabel("V_Reg",   "—",         col2, mly, ClrNeutral, true, 10);
   mly += 18;
   CreateLabel("L_Sess",  "Session:",  col1, mly, C'120,130,150', false, 9);
   CreateLabel("V_Sess",  "—",         col2, mly, ClrNeutral, true, 10);
   mly += 16;
   CreateLabel("L_Win",   "OpenWin:",  col1, mly, C'120,130,150', false, 9);
   CreateLabel("V_Win",   "—",         col2, mly, C'180,180,200', false, 9);
   mly += 18;
   CreateLabel("L_Spr",   "Spread:",   col1, mly, C'120,130,150', false, 9);
   CreateLabel("V_Spr",   "—",         col2, mly, ClrNeutral, true, 10);
   mly += 18;
   CreateLabel("L_DOM",   "DOM imb:",  col1, mly, C'120,130,150', false, 9);
   CreateLabel("V_DOM",   "—",         col2, mly, ClrNeutral, true, 10);
   mly += 18;
   CreateLabel("L_Atr",   "ATR M1/5:", col1, mly, C'120,130,150', false, 9);
   CreateLabel("V_Atr",   "—",         col2, mly, C'180,180,200', false, 9);
   ly += hMarket + gap;
   
   // ─── TODAY ────────────────────────────────────────────────────────
   CreateRect("TodBox",   x+5, ly,    innerW, hToday,          C'28,34,46');
   CreateRect("TodBar",   x+5, ly,    innerW, 22,              C'62,52,42');
   CreateLabel("TodTtl",  "▼ TODAY",  col1, ly + 4,            C'255,210,150', true, 10);
   int dly = ly + 28;
   CreateLabel("L_DPnL",  "P/L:",      col1, dly, C'120,130,150', false, 9);
   CreateLabel("V_DPnL",  "$0.00",     col2, dly, ClrNeutral, true, 12);
   dly += 22;
   // P/L progress bar (vs daily target)
   CreateProgressBar("DPnL", col1, dly, innerW - 24, 8, 0.0, ClrBullish, C'45,50,60');
   CreateLabel("V_DPnLLbl", "vs $" + DoubleToString(InpDailyProfitTarget, 0), col1, dly + 12, C'120,130,150', false, 8);
   dly += 26;
   CreateLabel("L_Trd",   "W/L Avg:",  col1, dly, C'120,130,150', false, 9);
   CreateLabel("V_Trd",   "0/0 (0%)",  col2, dly, clrWhite, true, 10);
   dly += 18;
   CreateLabel("L_Exit",  "Exits:",    col1, dly, C'120,130,150', false, 9);
   CreateLabel("V_Exit",  "—",         col2, dly, C'200,200,210', false, 9);
   ly += hToday + gap;
   
   // ─── SUSPENDED ────────────────────────────────────────────────────
   CreateRect("SuspBox",  x+5, ly,    innerW, hSusp,           C'28,34,46');
   CreateRect("SuspBar",  x+5, ly,    innerW, 22,              C'72,52,52');
   CreateLabel("SuspTtl", "▼ SUSPENDED", col1, ly + 4,          C'255,170,170', true, 10);
   int sly = ly + 28;
   for(int i = 0; i < 3; i++)
   {
      string si = IntegerToString(i+1);
      CreateLabel("V_Sus"+si, "—", col1, sly, C'180,180,200', false, 9);
      sly += 16;
   }
   ly += hSusp + gap;
   
   // ─── RISK ─────────────────────────────────────────────────────────
   CreateRect("RskBox",   x+5, ly,    innerW, hRisk,           C'28,34,46');
   CreateRect("RskBar",   x+5, ly,    innerW, 22,              C'52,62,82');
   CreateLabel("RskTtl",  "▼ RISK",    col1, ly + 4,            C'160,200,255', true, 10);
   int rsly = ly + 28;
   CreateLabel("L_Cap",   "Cap/trade:", col1, rsly, C'120,130,150', false, 9);
   CreateLabel("V_Cap",   "—",          col2, rsly, ClrNeutral, true, 10);
   rsly += 18;
   CreateLabel("L_Hlt",   "Halt:",       col1, rsly, C'120,130,150', false, 9);
   CreateProgressBar("Hlt", col2, rsly + 3, innerW - (col2 - x) - 12, 8, 0.0, ClrWarning, C'45,50,60');
   ly += hRisk + gap;
   
   // ─── FOOTER ───────────────────────────────────────────────────────
   CreateRect("FtrBox", x+5, ly, innerW, hFooter, C'22,26,34');
   CreateLabel("V_Ftr", "v1.10 | TF=" + EntryBarTfLabel() + " | cycle " +
               IntegerToString(InpEntryIntervalSec) + "s | min " +
               DoubleToString(InpMinSetupScore, 0), col1, ly + 7, C'120,130,150', false, 8);
}

//+------------------------------------------------------------------+
//| Format helpers (purely visual)                                   |
//+------------------------------------------------------------------+
string FormatBiasArrow(int bias)
{
   if(bias > 0) return "UP";
   if(bias < 0) return "DN";
   return "—";
}

string FormatSecsHumane(int secs)
{
   if(secs < 60) return IntegerToString(secs) + "s";
   int m = secs / 60; int s = secs % 60;
   if(m < 60) return IntegerToString(m) + "m" + IntegerToString(s) + "s";
   int h = m / 60; m = m % 60;
   return IntegerToString(h) + "h" + IntegerToString(m) + "m";
}

string FormatExitTally()
{
   if(g_ExitTallyCount == 0) return "—";
   string s = "";
   for(int i = 0; i < g_ExitTallyCount; i++)
   {
      if(i > 0) s += "  ";
      s += g_ExitTallyTags[i] + "×" + IntegerToString(g_ExitTallyHits[i]);
   }
   if(StringLen(s) > 28) s = StringSubstr(s, 0, 28) + "…";
   return s;
}

//+------------------------------------------------------------------+
//| PANEL — Update                                                   |
//+------------------------------------------------------------------+
void UpdatePanel()
{
   int innerW = InpPanelWidth - 14;
   
   // ─── TRADE ────────────────────────────────────────────────────────
   if(g_Ticket > 0)
   {
      string statTxt = (g_PosType == POSITION_TYPE_BUY ? "▲ BUY " : "▼ SELL ");
      double posLot = 0;
      if(PositionSelectByTicket(g_Ticket)) posLot = PositionGetDouble(POSITION_VOLUME);
      statTxt += DoubleToString(posLot, 2) + " lot";
      if(g_PartialTaken) statTxt += " (partial)";
      color statClr = g_PosType == POSITION_TYPE_BUY ? ClrBullish : ClrBearish;
      SetLabel("V_TStat", statTxt, statClr);
      
      string setTxt = g_OpenSetupName + " (" + DoubleToString(g_OpenSetupScore, 0) + ")";
      if(StringLen(setTxt) > 26) setTxt = StringSubstr(setTxt, 0, 26);
      SetLabel("V_TSet", setTxt, ClrAccent);
      
      double sl = 0;
      if(PositionSelectByTicket(g_Ticket)) sl = PositionGetDouble(POSITION_SL);
      SetLabel("V_TEnt", DoubleToString(g_EntryPrice, g_Digits) + " / " +
               (sl > 0 ? DoubleToString(sl, g_Digits) : "—"), clrWhite);
      
      // R:R: ratio of (current best vs entry) is hard at runtime; show open snap
      SetLabel("V_TT1",  "—", clrWhite);  // T1 not stored on position; leave blank
      
      string pnlText = (g_CurrentProfit >= 0 ? "+$" : "-$") + DoubleToString(MathAbs(g_CurrentProfit), 2);
      color  pnlClr  = g_CurrentProfit >= 0 ? ClrBullish : ClrBearish;
      if(g_CurrentProfit >= InpQuickProfitGreat) pnlText += " 🎯";
      else if(g_CurrentProfit >= InpQuickProfitGood) pnlText += " ✓";
      SetLabel("V_TPnL", pnlText, pnlClr);
      
      int ageS = (int)(TimeCurrent() - g_EntryTime);
      string maxTxt = "+$" + DoubleToString(g_MaxProfit, 2) + " / " + FormatSecsHumane(ageS);
      SetLabel("V_TMax", maxTxt, ClrBullish);
   }
   else
   {
      SetLabel("V_TStat", "— NO TRADE",      ClrNeutral);
      SetLabel("V_TSet",  "—",                ClrNeutral);
      SetLabel("V_TEnt",  "— / —",            ClrNeutral);
      SetLabel("V_TT1",   "— / —",            ClrNeutral);
      SetLabel("V_TPnL",  "$0.00",            ClrNeutral);
      SetLabel("V_TMax",  "$0.00 / 0s",       ClrNeutral);
   }
   
   // ─── RANKER ───────────────────────────────────────────────────────
   color vClr = ClrNeutral;
   if(StringFind(g_LastRankerVerdict, "FIRED")    == 0) vClr = ClrBullish;
   else if(StringFind(g_LastRankerVerdict, "REJEC") == 0) vClr = ClrBearish;
   else if(StringFind(g_LastRankerVerdict, "GATED") == 0) vClr = ClrWarning;
   else if(StringFind(g_LastRankerVerdict, "BELOW") == 0) vClr = ClrWarning;
   else if(StringFind(g_LastRankerVerdict, "NO_")   == 0) vClr = ClrNeutral;
   string vTxt = g_LastRankerVerdict;
   if(StringLen(vTxt) > 30) vTxt = StringSubstr(vTxt, 0, 30) + "…";
   SetLabel("V_Vrd", vTxt, vClr);
   
   // Cycle ETA + last proposal count
   int eta = (int)(g_LastRankerCycle + InpEntryIntervalSec - TimeCurrent());
   if(eta < 0) eta = 0;
   string cycTxt = "in " + IntegerToString(eta) + "s | last cycle: " + IntegerToString(g_LastProposalCount);
   SetLabel("V_Cyc", cycTxt, C'180,180,200');
   
   for(int i = 0; i < 3; i++)
   {
      string si = IntegerToString(i+1);
      if(i < g_LastTopCount)
      {
         string ln = "#" + si + " " + g_LastTop[i].name + " " + DoubleToString(g_LastTop[i].score, 0) +
                     " " + (g_LastTop[i].isBuy ? "B" : "S") +
                     " RR" + DoubleToString(g_LastTop[i].rrToT1, 1);
         color tCol = (i == 0 ? C'220,225,235' : (i == 1 ? C'180,185,200' : C'150,155,170'));
         SetLabel("V_Top" + si, ln, tCol);
         string blk = g_LastBlockReasonForTop[i];
         color bCol = ClrNeutral;
         if(blk == "PASS")              bCol = ClrBullish;
         else if(StringLen(blk) > 0)    bCol = ClrWarning;
         if(StringLen(blk) > 38) blk = StringSubstr(blk, 0, 38) + "…";
         SetLabel("V_TopBlk" + si, "  " + blk, bCol);
      }
      else
      {
         SetLabel("V_Top" + si, "#" + si + " —", ClrNeutral);
         SetLabel("V_TopBlk" + si, "", ClrNeutral);
      }
   }
   
   // ─── MARKET ───────────────────────────────────────────────────────
   string m15h1 = FormatBiasArrow(g_M15Bias) + " / " + FormatBiasArrow(g_H1Bias);
   color m15Clr = g_M15Bias > 0 ? ClrBullish : (g_M15Bias < 0 ? ClrBearish : ClrNeutral);
   SetLabel("V_M15", m15h1, m15Clr);
   
   string m5Text = "— NONE";
   if(g_M5Trend > 0)      m5Text = "▲ UP " + DoubleToString(g_M5TrendPct, 0) + "% / ADX " + DoubleToString(g_M5ADX, 1);
   else if(g_M5Trend < 0) m5Text = "▼ DN " + DoubleToString(g_M5TrendPct, 0) + "% / ADX " + DoubleToString(g_M5ADX, 1);
   else                   m5Text = "— FLAT / ADX " + DoubleToString(g_M5ADX, 1);
   color m5Clr = g_M5Trend > 0 ? ClrBullish : (g_M5Trend < 0 ? ClrBearish : ClrNeutral);
   SetLabel("V_M5T", m5Text, m5Clr);
   
   string regText = RegimeLabelForAI();
   color regClr = ClrAccent;
   if(StringFind(regText, "TREND") == 0)                regClr = ClrBullish;
   else if(regText == "RANGE_CHOP" || regText == "RANGE_QUIET") regClr = ClrNeutral;
   else if(regText == "VOL_EXPLOSIVE" || regText == "IMPULSE_BURST") regClr = ClrWarning;
   else if(regText == "NEWS_BLACKOUT")                  regClr = ClrBearish;
   SetLabel("V_Reg", regText, regClr);
   
   string sessText = g_Session + (g_SessionOK ? " ●" : " ○");
   if(g_SessionOK && g_MinutesUntilSessionClose > 0)
      sessText += " (closes " + IntegerToString(g_MinutesUntilSessionClose) + "m)";
   SetLabel("V_Sess", sessText, g_SessionOK ? ClrBullish : ClrBearish);
   
   string winTxt = "—";
   color  winClr = ClrNeutral;
   if(g_InSessionOpenWindow_NY)
   { winTxt = "NY +" + IntegerToString(g_MinutesSinceNYOpen) + "m (boost SPK/NEWS)"; winClr = ClrAccent; }
   else if(g_InSessionOpenWindow_London)
   { winTxt = "LON +" + IntegerToString(g_MinutesSinceLondonOpen) + "m (boost SPK/NEWS)"; winClr = ClrAccent; }
   SetLabel("V_Win", winTxt, winClr);
   
   string sprText = DoubleToString(g_Spread, 0) + " pts";
   color  sprClr  = ClrBullish;
   if(g_Spread > 40)        { sprText += " ⚠"; sprClr = ClrBearish; }
   else if(g_Spread > 30)   { sprText += " ~"; sprClr = ClrWarning; }
   else                     { sprText += " ✓"; sprClr = ClrBullish; }
   SetLabel("V_Spr", sprText, sprClr);
   
   string domText = DoubleToString(g_DomImbalancePct, 0) + "%";
   color  domClr  = ClrNeutral;
   if(g_DomImbalancePct >= 60)         { domText += " (BID heavy)"; domClr = ClrBullish; }
   else if(g_DomImbalancePct <= -60)   { domText += " (ASK heavy)"; domClr = ClrBearish; }
   else if(g_DomImbalancePct >= 40)    { domText += " (bid lean)";  domClr = ClrAccent; }
   else if(g_DomImbalancePct <= -40)   { domText += " (ask lean)";  domClr = ClrAccent; }
   else                                { domText += " (balanced)";  domClr = ClrNeutral; }
   SetLabel("V_DOM", domText, domClr);
   
   SetLabel("V_Atr",  "$" + DoubleToString(g_M1ATR, 2) + " / $" + DoubleToString(g_M5ATR, 2),
            C'200,200,210');
   
   // ─── TODAY ────────────────────────────────────────────────────────
   string dailyText = (g_DailyPnL >= 0 ? "+$" : "-$") + DoubleToString(MathAbs(g_DailyPnL), 2);
   color  dailyClr  = g_DailyPnL >= 0 ? ClrBullish : ClrBearish;
   SetLabel("V_DPnL", dailyText, dailyClr);
   
   double pct = 0;
   if(InpDailyProfitTarget > 0) pct = g_DailyPnL / InpDailyProfitTarget;
   if(pct < 0) pct = 0;
   if(pct > 1) pct = 1;
   color barClr = (g_DailyPnL >= 0 ? ClrBullish : ClrBearish);
   SetProgressBar("DPnL", innerW - 24, pct, barClr, C'45,50,60');
   
   double avgWin  = (g_TodayWins > 0)    ? g_TodayWinSum  / g_TodayWins    : 0.0;
   double avgLoss = (g_TodayLossCnt > 0) ? g_TodayLossSum / g_TodayLossCnt : 0.0;  // negative
   string trdText = IntegerToString(g_TodayWins) + "/" + IntegerToString(g_TodayTrades);
   if(g_TodayTrades > 0)
   {
      double winPct = (double)g_TodayWins / (double)g_TodayTrades * 100;
      trdText += " (" + DoubleToString(winPct, 0) + "%)  ";
      if(g_TodayWins > 0)    trdText += "+$" + DoubleToString(avgWin, 2);
      if(g_TodayLossCnt > 0) trdText += " / -$" + DoubleToString(MathAbs(avgLoss), 2);
   }
   SetLabel("V_Trd", trdText, clrWhite);
   
   SetLabel("V_Exit", FormatExitTally(), C'200,200,210');
   
   // ─── SUSPENDED ────────────────────────────────────────────────────
   int slot = 0;
   for(int k = 0; k < 12 && slot < 3; k++)
   {
      if(g_KindSuspendUntil[k] > TimeCurrent())
      {
         int remSec = (int)(g_KindSuspendUntil[k] - TimeCurrent());
         string nm = SetupKindName((SETUP_KIND)k);
         string si2 = IntegerToString(slot + 1);
         SetLabel("V_Sus" + si2, nm + ": resumes in " + FormatSecsHumane(remSec), C'255,180,180');
         slot++;
      }
   }
   if(slot == 0)
      SetLabel("V_Sus1", "(no setups suspended) ✓", ClrBullish);
   for(int k = MathMax(slot, 1); k < 3; k++)
   {
      string si3 = IntegerToString(k + 1);
      SetLabel("V_Sus" + si3, "", ClrNeutral);
   }
   
   // ─── RISK ─────────────────────────────────────────────────────────
   double cap = CurrentRiskCapUsd();
   string capTxt = "$" + DoubleToString(cap, 2) + "  (max $" + DoubleToString(InpMaxRiskDollars, 2) + ")";
   SetLabel("V_Cap", capTxt, ClrAccent);
   
   double haltPct = 0;
   if(InpMaxDailyLoss > 0) haltPct = (-g_DailyPnL) / InpMaxDailyLoss;  // only count losses
   if(haltPct < 0) haltPct = 0;
   if(haltPct > 1) haltPct = 1;
   color haltClr = haltPct < 0.5 ? ClrBullish : (haltPct < 0.85 ? ClrWarning : ClrBearish);
   // Width matches DrawPanel: innerW - (col2 - x) - 12 where (col2 - x) == 138
   SetProgressBar("Hlt", innerW - 138 - 12, haltPct, haltClr, C'45,50,60');
   
   // v1.11: throttle ChartRedraw — Gold can fire 100+ ticks/sec at NY open and
   // ChartRedraw is the heaviest call in the panel update path. Redraw every N
   // calls (or always force on trade events via ForceChartRedraw helper).
   g_PanelRedrawCounter++;
   int every = (InpPanelRedrawEveryN < 1 ? 1 : InpPanelRedrawEveryN);
   if(g_PanelRedrawCounter % every == 0)
      ChartRedraw(0);
}

// v1.11: Force a chart redraw immediately (used when a trade fires/closes so
// the panel updates without waiting for the throttle interval).
void ForceChartRedraw()
{
   ChartRedraw(0);
   g_PanelRedrawCounter = 0;
}
//+------------------------------------------------------------------+
