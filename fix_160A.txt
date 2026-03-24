#!/bin/sh
# fix_160A.sh - Keep 5GHz radio on 160MHz
# Runs via cron every 30 minutes (1,31 * * * *)

LOG_FILE="/jffs/scripts/fix_160A.log"
LOCK_FILE="/tmp/fix_160A.last_action"
INIT_START="/jffs/scripts/init-start"
IFACE="eth7"
COOLDOWN=60                 # Minimum seconds between recovery attempts
CAC_WAIT=601                 # Seconds to wait for CAC to complete
VERBOSE=1                   # DFS status logging: 0=off, 1=basic (no DFS status), 2=verbose (all DFS status blocks)
PREFERRED="100/160"         # Preferred 160MHz target when outside DFS blocks (149-165)
BACKUP_FREQS="100/160"         # Backup 160MHz target when outside DFS blocks (149-165)
MANAGE_CRON=1               # Set to 1/0 to add/remove cron and init-start entries

# -----------------------------------------
# 0. Start
# -----------------------------------------
echo "$(date): [__START] Script triggered by cron" >> "$LOG_FILE"

# -----------------------------------------
# 1. Functions
# -----------------------------------------

manage_cron_job() {
    local action="$1"
    local job_id="fix_160A"
    local cron_schedule="1,31 * * * * /jffs/scripts/fix_160A.sh"

    if [ "$action" = "add" ]; then
        if cru l 2>/dev/null | grep -q "$job_id"; then
            echo "$(date): [CRON] Cron job '$job_id' already exists. No action needed." >> "$LOG_FILE"
        else
            cru a "$job_id" "$cron_schedule"
            echo "$(date): [ACTION] Added cron job '$job_id'." >> "$LOG_FILE"
        fi
    elif [ "$action" = "remove" ]; then
        if cru l 2>/dev/null | grep -q "$job_id"; then
            cru d "$job_id"
            echo "$(date): [ACTION] Removed cron job '$job_id'." >> "$LOG_FILE"
        else
            echo "$(date): [CRON] Cron job '$job_id' does not exist. No action needed." >> "$LOG_FILE"
        fi
    fi
}

manage_init_start() {
    local action="$1"
    local entry='cru a "fix_160A" "1,31 * * * * /jffs/scripts/fix_160A.sh"'

    if [ "$action" = "add" ]; then
        if [ ! -f "$INIT_START" ]; then
            printf '#!/bin/sh\n%s\n' "$entry" > "$INIT_START"
            chmod +x "$INIT_START"
            echo "$(date): [ACTION] Created '$INIT_START' and added entry." >> "$LOG_FILE"
        elif ! grep -qF "$entry" "$INIT_START"; then
            echo "$entry" >> "$INIT_START"
            echo "$(date): [ACTION] Added entry to existing '$INIT_START'." >> "$LOG_FILE"
        else
            echo "$(date): [INIT] Entry already present in '$INIT_START'. No action needed." >> "$LOG_FILE"
        fi
    elif [ "$action" = "remove" ]; then
        if [ ! -f "$INIT_START" ]; then
            echo "$(date): [INIT] '$INIT_START' does not exist. No action needed." >> "$LOG_FILE"
        elif grep -qF "$entry" "$INIT_START"; then
            sed -i '/fix_160A/d' "$INIT_START"
            echo "$(date): [ACTION] Removed entry from '$INIT_START'." >> "$LOG_FILE"
        else
            echo "$(date): [INIT] Entry not found in '$INIT_START'. No action needed." >> "$LOG_FILE"
        fi
    fi
}

