# OpenClaw Setup Token - One-Shot Runbook

## Overview
Generate a 1-year Claude OAuth token locally, inject it into the OpenClaw VPS container, verify it works. One terminal session, no back-and-forth.

---

## Prerequisites
- Claude Code CLI installed locally (`claude` in PATH)
- Active Claude Max subscription on the account you'll authenticate with
- SSH access to VPS (`ssh openclaw` - alias configured in `~/.ssh/config`)
- Browser available locally for OAuth flow

---

## Step 1: Generate the Token (Local Machine)

```bash
claude setup-token
```

**What happens:** Browser opens, you authenticate with your Claude Max account. CLI prints a token starting with `sk-ant-oat01-`. Copy it - you'll need it exactly once.

**If it fails:** You may need to `claude auth logout` first if a stale session exists, then retry.

Store the token temporarily:
```bash
export NEW_TOKEN="sk-ant-oat01-PASTE_YOUR_TOKEN_HERE"
```

---

## Step 2: SSH into VPS and Discover Current State

```bash
ssh openclaw
```

Once connected, run this diagnostic block to map the current auth state:

```bash
echo "=== STEP 2: Environment Discovery ==="

# Locate the docker compose directory
COMPOSE_DIR=$(find /docker -maxdepth 2 -name "docker-compose.yml" -path "*openclaw*" -exec dirname {} \; 2>/dev/null | head -1)
echo "Compose dir: ${COMPOSE_DIR:-NOT FOUND}"

# Locate the .env file
ENV_FILE="${COMPOSE_DIR}/.env"
echo "Env file: ${ENV_FILE}"
[ -f "$ENV_FILE" ] && echo "  EXISTS" || echo "  MISSING - STOP HERE"

# Show current OAuth token (masked)
CURRENT_TOKEN=$(grep '^CLAUDE_CODE_OAUTH_TOKEN=' "$ENV_FILE" 2>/dev/null | cut -d= -f2-)
echo "Current OAuth token: ${CURRENT_TOKEN:0:20}...${CURRENT_TOKEN: -6}"

# Check settings.json for token references
SETTINGS_FILE="${COMPOSE_DIR}/data/.claude/settings.json"
echo "Settings file: ${SETTINGS_FILE}"
[ -f "$SETTINGS_FILE" ] && echo "  EXISTS" || echo "  NOT FOUND (ok - env var is primary)"

# Check the shim
SHIM_FILE="${COMPOSE_DIR}/data/.agents/bin/claude"
echo "Claude shim: ${SHIM_FILE}"
[ -f "$SHIM_FILE" ] && echo "  EXISTS" || echo "  MISSING - needs investigation"

# Check container name
CONTAINER=$(docker ps --format '{{.Names}}' | grep -i openclaw | head -1)
echo "Running container: ${CONTAINER:-NOT RUNNING}"

# Check entrypoint wrapper
ENTRYPOINT="${COMPOSE_DIR}/host-ops/entrypoint-wrapper.sh"
echo "Entrypoint wrapper: ${ENTRYPOINT}"
[ -f "$ENTRYPOINT" ] && echo "  EXISTS" || echo "  NOT FOUND"

echo ""
echo "=== Discovery complete. Review above before proceeding. ==="
```

**Decision point - read the output:**
- If `COMPOSE_DIR` is empty: search with `find /docker -maxdepth 3 -type d -name "*openclaw*"`
- If `ENV_FILE` is missing: the token injection path changes - check `docker inspect $CONTAINER` for env source
- If container is not running: proceed with injection, start after

---

## Step 3: Inject the New Token

Set the new token variable on the VPS (paste the token you copied in Step 1):

```bash
NEW_TOKEN="sk-ant-oat01-PASTE_YOUR_TOKEN_HERE"
```

Then run the injection:

```bash
echo "=== STEP 3: Token Injection ==="

# Backup current .env
cp "$ENV_FILE" "${ENV_FILE}.bak.$(date +%Y%m%d-%H%M%S)"
echo "Backup created: ${ENV_FILE}.bak.$(date +%Y%m%d-%H%M%S)"

# Replace the OAuth token in .env
if grep -q '^CLAUDE_CODE_OAUTH_TOKEN=' "$ENV_FILE"; then
    sed -i "s|^CLAUDE_CODE_OAUTH_TOKEN=.*|CLAUDE_CODE_OAUTH_TOKEN=${NEW_TOKEN}|" "$ENV_FILE"
    echo "Replaced existing CLAUDE_CODE_OAUTH_TOKEN in .env"
else
    echo "CLAUDE_CODE_OAUTH_TOKEN=${NEW_TOKEN}" >> "$ENV_FILE"
    echo "Appended CLAUDE_CODE_OAUTH_TOKEN to .env (was not present)"
fi

# Verify the write
WRITTEN_TOKEN=$(grep '^CLAUDE_CODE_OAUTH_TOKEN=' "$ENV_FILE" | cut -d= -f2-)
if [ "${WRITTEN_TOKEN}" = "${NEW_TOKEN}" ]; then
    echo "VERIFIED: Token written correctly (${WRITTEN_TOKEN:0:20}...${WRITTEN_TOKEN: -6})"
else
    echo "ERROR: Token mismatch after write. Check manually."
    echo "Expected: ${NEW_TOKEN:0:20}..."
    echo "Got:      ${WRITTEN_TOKEN:0:20}..."
    exit 1
fi

# If settings.json exists and contains a token reference, update it too
if [ -f "$SETTINGS_FILE" ]; then
    if grep -q 'CLAUDE_CODE_OAUTH_TOKEN' "$SETTINGS_FILE"; then
        echo "settings.json also references token - updating"
        # Use python for safe JSON editing
        python3 -c "
import json
with open('${SETTINGS_FILE}', 'r') as f:
    data = json.load(f)
if 'env' in data and 'CLAUDE_CODE_OAUTH_TOKEN' in data['env']:
    data['env']['CLAUDE_CODE_OAUTH_TOKEN'] = '${NEW_TOKEN}'
    with open('${SETTINGS_FILE}', 'w') as f:
        json.dump(data, f, indent=2)
    print('settings.json updated')
else:
    print('No env.CLAUDE_CODE_OAUTH_TOKEN in settings.json - skipping')
"
    else
        echo "settings.json does not reference token directly - no change needed"
    fi
fi

echo ""
echo "=== Injection complete ==="
```

---

## Step 4: Restart the Container

```bash
echo "=== STEP 4: Container Restart ==="

# Use the approved restart method (never docker compose directly)
bash /host-ops/vps-ops.sh oc-restart

# Wait for container to be healthy
echo "Waiting for container..."
sleep 10

# Check container status
CONTAINER=$(docker ps --format '{{.Names}}' | grep -i openclaw | head -1)
if [ -n "$CONTAINER" ]; then
    STATUS=$(docker inspect --format='{{.State.Status}}' "$CONTAINER")
    echo "Container: $CONTAINER - Status: $STATUS"
else
    echo "ERROR: Container not found after restart"
    echo "Check: docker ps -a | grep openclaw"
    exit 1
fi
```

---

## Step 5: Verify Auth Works End-to-End

```bash
echo "=== STEP 5: Verification ==="

# Test 1: Check the token is visible inside the container
docker exec "$CONTAINER" bash -c '
    echo "--- Inside container ---"
    echo "CLAUDE_CODE_OAUTH_TOKEN set: $([ -n "$CLAUDE_CODE_OAUTH_TOKEN" ] && echo YES || echo NO)"
    echo "ANTHROPIC_API_KEY set: $([ -n "$ANTHROPIC_API_KEY" ] && echo YES || echo NO)"
    echo "Token prefix: ${CLAUDE_CODE_OAUTH_TOKEN:0:20}..."
'

# Test 2: Run a minimal Claude Code call through the shim
docker exec "$CONTAINER" bash -c '
    export PATH="/data/.agents/bin:$PATH"
    echo "Running: claude -p \"say hello\""
    timeout 30 claude -p "say hello" 2>&1 | head -5
    EXIT_CODE=$?
    if [ $EXIT_CODE -eq 0 ]; then
        echo "AUTH VERIFIED - Claude responded via OAuth token"
    else
        echo "AUTH FAILED - exit code $EXIT_CODE"
        echo "Debug: check if shim unsets API key correctly"
        cat /data/.agents/bin/claude
    fi
'

# Test 3: Confirm the shim is still correctly configured
docker exec "$CONTAINER" cat /data/.agents/bin/claude

echo ""
echo "=== Verification complete ==="
```

---

## Step 6: Cleanup

```bash
# Clear the token from shell history on the VPS
unset NEW_TOKEN
history -d $(history 1 | awk '{print $1}') 2>/dev/null

# Exit VPS
exit

# Clear the token from local shell history too
unset NEW_TOKEN
history -d $(history 1 | awk '{print $1}') 2>/dev/null
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `claude setup-token` says "not subscribed" | Wrong account or lapsed sub | Check account at claude.ai/settings |
| Token writes but auth fails | Token expired or wrong format | Must start with `sk-ant-oat01-` |
| Container won't restart | entrypoint crash | `docker logs $CONTAINER --tail 50` |
| Shim missing after restart | entrypoint overwrites PATH | Check entrypoint-wrapper.sh restores shim |
| Rate limited immediately | Old token was already rate-limited | New token resets this - wait 5 min |
| `vps-ops.sh oc-restart` fails | Script error | Fall back: `cd $COMPOSE_DIR && docker compose restart` |

## Token Lifecycle
- **Valid for:** ~1 year from generation
- **Renewal:** Run this entire runbook again before expiry
- **Monitoring:** If OpenClaw starts getting 401s, token has expired
- **Calendar reminder:** Set for 11 months from today (February 2027)

---

## Security Notes
- Never commit the token to git
- The .env.bak files contain the old token - remove them after confirming the new one works: `rm ${ENV_FILE}.bak.*`
- The token is scoped to your Max subscription - if the sub lapses, the token stops working
- `claude setup-token` invalidates any previous setup token for that account
