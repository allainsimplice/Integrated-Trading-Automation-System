# DOCUMENTATION: The Resilient Execution Expert Advisor (EA) - Integrated Version

**Version:** 6.04 (MS Engine Integration + Trading Session Filter)  
**Date:** 18-Jun-2024 (Last Update)  
**Project:** A specialized trade execution EA designed to work with the "MS Engine Integrated" indicator.

## 1. High-Level Overview

This MetaTrader 5 Expert Advisor (EA) is a specialized "Execution Engine." Its primary purpose is to execute trades with precision based on signals and context provided by the "MS Engine Integrated" indicator and its own candlestick pattern validation, while respecting configurable trading session restrictions.

The core strategy within the integrated system is as follows:
1. **Trading Session Validation:** First checks if current time falls within the configured trading session (London, New York, Crossover, or 24/7).
2. **Receives Setup from MS Engine:** On each new bar, the EA checks for an active trading setup communicated by the MS Engine via Global Variables. This setup includes a trade direction (BUY/SELL) and a MaxTP (Take Profit) level.
3. **Candlestick Pattern Search:** If a setup is active from the MS Engine and within valid trading session, the EA analyzes the previously closed bar for a specific "Hammer" (for BUYs) or "Inverted Hammer" (for SELLs) profile, matching the direction given by the MS Engine.
4. **Fibonacci Test:** The identified signal candle must pass a strict Fibonacci test (body location within the candle's range).
5. **Engulfing Stop Loss:** Intelligently adjusts the stop loss if the signal candle (or the structure it's part of) is engulfed by prior candles, defining risk by the true controlling market structure.
6. **Take Profit Validation:** The EA calculates its ideal Take Profit (TP) based on its input `RRRatio`. This ideal TP is then validated against the `MaxTPLevel` provided by the MS Engine.
   - For BUY setups: Ideal TP must be `<= MaxTPLevel`.
   - For SELL setups: Ideal TP must be `>= MaxTPLevel`.
   - If this condition is violated, the trade is **rejected**. Otherwise, the ideal TP is used.
7. **Execution:** If all validations pass, it attempts to enter a trade with its preferred method: a pending Buy Stop or Sell Stop order.
8. **Conditional Market Fallback:** If the pending order fails due to a price gap (invalid price), the EA enters a monitoring mode, executing a market order only when the price breaks the high/low of the original signal candle.
9. **Notification:** Upon successful trade execution (position opened), the EA notifies the MS Engine by setting a specific Global Variable, allowing the MS Engine to invalidate its setup.
10. **Trade Management:** Once in a trade, it manages the position with an advanced trailing stop loss.

## 2. The Development Journey

**Phases 1-5:** (Original EA development)
- **Phase 1 & 2: Foundation:** Stripped down to an efficient "once per bar" model with core safety features.
- **Phase 3: Perfecting Entry Signal (Fib Test & Engulfing Logic):** Implemented the "Fib Test" and the "Engulfing SL" feature for more accurate risk definition.
- **Phase 4: Solving "Invalid Price" Execution:** Developed the "Conditional Market Fallback" for resilient execution.
- **Phase 5: Advanced Trade Management:** Implemented a sophisticated trailing stop loss.

**Phase 6: MS Engine Integration (v6.00-6.03):**
- Added logic to read setup parameters (direction, MaxTP level, Setup ID) from MS Engine via Global Variables.
- Modified signal finding to filter by MS Engine's direction.
- Implemented strict TP validation against MS Engine's MaxTP level, rejecting trades if the EA's ideal R:R TP is beyond the MaxTP boundary.
- Restored original EA's lot sizing calculation logic, fed by price distance converted to pips.
- Added notification mechanism (setting a GV) to inform MS Engine of trade execution.

**Phase 7: Trading Session Filter (v6.04):**
- Added comprehensive trading session filtering with four options: Disabled (24/7), London Only, New York Only, or London/NY Crossover.
- Implemented UTC-based session calculations with automatic Daylight Saving Time (DST) detection for both London and New York markets.
- Session filter acts as the first validation check, preventing new trade entries outside configured trading hours.
- Weekend detection automatically active when any session filter is enabled.
- Enhanced logging to show current session status and timezone information.

## 3. Detailed Breakdown of Final Code (Version 6.04)

### Input Parameters:
- `RiskPercent`: Risk per trade in percent of account balance.
- `RRRatio`: Desired Risk:Reward ratio for trade setups.
- `EnableTrailingStop`, `TrailingTriggerPercent`, `ProfitLockPercent`: Control the advanced trailing stop loss.
- `EnableMSEngineIntegration`: Master switch to enable/disable communication with MS Engine (should be `true` for integrated operation).
- **`TradingSession`**: NEW - Dropdown parameter to control trading session restrictions:
  - `SESSION_DISABLED` (Default): Trade 24/7 as before
  - `SESSION_LONDON`: Trade only during London market hours (08:00-17:00 local time)
  - `SESSION_NEWYORK`: Trade only during New York market hours (08:00-17:00 local time)  
  - `SESSION_CROSSOVER`: Trade only during London/NY overlap period

### Core Functions & Logic:

**`OnInit()`**:
- Initializes Global Variable prefixes for MS Engine communication.
- Logs the selected trading session configuration.
- Displays current session status with timezone information.

**`OnTick()`**:
- **Tick-Based Logic:** Manages `ManageTrailingStop()` and Conditional Fallback Monitoring on every tick.
- **Bar-Based Logic (Enhanced):** Once per new bar:
  1. **NEW: Session Validation (First Priority):** Calls `IsValidTradingSession()` to check if current UTC time falls within the configured trading session. If outside valid session, exits early with periodic logging.
  2. **MS Engine Integration:** If `EnableMSEngineIntegration` is true, checks Global Variables for an active setup from MS Engine.
  3. **Signal Processing:** If valid session and active setup exist, calls `FindSignalCandle()` with direction and MaxTP level from MS Engine.
  4. **Trade Execution:** If valid signal found, calculates lot size and attempts trade execution.

**Session Filter Functions (NEW in v6.04):**
- **`IsLondonDST()` / `IsNewYorkDST()`:** Automatically detect current Daylight Saving Time status for each market using precise DST transition rules.
- **`IsLondonSession()` / `IsNewYorkSession()`:** Check if current UTC time falls within respective market hours, accounting for DST.
- **`IsCrossoverSession()`:** Determines if both London and New York sessions are simultaneously active.
- **`IsWeekend()`:** Detects Saturday/Sunday in UTC time.
- **`IsValidTradingSession()`:** Main validation function that combines all session logic based on the selected `TradingSession` parameter.
- **`GetCurrentSessionStatus()`:** Returns detailed string showing current UTC time, session status, and DST information for logging.

**Session Calculations:**
- **London Session:**
  - Winter (GMT): 08:00-17:00 local = 08:00-17:00 UTC
  - Summer (BST): 08:00-17:00 local = 07:00-16:00 UTC
- **New York Session:**
  - Winter (EST): 08:00-17:00 local = 13:00-22:00 UTC  
  - Summer (EDT): 08:00-17:00 local = 12:00-21:00 UTC
- **Crossover Period:** Dynamic overlap that varies from 3-5 hours depending on DST combinations

**`FindSignalCandle(int msSetupDirection, double msMaxTPLevel)`** (Unchanged):
- Performs directional filtering, Fibonacci test, engulfing pattern SL adjustment, and MaxTP validation.
- Returns valid setup only if all criteria are met.

**`CalculateAndValidateLotSize(double stopLossPips)`** (Unchanged):
- Uses the exact logic from the original EA.
- Takes stop loss distance in pips and calculates appropriate lot size based on risk parameters.

**Conditional Market Fallback Logic** (Unchanged):
- Monitors for market entry when pending orders fail due to invalid price.
- Executes market order when price breaks the original signal candle level.
- Notifies MS Engine upon successful fallback execution.

### Session Filter Behavior:

**Entry Restriction Only:**
- Session filter ONLY prevents NEW trade entries outside configured hours.
- Running positions remain completely unaffected (continue to TP/SL naturally).
- Position management (trailing stops, etc.) continues regardless of session.
- Fallback market order monitoring continues during all sessions.

**Weekend Handling:**
- When any session filter is enabled (not Disabled), automatically respects weekends.
- No trades Saturday/Sunday when session filtering is active.
- When set to Disabled, maintains full 24/7 behavior.

**Timezone Independence:**
- Uses `TimeGMT()` for all calculations, making the system independent of:
  - Trader's physical location
  - Local computer timezone settings
  - Broker server timezone (London, Cyprus, New York, etc.)
- Automatic DST detection eliminates the "sometimes opens 1 hour different" problem.

**Enhanced Logging:**
- Session status logged at EA startup.
- Periodic logging (hourly) when outside valid trading sessions to avoid spam.
- Successful trade logs include current session status.
- Sample log: `"EA EURUSD: Current Status: UTC: 14:30 | London:OPEN(BST) NY:OPEN(EDT) | CROSSOVER-ACTIVE"`

## 4. Final Philosophy and Conclusion

The integrated EA with session filtering acts as an intelligent, resilient, and time-aware execution agent for setups identified by the MS Engine. It prioritizes valid entries according to specific candlestick profiles, context (direction, MaxTP boundary) provided externally, and configurable trading session restrictions. Its risk management is context-aware (engulfing SL), its execution logic is robust (pending with market fallback), and its session filtering is timezone-independent with automatic DST handling. By adhering to the MS Engine's directives, validating against structural levels, and respecting trading session boundaries, it forms a crucial part of a sophisticated, collaborative, and professionally time-managed trading system.

The session filter feature enables traders to implement institutional-style trading schedules while maintaining the system's core counter-trend strategy, ensuring trades are only taken during optimal market conditions without affecting existing position management or system reliability.