# Log current dfs_ap_move status.
# $1 = label, $2 = minimum VERBOSE level required to log
# VERBOSE=0: no DFS status logs
# VERBOSE=1: basic logging only, no DFS status blocks
# VERBOSE=2: all DFS status blocks (PRE-MOVE, BGDFS-TRANSITION, DURING-CAC, POST-CAC)
log_dfs_status() {
    local label="$1"
    local min_level="${2:-1}"
    [ "$VERBOSE" -lt "$min_level" ] 2>/dev/null && return 0
    [ "$VERBOSE" = "0" ] && return 0
    local status
    status=$(wl -i "$IFACE" dfs_ap_move 2>/dev/null)
    echo "$(date): [DFS_STATUS:$label]" >> "$LOG_FILE"
    echo "$status" | while IFS= read -r line; do
        echo "$(date):   $line" >> "$LOG_FILE"
    done
}

# Attempt dfs_ap_move to target chanspec.
# Returns 0 if accepted, 1 if rejected. Logs both outcomes.
try_dfs_ap_move() {
    local target="$1"
    local err
    err=$(wl -i "$IFACE" dfs_ap_move "$target" 2>&1)
    if [ $? -eq 0 ]; then
        echo "$(date): [ACTION] dfs_ap_move accepted [$target]. Background CAC started." >> "$LOG_FILE"
        return 0
    else
        echo "$(date): [INFO] dfs_ap_move rejected [$target]: $err" >> "$LOG_FILE"
        return 1
    fi
}

# -----------------------------------------
# 2. Self-registration
# -----------------------------------------
if [ "$MANAGE_CRON" = "1" ]; then
    manage_cron_job "add"
    manage_init_start "add"
else
    manage_cron_job "remove"
    manage_init_start "remove"
fi

# -----------------------------------------
# 3. Cooldown check
# -----------------------------------------
if [ -f "$LOCK_FILE" ]; then
    LAST_ACTION=$(cat "$LOCK_FILE")
    NOW=$(date +%s)
    ELAPSED=$((NOW - LAST_ACTION))
    if [ "$ELAPSED" -lt "$COOLDOWN" ]; then
        echo "$(date): [SKIP] Cooldown active. ${ELAPSED}s/${COOLDOWN}s elapsed." >> "$LOG_FILE"
        echo "$(date): [__END] Script completed (cooldown)" >> "$LOG_FILE"
        exit 0
    fi
fi

# -----------------------------------------
# 4. Radio up check
# -----------------------------------------
IS_UP=$(wl -i "$IFACE" isup 2>/dev/null)
if [ "$IS_UP" != "1" ]; then
    echo "$(date): [OFFLINE] Radio $IFACE is [DOWN]. Exiting." >> "$LOG_FILE"
    echo "$(date): [__END] Script completed (radio offline)" >> "$LOG_FILE"
    exit 0
fi
echo "$(date): [ONLINE] Radio $IFACE is [UP]. Proceeding." >> "$LOG_FILE"

# -----------------------------------------
# 5. Read current radio state
# -----------------------------------------
CURRENT_SPEC=$(wl -i "$IFACE" chanspec 2>/dev/null | awk '{print $1}')
CURRENT_CHAN="${CURRENT_SPEC%%/*}"   # Primary channel e.g. "116" from "116/80"
CURRENT_WIDTH="${CURRENT_SPEC#*/}"  # Width e.g. "80" from "116/80"
echo "$(date): [INFO] Current state [CHANSPEC=$CURRENT_SPEC]" >> "$LOG_FILE"

# -----------------------------------------
# 6. Check if already on 160MHz
# -----------------------------------------
if [ "$CURRENT_CHAN" -ge 100 ] && [ "$CURRENT_WIDTH" = "160" ]; then
    echo "$(date): [OK] Already on 160MHz [$CURRENT_SPEC] on 100+ channel. No action needed." >> "$LOG_FILE"
    tail -n 200 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
    echo "$(date): [__END] Script completed successfully" >> "$LOG_FILE"
    exit 0
fi
echo "$(date): [INFO] Current width is [${CURRENT_WIDTH}MHz]. Attempting recovery." >> "$LOG_FILE"
log_dfs_status "PRE-MOVE" 2

# -----------------------------------------
# 7. Select and attempt dfs_ap_move target
# Always target 36 or 100 as primary channel for stability.
# Block membership determines which target to try first,
# falling back to the opposite block if rejected.
# If outside DFS blocks (149-165), use BACKUP_FREQS channel.
# -----------------------------------------
MOVED=0

