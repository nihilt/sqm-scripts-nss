
# sqm-scripts-nss
Smart Queue Management Scripts for OpenWRT for use with NSS optimized builds.

NSS FQ-Codel proves very effective at maintaining low latency under load, while causing minimal CPU load on the router. 

Currently only supports nssfq-codel and no traffic classification / marking due to limitations of the current driver. 
```
https://github.com/rickkdotnet/sqm-scripts-nss
```
# SQM-Autorate (NSS version)

```markdown

SQM‑Autorate is a Perl daemon that works alongside OpenWrt’s `sqm-scripts` using the NSS QoS script (`nss-rk.qos`). It dynamically adjusts only the shaper **rate** of `nsstbl` based on measured latency and keeps advanced qdisc parameters (`buffer`, `mtu`, `target`, `interval`, `flows`, `quantum`, `accel_mode`) as configured by SQM.

---

# === SQM-AUTORATE INIT.D SERVICE COMMANDS ===
/etc/init.d/sqm-autorate start        # Start the sqm-autorate service
/etc/init.d/sqm-autorate stop         # Stop the sqm-autorate service
/etc/init.d/sqm-autorate restart      # Restart the service (stop + start)
/etc/init.d/sqm-autorate enable       # Enable service autostart at boot
/etc/init.d/sqm-autorate disable      # Disable service autostart at boot


# === SWITCH-PROFILE COMMANDS ===
switch-profile list                   # List all available profiles
switch-profile gaming                 # Switch to gaming profile
switch-profile streaming              # Switch to streaming profile
switch-profile performance            # Switch to performance profile
switch-profile debug                  # Switch to debug profile
switch-profile <profile> --default    # Switch to profile AND set it as default for next boot

# === SQM-STATUS HELPER COMMANDS ===
sqm-status                            # Show full dashboard (service state, profiles, uptime, logs, summary)
sqm-status current                    # Show only the active profile (one-liner)

# Logging controls
sqm-status log-on                     # Enable logging (writes log_enabled=1 into config and restarts service)
sqm-status log-off                    # Disable logging (writes log_enabled=0 into config and restarts service)
sqm-status log-enable-all             # Enable logging AND rotation together
sqm-status log-disable-all            # Disable logging AND rotation together

# Verbosity / debugging levels
sqm-status log-level-1                # Set verbosity to LEVEL 1 (minimal logging detail)
sqm-status log-level-2                # Set verbosity to LEVEL 2 (medium logging detail)
sqm-status log-level-3                # Set verbosity to LEVEL 3 (debug logging, most detail)

# Rotation controls
sqm-status rotate-on                  # Enable log rotation (moves config back into /etc/logrotate.d)
sqm-status rotate-off                 # Disable log rotation (renames config to .disabled)

# Log file management
sqm-status log-live                   # Live stream the log file (tail -f, Ctrl+C to stop)
sqm-status log-clear                  # Clear the log file contents (truncate to empty)

## Dependencies installation

apk update && apk add perl perlbase-file perlbase-getopt perlbase-time perlbase-threads ip-full tc-full iputils-ping logrotate procps-ng coreutils procd jsonfilter

CONFIG_PACKAGE_jsonfilter=y
CONFIG_PACKAGE_perl=y
CONFIG_PACKAGE_perlbase-file=y
CONFIG_PACKAGE_perlbase-getopt=y
CONFIG_PACKAGE_perlbase-time=y
CONFIG_PACKAGE_perlbase-threads=y
CONFIG_PACKAGE_ip-full=y
CONFIG_PACKAGE_tc-full=y
CONFIG_PACKAGE_iputils-ping=y
CONFIG_PACKAGE_logrotate=y
CONFIG_PACKAGE_procps-ng=y
CONFIG_PACKAGE_coreutils=y
CONFIG_PACKAGE_procd=y

Delete this entry:
CONFIG_FEED_sqm_scripts_nss=n

feeds.conf.default:
```
```
src-git sqm_scripts_nss https://github.com/nihilt/sqm-scripts-nss.git;SQM-Autorate-NSS
```
---

## Streaming Profile Example

