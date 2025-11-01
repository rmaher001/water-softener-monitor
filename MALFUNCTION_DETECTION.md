# Malfunction Detection - Future Feature (v1.7.0+)

## Decision: Wait for Baseline Data

**Date:** November 1, 2025
**Status:** Deferred pending data collection

## Current Observations (Oct 13 - Nov 1)

### Regeneration Cycle Analysis

**First Cycle (Oct 24-25):**
- Pre-regen: ~33cm distance
- Post-regen: ~45cm distance
- Net change: **+12cm** (large drop in salt level)
- Context: User changed softener hardness setting to increase salt consumption
- Interpretation: One-time adjustment to new settings, not typical consumption

**Second Cycle (Oct 27):**
- Pre-regen: ~46cm distance
- Post-regen: ~42cm distance
- Net change: **+4cm** (~8-10% salt consumed)
- Interpretation: Normal consumption pattern with new hardness settings

### Water-Level Fluctuations

**Observed behavior:**
- ±1-2cm fluctuations around stable baseline
- Caused by brine tank water dynamics (evaporation, settling)
- Not related to actual salt consumption

**Key insight:** ToF sensor measures distance to water/salt surface. Cannot distinguish between:
- Real salt consumption (gradual, downward trend only)
- Water level changes (±fluctuations, can go up or down)

## Next Steps

### Data Collection Phase

**Third Cycle Expected:** November 3, 2025 ~2-4 AM PDT (DO=7 max)

**Target metrics:**
- Pre-regen distance: ~42cm (current stable)
- Expected post-regen: ~38cm (if 4cm pattern holds)
- Net change: ~4cm

**Success criteria for baseline confirmation:**
- Third cycle: 4cm ±1cm change
- Fourth cycle (Nov 10): 4cm ±1cm change
- Consistent pattern = ~4cm per 7 days = 0.57cm/day consumption

### Implementation Requirements (v1.7.0+)

**After 2+ cycles confirm pattern:**

1. **Baseline Consumption Rate**
   - Expected: ~4cm per 7-day cycle
   - Daily rate: ~0.57cm/day
   - Use this to detect abnormal behavior

2. **Malfunction Detection Logic**
   ```
   IF distance decreases (level appears to improve)
      AND no manual refill in last 24 hours
      AND NOT during regeneration cycle
      AND change > hysteresis threshold (±2cm)
   THEN
      Flag potential malfunction
      - Could be sensor drift
      - Could be salt bridge (air gap)
      - Could be water evaporation anomaly
   ```

3. **Hysteresis Threshold**
   - Use ±2cm to handle water fluctuations
   - Only alert if distance decreases >2cm beyond normal fluctuation
   - Prevents false positives from daily water-level changes

4. **Status Improvement Blocking**
   - Status can ONLY improve within 24 hours of manual refill button press
   - Otherwise, status frozen or can only degrade
   - Accounts for legitimate refills vs. sensor anomalies

## Technical Implementation Plan

### New Entities (ESPHome)

**Binary Sensor:**
```yaml
binary_sensor:
  - platform: template
    name: "Potential Malfunction Detected"
    id: malfunction_detected
    icon: mdi:alert-circle
```

**Configuration Parameter:**
```yaml
number:
  - platform: template
    name: "Malfunction Detection Threshold"
    id: malfunction_threshold_cm
    min_value: 0
    max_value: 10
    step: 0.5
    initial_value: 2.0
    unit_of_measurement: "cm"
    icon: mdi:arrow-expand-horizontal
```

### Detection Logic (Pseudocode)

```cpp
// Track baseline distance after each regeneration
global float baseline_distance_post_regen = 42.0; // Updated after each cycle

// In distance sensor update lambda:
float current_distance = id(distance_sensor).state;
float change_from_baseline = current_distance - baseline_distance_post_regen;

// Malfunction if distance DECREASES beyond threshold (surface appears closer)
if (change_from_baseline < -id(malfunction_threshold_cm)->state) {
  // Check refill window
  unsigned long hrs_since_refill = (millis()/1000 - id(last_manual_refill_time)) / 3600;

  if (hrs_since_refill > 24 && !id(regen_active)) {
    id(malfunction_detected)->publish_state(true);
    ESP_LOGW("malfunction", "Distance decreased %.1fcm from baseline (%.1f→%.1f)",
             abs(change_from_baseline), baseline_distance_post_regen, current_distance);
  }
} else {
  id(malfunction_detected)->publish_state(false);
}

// Update baseline after regeneration completes
if (id(regen_active) && previous_regen_active) {
  // Cycle just ended, update baseline
  baseline_distance_post_regen = current_distance;
}
```

## Why Wait?

**Risk of premature implementation:**
1. **First cycle was anomaly** - Massive 12cm change due to settings adjustment
2. **Only one "normal" cycle observed** - Oct 27 showed 4cm, but could be outlier
3. **Unknown variables** - Seasonal changes, usage patterns, water hardness variations

**Benefits of waiting:**
1. **Establish reliable baseline** - Need 2-3 consistent cycles to confirm 4cm pattern
2. **Tune hysteresis properly** - Observe actual water fluctuation range over time
3. **Avoid false alerts** - Better to wait than deploy buggy malfunction detection

## Review Timeline

- **Nov 3:** Analyze third cycle data
- **Nov 10:** Analyze fourth cycle data
- **Nov 10+:** If pattern confirmed, implement v1.7.0 malfunction detection
- **Dec 2025:** Re-evaluate if seasonal changes affect consumption pattern

## Notes

- Water softener setting: Increased hardness compensation (Oct 24)
- Demand override: DO=7 (max 7 days between cycles)
- Current stable distance: ~42cm (Nov 1)
- Manual refill feature: Already implemented in v1.6.0 (button press)
