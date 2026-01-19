# AI Debate Hub - Helper Functions

Reference documentation for bash helper functions used in debate orchestration.

---

## Atomic State Updates

Use this helper to update state.json safely. Direct writes risk corruption.

```bash
update_debate_state() {
    local state_file="$1"
    local new_data="$2"

    # Check if flock is available
    if ! command -v flock &>/dev/null; then
        echo "WARN: flock not available, skipping file locking" >&2
        echo "$new_data" > "${state_file}.tmp" || return 1
        if command -v jq &>/dev/null && ! jq empty "${state_file}.tmp" 2>/dev/null; then
            rm "${state_file}.tmp" 2>/dev/null
            return 1
        fi
        mv "${state_file}.tmp" "$state_file" || return 1
        return 0
    fi

    (
        flock -x -w 5 200 || {
            echo "ERROR: Could not acquire lock on $state_file after 5s" >&2
            return 1
        }

        if [[ -f "$state_file" ]]; then
            cp "$state_file" "${state_file}.backup" 2>/dev/null
        fi

        echo "$new_data" > "${state_file}.tmp" || {
            echo "ERROR: Failed to write to ${state_file}.tmp" >&2
            return 1
        }

        if command -v jq &>/dev/null; then
            if ! jq empty "${state_file}.tmp" 2>/dev/null; then
                echo "ERROR: Invalid JSON in new state" >&2
                rm "${state_file}.tmp" 2>/dev/null
                return 1
            fi
        fi

        mv "${state_file}.tmp" "$state_file" || {
            echo "ERROR: Failed to atomically update $state_file" >&2
            if [[ -f "${state_file}.backup" ]]; then
                mv "${state_file}.backup" "$state_file" 2>/dev/null
            fi
            return 1
        }

        rm "${state_file}.backup" 2>/dev/null
        rm "${state_file}.tmp" 2>/dev/null

    ) 200>"${state_file}.lock"
}
```

**Usage:**
```bash
current_state=$(cat "$STATE_FILE")
new_state=$(echo "$current_state" | jq '.current_round = 2')
update_debate_state "$STATE_FILE" "$new_state"
```

---

## Exponential Backoff Retry

Retry advisor calls with intelligent failure detection.

```bash
run_advisor_with_retry() {
    local advisor="$1"
    local prompt="$2"
    local max_retries="${3:-3}"
    local base_timeout="${4:-90}"

    local attempt=1
    local timeout=$base_timeout

    while [[ $attempt -le $max_retries ]]; do
        echo "[$advisor] Attempt $attempt/$max_retries (timeout: ${timeout}s)" >&2

        if run_advisor "$advisor" "$prompt" "$timeout"; then
            return 0
        fi

        local error_output=$(get_last_error 2>&1)

        if [[ $attempt -eq $max_retries ]]; then
            echo "ERROR: [$advisor] Failed after $max_retries attempts" >&2
            return 1
        fi

        local failure_mode=$(detect_failure_mode "$advisor" "$error_output")

        case "$failure_mode" in
            rate_limit)
                echo "WARN: [$advisor] Rate limit, waiting 60s..." >&2
                sleep 60
                ;;
            network_timeout)
                local wait_time=$((2 ** (attempt - 1)))
                echo "WARN: [$advisor] Timeout, waiting ${wait_time}s..." >&2
                sleep "$wait_time"
                timeout=$((timeout * 2))
                ;;
            session_expired)
                echo "WARN: [$advisor] Session expired" >&2
                return 2  # Caller should create new session
                ;;
            usage_limit)
                echo "ERROR: [$advisor] Usage limit reached" >&2
                return 3  # Caller should skip this advisor
                ;;
            *)
                local wait_time=$((2 ** (attempt - 1)))
                echo "WARN: [$advisor] Error, waiting ${wait_time}s..." >&2
                sleep "$wait_time"
                timeout=$((timeout + base_timeout))
                ;;
        esac

        attempt=$((attempt + 1))
    done

    return 1
}

detect_failure_mode() {
    local advisor="$1"
    local error_output="$2"

    if echo "$error_output" | grep -Eqi "session.*(expired|not found|invalid|closed)"; then
        echo "session_expired"
        return
    fi

    if [[ "$advisor" == "gemini" ]]; then
        if echo "$error_output" | grep -Eqi "quota exceeded|rate limit|too many requests|429"; then
            echo "rate_limit"
            return
        fi
    elif [[ "$advisor" == "codex" ]]; then
        if echo "$error_output" | grep -Eqi "rate limit|too many requests|slow down|429"; then
            echo "rate_limit"
            return
        fi
    fi

    if echo "$error_output" | grep -Eqi "timeout|timed out|ETIMEDOUT|ECONNRESET"; then
        echo "network_timeout"
        return
    fi

    if echo "$error_output" | grep -Eqi "usage.*limit|quota.*exceeded|billing|403"; then
        echo "usage_limit"
        return
    fi

    echo "unknown_error"
}
```

**Return codes:**
- `0` — Success
- `1` — All retries failed
- `2` — Session expired (create new session)
- `3` — Usage limit (skip advisor)

---

## Contextual Error Messages

User-friendly error output with troubleshooting steps.

```bash
log_contextual_error() {
    local advisor="$1"
    local error_type="$2"
    local error_details="$3"
    local recovery_action="$4"

    echo "" >&2
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" >&2
    echo "ERROR: $error_type" >&2
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" >&2
    echo "Advisor: $advisor" >&2
    echo "Details: $error_details" >&2
    echo "" >&2
    echo "Troubleshooting:" >&2

    case "$error_type" in
        "Session Resume Failed")
            echo "  1. Check CLI: $advisor --version" >&2
            echo "  2. List sessions:" >&2
            [[ "$advisor" == "gemini" ]] && echo "     gemini --list-sessions" >&2
            [[ "$advisor" == "codex" ]] && echo "     codex resume --all" >&2
            ;;
        "Network Timeout")
            echo "  1. Check internet connection" >&2
            echo "  2. Retrying with longer timeout" >&2
            ;;
        "Rate Limit")
            echo "  1. Waiting 60s for reset" >&2
            echo "  2. Check quota:" >&2
            [[ "$advisor" == "gemini" ]] && echo "     https://console.cloud.google.com/apis/dashboard" >&2
            [[ "$advisor" == "codex" ]] && echo "     https://platform.openai.com/usage" >&2
            ;;
        "Usage Limit")
            echo "  1. Check billing:" >&2
            [[ "$advisor" == "gemini" ]] && echo "     https://console.cloud.google.com/billing" >&2
            [[ "$advisor" == "codex" ]] && echo "     https://platform.openai.com/account/billing" >&2
            ;;
    esac

    echo "" >&2
    echo "Recovery: $recovery_action" >&2
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━" >&2
}

log_advisor_success() {
    local advisor="$1"
    local round="$2"
    local words="$3"
    local seconds="$4"

    echo "✓ $advisor Round $round completed ($words words, ${seconds}s)" >&2
}
```

**Usage:**
```bash
log_contextual_error "gemini" "Session Resume Failed" \
    "Session abc123 not found" \
    "Creating new session with full context"

log_advisor_success "gemini" 1 287 12
```