```ini
# PROFILE: streaming
# /etc/sqm-autorate-streaming.conf

# Base line rates (set to ~95% of your maximum line capacity).
# These are the "fixed ceilings" that autorate will try to recover back to.
upload_base_kbits=85000               # Maximum uplink speed in kbps
download_base_kbits=830000            # Maximum downlink speed in kbps

# Logging defaults
log_enabled=1                         # 1 = log activity, 0 = disable logging
log_level=1                           # 1 = minimal info, 2 = medium detail, 3 = full debug

# Reflectors (servers to ping for latency measurement).
# Multiple reflectors improve reliability if one server is slow or unreachable.
reflectors="1.1.1.1 208.67.222.222 208.67.220.220 9.9.9.9"

# Minimum allowed rates (floors as % of base).
# Prevents autorate from cutting speeds too low; ensures a usable minimum bandwidth.
upload_min_percent=50                 # Uplink will never drop below 50% of base
download_min_percent=50               # Downlink will never drop below 50% of base

# Adjustment aggressiveness.
# Defines how quickly rates are raised or lowered when latency crosses thresholds.
increase_rate_percent_up=6            # Raise uplink by 6% when latency is low
decrease_rate_percent_up=10           # Cut uplink by 10% when latency is high
increase_rate_percent_down=6          # Raise downlink by 6% when latency is low
decrease_rate_percent_down=8          # Cut downlink by 8% when latency is high

# Latency thresholds (ms).
# These are the "trigger points" for rate changes.
delay_low_target_up=12                # If uplink latency < 12 ms, increase rate
delay_high_target_up=15               # If uplink latency > 15 ms, decrease rate
delay_low_target_down=15              # If downlink latency < 15 ms, increase rate
delay_high_target_down=20             # If downlink latency > 20 ms, decrease rate

# Latency smoothing.
# Prevents reacting to single spikes by averaging or filtering samples.
latency_filter=median                 # Use median of last samples for stability
latency_window_size=5                 # Number of samples considered

# Probing intervals.
# How often latency is checked. Elastic probing speeds up checks when latency is unstable.
ping_interval_ms=250                  # Normal probe interval (every 250 ms)
ping_interval_fast_ms=150             # Faster probe interval when variance is high
elastic_probe=1                       # Enable automatic switch between normal/fast probing
elastic_variance_ms=2                 # If latency spread > 2 ms, switch to fast probing

# Adaptive floor settings.
# Floors can rise temporarily if latency stays bad, preventing rates from bouncing too low.
adaptive_floor=1                      # Enable adaptive floor adjustments
adaptive_floor_step=2                 # Raise floor by 2% each time triggered
adaptive_floor_max=70                 # Floors will never exceed 70% of base
adaptive_floor_trigger_ms=15          # Latency above 15 ms contributes to streak
adaptive_floor_trigger_count=5        # After 5 consecutive triggers, floor is raised

# Decay settings.
# Allow adaptive floors to "relax" back down over time, so speeds can recover.
# In plain terms: decay settings allow faster recovery back to your fixed base rates.
adaptive_floor_min=50                 # Floors will not decay below 50% of base
adaptive_floor_decay_interval=300     # Every 300 seconds (5 minutes), decay check runs
adaptive_floor_decay_step=2           # Floors drop by 2% each decay interval

# Load-aware bias.
# Adds extra cuts when traffic volume is consistently high, to keep latency under control.
load_aware=1                          # Enable load-aware bias
load_check_interval=3                 # Check every 3 cycles
load_bias_decrease=5                  # Cut rates by 5% if threshold exceeded
load_bias_threshold_bytes=4000000     # If >4 MB transferred per cycle, bias is triggered
```

---
- **Base rates** = your maximum speeds (autorate tries to return to these).  
- **Floors** = the lowest speeds autorate will allow.  
- **Latency thresholds** = when to cut or raise speeds.  
- **Adaptive floors** = floors rise if latency stays bad, preventing wild swings.  
- **Decay settings** = floors slowly drop back down, letting speeds recover faster to base.  
- **Load-aware bias** = extra safety cut when traffic is heavy, keeping latency smooth.  
---

## Recommended tuning procedure

1. Disable SQM/NSS QoS and run the Waveform Bufferbloat Test. Note max up/down throughput.
2. Set `upload_base_kbits` and `download_base_kbits` to ~95% of measured line rate.
3. Enable SQM with NSS QoS (`nss-rk.qos`) and re-run Waveform to verify reduced latency under load.
4. Adjust base values to match shaped throughput reported by Waveform.
5. Copy tuned base values into each profile (gaming, streaming, performance, debug).
6. Enable autorate at boot and restart:
   ```
   /etc/init.d/sqm-autorate enable
   /etc/init.d/sqm-autorate restart
   sqm-status
   ```

---

## Example log output

```text
Thu Dec  4 06:38:03 2025 Cycle latency=13 ms
Thu Dec  4 06:38:03 2025 Probe interval: 150 ms (spread=2, lat=13)
Thu Dec  4 06:38:04 2025 Latency=19 ms Applied NSS rates: UPLINK=76500 kbps burst=15000b (rc=0), DOWNLINK=747000 kbps burst=15000b (rc=0)
Thu Dec  4 06:38:11 2025 Latency=9 ms Applied NSS rates: UPLINK=80306 kbps burst=15000b (rc=0), DOWNLINK=784170 kbps burst=15000b (rc=0)
Thu Dec  4 06:39:54 2025 sqm-autorate started
```

---
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------