if [ "$CURRENT_CHAN" -ge 36 ] && [ "$CURRENT_CHAN" -le 64 ]; then
    echo "$(date): [INFO] Channel $CURRENT_CHAN is in 36/160 block. Trying to move to $PREFERRED." >> "$LOG_FILE"
    if try_dfs_ap_move "$PREFERRED"; then
        MOVED=1
    fi
elif [ "$CURRENT_CHAN" -ge 100 ] && [ "$CURRENT_CHAN" -le 128 ]; then
    echo "$(date): [INFO] Channel $CURRENT_CHAN is in 100/160 block. Trying 100/160." >> "$LOG_FILE"
    if try_dfs_ap_move "100/160"; then
        MOVED=1
    fi
else
    echo "$(date): [INFO] Channel $CURRENT_CHAN is outside DFS blocks. Trying backup [$BACKUP_FREQS]." >> "$LOG_FILE"
    if try_dfs_ap_move "$BACKUP_FREQS"; then
        MOVED=1
    else
        # Derive the opposite block from BACKUP_FREQS for the fallback attempt
        if [ "$BACKUP_FREQS" = "100/160" ]; then
            FALLBACK="36/160"
        else
            FALLBACK="100/160"
        fi
        echo "$(date): [INFO] Falling back to $FALLBACK." >> "$LOG_FILE"
        if try_dfs_ap_move "$FALLBACK"; then
            MOVED=1
        fi
    fi
fi

# -----------------------------------------
# 8. If no move was accepted, log and hold
# -----------------------------------------
if [ "$MOVED" = "0" ]; then
    echo "$(date): [NOTICE] All dfs_ap_move attempts rejected. Holding until next cron run." >> "$LOG_FILE"
    tail -n 200 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
    echo "$(date): [__END] Script completed (no move accepted)" >> "$LOG_FILE"
    exit 0
fi

# -----------------------------------------
# 9. Stamp lockfile and wait for CAC to complete
# -----------------------------------------
date +%s > "$LOCK_FILE"
echo "$(date): [INFO] Waiting ${CAC_WAIT}s for CAC to complete..." >> "$LOG_FILE"
log_dfs_status "BGDFS-TRANSITION" 2
sleep 10
log_dfs_status "DURING-CAC" 2
sleep $((CAC_WAIT - 10))

# -----------------------------------------
# 10. Verify result
# -----------------------------------------
POST_SPEC=$(wl -i "$IFACE" chanspec 2>/dev/null | awk '{print $1}')
POST_WIDTH="${POST_SPEC#*/}"
log_dfs_status "POST-CAC" 2
echo "$(date): [INFO] Post-CAC state [CHANSPEC=$POST_SPEC]" >> "$LOG_FILE"

if [ "$POST_WIDTH" = "160" ]; then
    echo "$(date): [OK] Recovery successful. Now on [$POST_SPEC]." >> "$LOG_FILE"
else
    if [ "$BACKUP_FREQS" = "100/160" ]; then
        FALLBACK="36/160"
    else
        FALLBACK="100/160"
    fi
    echo "$(date): [NOTICE] Recovery failed. Still on [$POST_SPEC]. Trying fallback [$FALLBACK]." >> "$LOG_FILE"
    if try_dfs_ap_move "$FALLBACK"; then
        date +%s > "$LOCK_FILE"
        echo "$(date): [INFO] Fallback dfs_ap_move [$FALLBACK] accepted. Result confirmed on next cron run." >> "$LOG_FILE"
    else
        echo "$(date): [NOTICE] Fallback [$FALLBACK] also rejected. Holding until next cron run." >> "$LOG_FILE"
    fi
fi

# -----------------------------------------
# 11. Rotate log
# -----------------------------------------
tail -n 200 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"

# -----------------------------------------
# 12. End
# -----------------------------------------
echo "$(date): [__END] Script completed successfully" >> "$LOG_FILE"

exit 0

