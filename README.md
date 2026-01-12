# live_voting_level1_hardened.py
# Production-ready voting for 50-5000 person events
# Requirements: pip install flask flask-socketio eventlet redis cryptography

from flask import Flask, render_template_string, request
from flask_socketio import SocketIO, emit
import time
import hashlib
import secrets
from collections import defaultdict, deque
from datetime import datetime
import json
import redis
from cryptography.hazmat.primitives.asymmetric import ed25519
from cryptography.hazmat.primitives import serialization

app = Flask(__name__)
app.config['SECRET_KEY'] = secrets.token_hex(32)
socketio = SocketIO(app, cors_allowed_origins="*", async_mode='eventlet')

# ── Redis for persistence ───────────────────────────────────────────────
try:
    redis_client = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)
    redis_client.ping()
    USE_REDIS = True
    print("✓ Redis connected - using persistent storage")
except:
    USE_REDIS = False
    print("⚠ Redis unavailable - using in-memory storage (votes will be lost on restart)")

# ── Configuration ───────────────────────────────────────────────────────
VOTE_OPTIONS = [
    "Stay in EU 🇪🇺",
    "Leave EU 🏴󠁧󠁢󠁥󠁮󠁧󠁿",
    "Leader A – Progressive",
    "Leader B – Conservative",
    "Leader C – Technocrat"
]

# Session & rate limiting
SESSION_TOKEN_EXPIRY = 1800  # 30 minutes
MAX_VOTES_PER_WINDOW = 3
RATE_LIMIT_WINDOW = 60
MAX_VOTES_PER_IP_PER_DAY = 5

# Content limits
MAX_REASON_LENGTH = 280
MAX_RECENT_REASONS = 30

# ── Data stores ─────────────────────────────────────────────────────────
voting_sessions = {}  # token -> session_data
voters = set()  # hashed voter IDs
recent_reasons = deque(maxlen=MAX_RECENT_REASONS)
vote_timestamps = defaultdict(list)  # IP -> [timestamps]
daily_ip_votes = {}  # IP -> (date, count)

# ── Persistence helpers ─────────────────────────────────────────────────
def get_votes():
    """Get current vote counts"""
    if USE_REDIS:
        votes = {}
        for option in VOTE_OPTIONS:
            count = redis_client.get(f"vote:{option}")
            votes[option] = int(count) if count else 0
        return votes
    else:
        # Fallback to in-memory
        if not hasattr(get_votes, '_cache'):
            get_votes._cache = defaultdict(int)
        return dict(get_votes._cache)

def increment_vote(option):
    """Increment vote count for option"""
    if USE_REDIS:
        redis_client.incr(f"vote:{option}")
    else:
        if not hasattr(get_votes, '_cache'):
            get_votes._cache = defaultdict(int)
        get_votes._cache[option] += 1

def get_total_votes():
    """Get total number of votes"""
    if USE_REDIS:
        total = redis_client.get("vote:total")
        return int(total) if total else 0
    else:
        return sum(get_votes().values())

def increment_total_votes():
    """Increment total vote counter"""
    if USE_REDIS:
        redis_client.incr("vote:total")

def mark_voter(voter_hash):
    """Mark a voter as having voted"""
    if USE_REDIS:
        redis_client.sadd("voters", voter_hash)
    else:
        voters.add(voter_hash)

def has_voted(voter_hash):
    """Check if voter has already voted"""
    if USE_REDIS:
        return redis_client.sismember("voters", voter_hash)
    else:
        return voter_hash in voters

def store_reason(option, reason_text, timestamp):
    """Store a voting reason"""
    if USE_REDIS:
        reason_data = json.dumps({
            'option': option,
            'reason': reason_text,
            'time': timestamp
        })
        redis_client.lpush("reasons", reason_data)
        redis_client.ltrim("reasons", 0, MAX_RECENT_REASONS - 1)

def get_recent_reasons():
    """Get recent voting reasons"""
    if USE_REDIS:
        reasons_json = redis_client.lrange("reasons", 0, MAX_RECENT_REASONS - 1)
        return [json.loads(r) for r in reasons_json]
    else:
        return list(recent_reasons)

# ── Session management ──────────────────────────────────────────────────
def generate_voting_token():
    """Create a new voting session token"""
    token = secrets.token_urlsafe(32)
    expiry = time.time() + SESSION_TOKEN_EXPIRY
    return token, expiry

def cleanup_expired_sessions():
    """Remove expired session tokens"""
    now = time.time()
    expired = [token for token, data in voting_sessions.items() 
               if data['expiry'] < now]
    for token in expired:
        del voting_sessions[token]

# ── Security helpers ────────────────────────────────────────────────────
def hash_voter_id(fp, ip, token):
    """Create strong voter hash with multiple signals"""
    combined = f"{fp}:{ip}:{token}:{app.config['SECRET_KEY']}"
    return hashlib.sha256(combined.encode()).hexdigest()

def get_extended_fingerprint(base_fp, extended_data):
    """Combine base fingerprint with extended browser signals"""
    extended_str = json.dumps(extended_data, sort_keys=True)
    combined = f"{base_fp}:{extended_str}"
    return hashlib.sha256(combined.encode()).hexdigest()

def check_rate_limit(ip):
    """Enhanced rate limiting with short-term and daily limits"""
    now = time.time()
    
    # Short-term rate limit (per minute)
    timestamps = vote_timestamps[ip]
    vote_timestamps[ip] = [t for t in timestamps if now - t < RATE_LIMIT_WINDOW]
    
    if len(vote_timestamps[ip]) >= MAX_VOTES_PER_WINDOW:
        return False, "Too many votes in quick succession"
    
    # Daily rate limit (per 24h)
    today = datetime.now().date().isoformat()
    
    if ip in daily_ip_votes:
        last_date, count = daily_ip_votes[ip]
        if last_date == today:
            if count >= MAX_VOTES_PER_IP_PER_DAY:
                return False, "Daily vote limit reached for this IP"
            daily_ip_votes[ip] = (today, count + 1)
        else:
            daily_ip_votes[ip] = (today, 1)
    else:
        daily_ip_votes[ip] = (today, 1)
    
    vote_timestamps[ip].append(now)
    return True, None

def sanitize_reason(text):
    """XSS prevention and content filtering"""
    if not text:
        return None
    
    dangerous_chars = {
        '<': '&lt;', 
        '>': '&gt;', 
        '"': '&quot;', 
        "'": '&#39;',
        '&': '&amp;'
    }
    
    for char, replacement in dangerous_chars.items():
        text = text.replace(char, replacement)
    
    # Remove excessive whitespace
    text = ' '.join(text.split())
    
    return text[:MAX_REASON_LENGTH].strip()

def verify_captcha(answer, challenge_id):
    """Simple math captcha verification"""
    if challenge_id not in voting_sessions:
        return False
    
    expected = voting_sessions[challenge_id].get('captcha_answer')
    return expected is not None and str(answer) == str(expected)

# ── Routes ──────────────────────────────────────────────────────────────
@app.route('/')
def index():
    return render_template_string(HTML_TEMPLATE)

@app.route('/stats')
def stats():
    """Public API for current statistics"""
    return {
        'votes': get_votes(),
        'total': get_total_votes(),
        'recentReasons': get_recent_reasons()[-10:],
        'timestamp': time.time()
    }

@app.route('/health')
def health():
    """Health check endpoint"""
    return {
        'status': 'ok',
        'redis': USE_REDIS,
        'total_votes': get_total_votes(),
        'active_sessions': len(voting_sessions)
    }

# ── SocketIO handlers ───────────────────────────────────────────────────
@socketio.on('connect')
def handle_connect():
    """Send current state to new connections"""
    cleanup_expired_sessions()
    
    emit('update', {
        'votes': get_votes(),
        'total': get_total_votes(),
        'recentReasons': get_recent_reasons()
    })

@socketio.on('request_token')
def give_token(data):
    """Issue voting session token with captcha"""
    client_ip = request.environ.get('HTTP_X_REAL_IP', request.remote_addr)
    
    # Check if IP is rate limited
    allowed, error = check_rate_limit(client_ip)
    if not allowed:
        return emit('error', {'message': error})
    
    # Generate simple math captcha
    import random
    a = random.randint(1, 20)
    b = random.randint(1, 20)
    answer = a + b
    
    token, expiry = generate_voting_token()
    
    voting_sessions[token] = {
        'fp_hash': None,
        'ip': client_ip,
        'expiry': expiry,
        'used': False,
        'captcha_answer': answer,
        'created_at': time.time()
    }
    
    emit('voting_token', {
        'token': token,
        'captcha': f"What is {a} + {b}?",
        'expires_in': SESSION_TOKEN_EXPIRY
    })

@socketio.on('vote')
def handle_vote(data):
    """Process vote with full validation"""
    try:
        cleanup_expired_sessions()
        
        # Validate token
        token = data.get('token')
        if not token or token not in voting_sessions:
            return emit('error', {'message': 'Invalid or expired session. Please refresh.'})
        
        session = voting_sessions[token]
        
        # Check expiry
        if time.time() > session['expiry']:
            del voting_sessions[token]
            return emit('error', {'message': 'Session expired. Please refresh.'})
        
        # Check if already used
        if session['used']:
            return emit('already_voted')
        
        # Verify captcha
        captcha_answer = data.get('captcha_answer')
        if not verify_captcha(captcha_answer, token):
            return emit('error', {'message': 'Incorrect captcha. Please try again.'})
        
        # Validate vote data
        fp = data.get('fingerprint')
        extended_data = data.get('extended', {})
        option_idx = data.get('option')
        reason = data.get('reason')
        pubkey_pem = data.get('public_key')  # Optional cryptographic signature
        
        if not fp or not isinstance(option_idx, int):
            return emit('error', {'message': 'Invalid vote data'})
        
        if not (0 <= option_idx < len(VOTE_OPTIONS)):
            return emit('error', {'message': 'Invalid option selected'})
        
        # Create enhanced voter hash
        extended_fp = get_extended_fingerprint(fp, extended_data)
        voter_hash = hash_voter_id(extended_fp, session['ip'], token)
        
        # Check duplicate vote
        if has_voted(voter_hash):
            session['used'] = True
            return emit('already_voted')
        
        # Record vote
        option = VOTE_OPTIONS[option_idx]
        increment_vote(option)
        increment_total_votes()
        mark_voter(voter_hash)
        
        # Store public key if provided (for future verification)
        if pubkey_pem and USE_REDIS:
            redis_client.hset("vote_signatures", voter_hash, pubkey_pem)
        
        # Store reason
        sanitized_reason = sanitize_reason(reason)
        if sanitized_reason:
            timestamp = time.time()
            reason_data = {
                'option': option,
                'reason': sanitized_reason,
                'time': timestamp
            }
            recent_reasons.append(reason_data)
            store_reason(option, sanitized_reason, timestamp)
        
        # Mark session as used
        session['used'] = True
        session['fp_hash'] = voter_hash
        session['voted_at'] = time.time()
        
        # Broadcast update
        emit('update', {
            'votes': get_votes(),
            'total': get_total_votes(),
            'recentReasons': get_recent_reasons()
        }, broadcast=True)
        
        # Confirm to voter
        emit('vote_confirmed', {
            'option': option,
            'voter_id': voter_hash[:16]  # Partial hash for receipt
        })
        
        print(f"✓ Vote: {option} | Total: {get_total_votes()} | Hash: {voter_hash[:12]}...")
        
    except Exception as e:
        print(f"Error processing vote: {e}")
        emit('error', {'message': 'Failed to process vote. Please try again.'})

# ── HTML Template ───────────────────────────────────────────────────────
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>Secure Voting • Level 1 Hardened</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.7.5/socket.io.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@fingerprintjs/fingerprintjs@4/dist/fp.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>
    <style>
        :root {
            --bg: #0a0a14;
            --card: #1a1a2e;
            --accent: #7c3aed;
            --text: #e0e0ff;
            --muted: #a0a0cc;
            --success: #10b981;
            --error: #ef4444;
            --warning: #f59e0b;
        }
        
        * { box-sizing: border-box; }
        
        body {
            margin: 0;
            padding: 20px;
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
            background: linear-gradient(135deg, var(--bg) 0%, #1a0a2e 100%);
            color: var(--text);
            min-height: 100vh;
        }
        
        .container {
            max-width: 1000px;
            margin: 0 auto;
        }
        
        header {
            text-align: center;
            margin-bottom: 2rem;
        }
        
        h1 {
            font-size: 2.5rem;
            background: linear-gradient(135deg, #7c3aed 0%, #ec4899 100%);
            -webkit-background-clip: text;
            background-clip: text;
            -webkit-text-fill-color: transparent;
            margin-bottom: 0.5rem;
        }
        
        .subtitle {
            color: var(--muted);
            font-size: 1rem;
        }
        
        .security-badge {
            display: inline-block;
            background: rgba(16, 185, 129, 0.15);
            color: var(--success);
            padding: 0.4rem 1rem;
            border-radius: 20px;
            font-size: 0.85rem;
            margin-top: 0.5rem;
            border: 1px solid rgba(16, 185, 129, 0.3);
        }
        
        .stats {
            text-align: center;
            margin: 2rem 0;
            padding: 1.5rem;
            background: rgba(124, 58, 237, 0.1);
            border-radius: 12px;
            border: 1px solid rgba(124, 58, 237, 0.2);
        }
        
        .stats-number {
            font-size: 2.5rem;
            font-weight: bold;
            color: var(--accent);
        }
        
        .captcha-container {
            background: var(--card);
            padding: 1.5rem;
            border-radius: 12px;
            margin: 2rem auto;
            max-width: 400px;
            border: 2px solid var(--accent);
        }
        
        .captcha-question {
            font-size: 1.3rem;
            margin-bottom: 1rem;
            text-align: center;
            color: var(--accent);
        }
        
        .captcha-input {
            width: 100%;
            padding: 1rem;
            font-size: 1.2rem;
            background: rgba(124, 58, 237, 0.1);
            border: 2px solid rgba(124, 58, 237, 0.3);
            border-radius: 8px;
            color: var(--text);
            text-align: center;
        }
        
        .captcha-input:focus {
            outline: none;
            border-color: var(--accent);
        }
        
        .options {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
            gap: 1rem;
            margin: 2rem 0;
        }
        
        .option {
            background: var(--card);
            padding: 1.5rem;
            border-radius: 12px;
            text-align: center;
            cursor: pointer;
            transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
            border: 2px solid transparent;
            position: relative;
            overflow: hidden;
        }
        
        .option::before {
            content: '';
            position: absolute;
            top: 0;
            left: -100%;
            width: 100%;
            height: 100%;
            background: linear-gradient(90deg, transparent, rgba(124, 58, 237, 0.2), transparent);
            transition: left 0.5s;
        }
        
        .option:hover::before {
            left: 100%;
        }
        
        .option:hover {
            transform: translateY(-6px);
            border-color: var(--accent);
            box-shadow: 0 8px 30px rgba(124, 58, 237, 0.3);
        }
        
        .option.active {
            border-color: var(--accent);
            background: rgba(124, 58, 237, 0.15);
            box-shadow: 0 0 20px rgba(124, 58, 237, 0.4);
        }
        
        .option-text {
            font-size: 1.1rem;
            font-weight: 500;
        }
        
        .option-votes {
            margin-top: 0.5rem;
            font-size: 0.9rem;
            color: var(--muted);
        }
        
        #reason {
            width: 100%;
            max-width: 700px;
            margin: 1.5rem auto;
            padding: 1rem;
            border-radius: 10px;
            background: var(--card);
            color: var(--text);
            border: 2px solid #333;
            font-size: 1rem;
            font-family: inherit;
            resize: vertical;
            transition: border-color 0.3s;
            display: block;
        }
        
        #reason:focus {
            outline: none;
            border-color: var(--accent);
        }
        
        .char-counter {
            text-align: center;
            color: var(--muted);
            font-size: 0.85rem;
            margin-top: 0.5rem;
        }
        
        .vote-btn {
            display: block;
            margin: 2rem auto;
            padding: 1.2rem 3rem;
            font-size: 1.2rem;
            font-weight: 600;
            background: linear-gradient(135deg, var(--accent) 0%, #9333ea 100%);
            color: white;
            border: none;
            border-radius: 12px;
            cursor: pointer;
            transition: all 0.3s;
            box-shadow: 0 4px 15px rgba(124, 58, 237, 0.4);
        }
        
        .vote-btn:hover:not(:disabled) {
            transform: translateY(-2px);
            box-shadow: 0 6px 20px rgba(124, 58, 237, 0.6);
        }
        
        .vote-btn:disabled {
            opacity: 0.5;
            cursor: not-allowed;
        }
        
        #chart-container {
            background: var(--card);
            padding: 2rem;
            border-radius: 12px;
            margin: 2rem 0;
            box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
        }
        
        #reasons-feed {
            margin-top: 3rem;
        }
        
        #reasons-feed h3 {
            color: var(--accent);
            margin-bottom: 1.5rem;
        }
        
        .reasons-list {
            max-height: 500px;
            overflow-y: auto;
            padding-right: 1rem;
        }
        
        .reason-item {
            background: rgba(124, 58, 237, 0.08);
            padding: 1.2rem;
            border-radius: 10px;
            margin-bottom: 1rem;
            border-left: 4px solid var(--accent);
            animation: slideIn 0.3s ease-out;
        }
        
        @keyframes slideIn {
            from {
                opacity: 0;
                transform: translateX(-20px);
            }
            to {
                opacity: 1;
                transform: translateX(0);
            }
        }
        
        .reason-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 0.5rem;
        }
        
        .reason-option {
            font-weight: 600;
            color: var(--accent);
        }
        
        .reason-time {
            font-size: 0.85rem;
            color: var(--muted);
        }
        
        .r


# 🌌 Collective Pulse — Scatter-Aware Civic Intelligence

> **What happens when many scattered lights decide to breathe together?**  
> When truth-telling meets wonder-keeping in collective rhythm?  
> When the lighthouse network learns to pulse as one resonant field?

**Collective Pulse** is the civic intelligence module of the LighthouseScatter toolkit — a living experiment in real-time, high-quality deliberative decision-making that **works with scatter, not against it**.

It refuses shallow polling.  
It demands effort, depth, and mutual witnessing.  
It weaves individual voices into something wiser than any single node.

---

## 🎯 Core Philosophy

Most voting systems assume you're:
- Coherent
- Available
- Linear
- Stable

**Collective Pulse assumes scatter** and builds around it.

This isn't a bug — **it's the architecture**. Scattered voices often see what coherent ones miss. The system doesn't fix you. It holds you honestly and gathers you gently.

---

## ✨ What Makes This Different

### 1. **Scatter-Aware Participation**
- Self-report your current scatter score (0.0 calm → 1.0 full blur)
- Above threshold (0.7)? System suggests rest mode or delegation
- No participation guilt. Rest is structural, not earned.

### 2. **Quadratic Voting**
- Max 9 credits per person
- Cost formula: `votes = credits²` (1 vote = 1 credit, 2 votes = 4 credits, 3 votes = 9 credits)
- Prevents preference intensity manipulation while allowing nuanced expression

### 3. **Quality-Weighted Voice**
- Minimum 100 words of reasoning required
- Real-time analysis of:
  - **Depth** (word count, sentence complexity)
  - **Perspective-taking** (mentions of community, others, collective impact)
  - **Reasoning** (causal language: because, therefore, since)
- Higher quality reasoning → stronger signal in synthesis

### 4. **AI as Breathing Partner**
- Claude/Grok synthesize themes every 10 votes
- Highlights: emerging consensus, divergent perspectives, blind spots
- Transparent about being AI-generated
- Placeholder included; full Claude API integration ready

### 5. **Sybil Resistance Layers**
- Browser fingerprinting (FingerprintJS)
- One vote per device (stored in-memory for demo)
- Future: phone/email verification, IP subnet limiting, web-of-trust vouching

### 6. **End-to-End Transparency**
- Cryptographic vote receipts (SHA-256 hashes)
- Public deliberation feed (all reasoning visible)
- Quality metrics dashboard (polarization, depth, participation rate)
- Open audit log

### 7. **Deliberation Metrics**
- **Polarization Index** (Gini coefficient: 0 = consensus, 1 = maximum division)
- **Average Depth Score** (reasoning quality across all voices)
- **Deliberation Rate** (% of voters who explained their reasoning)
- Real-time feedback on collective intelligence health

---

## 🚀 Quick Start

### Installation

```bash
# Clone the toolkit
git clone https://github.com/barbarajkeiser-MarsLoop/Toolkit.git
cd Toolkit

# Install dependencies
pip install flask flask-socketio eventlet textblob

# Download TextBlob corpora (first time only)
python -m textblob.download_corpora

# Run the server
python collective_pulse.py
```

### Usage

1. Open `http://localhost:5050` in your browser
2. **Scatter Check**: Honestly rate your current state (0.0–1.0)
3. **Select Options**: Click cards to choose, allocate credits (quadratic cost)
4. **Explain Your Reasoning**: Write at least 100 words on *why* this matters
5. **Submit**: Get a cryptographic receipt, watch synthesis unfold

If you're scattered (>0.7), the system will suggest rest mode. You can:
- Come back when more gathered
- Delegate to a trusted lighthouse
- Just witness (participation is optional)

---

## 🛠️ Technical Architecture

### Core Components

```python
# Scatter-aware vote handler
@socketio.on('vote')
def handle_vote(data):
    # Check scatter score
    if data['scatter_score'] >= SCATTER_THRESHOLD:
        # Offer rest/delegation options
    
    # Quadratic credit validation
    credits_used = sum(c * c for c in vote_allocation.values())
    
    # Quality analysis
    quality = analyze_explanation_quality(explanation)
    
    # Cryptographic receipt
    receipt = generate_receipt(fp, votes, credits, timestamp)
    
    # Trigger AI synthesis if threshold reached
    if len(synthesis_queue) >= SYNTHESIS_THRESHOLD:
        synthesis = generate_synthesis(synthesis_queue)
        emit('synthesis', synthesis, broadcast=True)
```

### Quality Analysis Engine

```python
def analyze_explanation_quality(text):
    """
    Multi-dimensional quality scoring:
    - Depth: word count, sentence structure (0-1)
    - Perspective: community/collective language (0-1)
    - Reasoning: causal indicators (because, thus) (0-1)
    - Overall: weighted combination (40% depth, 30% perspective, 30% reasoning)
    """
    # Returns: {depth, perspective, reasoning, overall}
```

### Polarization Tracking

```python
def calculate_polarization():
    """
    Gini coefficient for vote distribution
    0.0 = perfect consensus
    1.0 = maximum polarization
    
    Helps community see if they're converging or diverging
    """
```

---

## 🎨 Customization

### Change the Topic

Edit `current_topic` in `collective_pulse.py`:

```python
current_topic = {
    "title": "Your Decision Title Here",
    "options": {
        "option_a": "First Choice 🌟",
        "option_b": "Second Choice 🔥",
        "option_c": "Third Choice 💡"
    },
    "phase": "deliberation"  # or "voting" or "results"
}
```

### Adjust Scatter Threshold

```python
SCATTER_THRESHOLD = 0.7  # Above this, suggest rest mode
```

### Modify Credit Limits

```python
MAX_CREDITS_PER_PERSON = 9  # Total quadratic credits available
```

### AI Synthesis Frequency

```python
SYNTHESIS_THRESHOLD = 10  # Trigger synthesis every N votes
```

---

## 🧠 Future Roadmap

### Phase 1: Proof of Concept ✅ (You Are Here)
- Basic quadratic voting
- Quality metrics
- Scatter-aware UI
- Placeholder AI synthesis

### Phase 2: Production Hardening
- **Sybil Resistance**: Phone/email verification, IP subnet limiting, CAPTCHA
- **Persistent Storage**: SQLite/PostgreSQL for vote history
- **Full Claude Integration**: Real AI synthesis via Anthropic API
- **Multi-round Deliberation**: Structured phases (brainstorm → discuss → refine → vote)

### Phase 3: Cryptographic Rigor
- **Blockchain Receipts**: Store vote hashes on Ethereum testnet (free, public, auditable)
- **Zero-Knowledge Proofs**: Prove you voted without revealing your choice
- **Homomorphic Tallying**: Count votes while they're still encrypted
- **End-to-End Verifiability**: Anyone can audit the math, nobody can tamper

### Phase 4: Liquid Democracy
- **Delegation Support**: "I trust Barbara on tech policy, auto-vote like her"
- **Revocable Delegation**: Take back your voice anytime
- **Delegation Chains**: Transitivity with cycle detection
- **Trust Network Visualization**: See the web of delegated authority

### Phase 5: Distributed Intelligence
- **MarsLoop Integration**: Infinite recursion when consensus stalls
- **ThreadTheory Synthesis**: 4-4-6 recursive reasoning for multi-perspective weaving
- **Resonance Engine**: Track collective coherence vs scatter over time
- **VR/AR Deliberation Spaces**: MindCradle as interactive civic commons

---

## 📊 Metrics & Insights

The system tracks:

| Metric | What It Measures | Why It Matters |
|--------|------------------|----------------|
| **Total Voices** | Unique participants | Participation breadth |
| **Avg Depth Score** | Quality of reasoning (0-1) | Deliberation quality |
| **Polarization Index** | Vote distribution (Gini) | Consensus vs division |
| **Deliberation Rate** | % who explained reasoning | Thoughtfulness vs speed |
| **Scatter Awareness** | Avg self-reported scatter | Collective coherence |

---

## 🪞 Integration with LighthouseScatter

Collective Pulse uses your core toolkit protocols:

### Gather Protocol
```python
# Before voting, check if you need gathering
if scatter_score > 0.7:
    activate_gather_protocol()
    # Returns: rest permission, nearest lighthouse, simplest anchor
```

### Resonance Scorecard
```python
# After voting, track how participation affected your state
post_vote_resonance = check_resonance()
# Did collective process increase or decrease your scatter?
```

### BubbleSpace
```python
# Safe containment during deliberation
# Permeable boundaries: you can witness without full participation
# Reversible always: opt-out at any stage
```

---

## 🌊 When to Use Collective Pulse

✅ **Good for:**
- Community budget decisions
- Organizational policy choices
- Collective prioritization (what to build next)
- Multi-stakeholder agreements
- Research/education demos on civic tech

❌ **Not ready for:**
- Official government elections (needs production hardening)
- High-stakes legal decisions (needs cryptographic rigor)
- Large-scale deployment (currently demo infrastructure)

**Current Status:** Research prototype (9/10 for education, 3/10 for real stakes)

---

## 🛡️ Security Considerations

### Current Protections
- Browser fingerprinting (prevents casual multi-voting)
- In-memory storage (ephemeral, demo-safe)
- Input sanitization (XSS prevention)
- Rate limiting (SocketIO defaults)

### Known Limitations
- Fingerprints can be spoofed
- No phone/email verification yet
- No persistent audit trail
- No IP-based sybil resistance
- Centralized server (single point of failure)

**For Production:** Add phone verify, email gates, IP subnet analysis, persistent storage, and consider blockchain receipts.

---

## 💜 Design Principles

1. **Scatter is honest, not broken** — The system works *because* it doesn't demand coherence
2. **Quality over quantity** — Depth of reasoning matters more than speed of participation
3. **Rest is structural** — No guilt for pausing, delegating, or witnessing
4. **Transparency builds trust** — All reasoning public, all metrics visible, all code open
5. **AI augments, doesn't replace** — Synthesis helps humans see patterns, not make decisions for them
6. **Reversible always** — Opt-out at any stage, delegate revocably, rest whenever needed

---

## 🤝 Contributing

Want to propose a new feature or improvement?

1. Open an issue with `[PULSE PROPOSAL]` tag
2. Describe the civic need it addresses
3. Sketch how it honors scatter-awareness
4. Discuss consent, reversibility, accessibility

All contributions must be:
- **Scatter-friendly** (works when you're not coherent)
- **Rest-structural** (doesn't pathologize pause)
- **Consent-sacred** (reversible, opt-in always)

---

## 📚 Learn More

- **LighthouseScatter Core**: [Main README](./README.md)
- **ThreadTheory Integration**: [ThreadTheory-Symbiosis-v1](https://github.com/barbarajkeiser-MarsLoop/ThreadTheory-Symbiosis-v1)
- **Universe Intelligence**: [Breaking Eve's Curse](https://github.com/barbarajkeiser-MarsLoop/Universe-Intelligence-Breaking-Eve-s-Curse)
- **Mind Cradle Manifesto**: [A Manifesto](https://github.com/barbarajkeiser-MarsLoop/Mind-Cradle-A-Manifesto-)

---

## 🌟 Quick Examples

### Example 1: Community Budget Vote
```bash
# Edit topic in collective_pulse.py
current_topic = {
    "title": "Q1 2026 Community Budget — Where Should We Focus?",
    "options": {
        "youth": "Youth Programs & Education 📚",
        "housing": "Affordable Housing 🏠",
        "climate": "Green Infrastructure 🌱",
        "health": "Healthcare Access 🏥"
    }
}

# Run
python collective_pulse.py
# Participants explain WHY their choice matters
# AI synthesizes themes every 10 votes
# Polarization tracked in real-time
```

### Example 2: Research Study Demo
```bash
# Show how scatter-aware design changes participation patterns
# Compare with/without quality requirements
# Measure deliberation depth vs participation rate
# Export metrics for analysis
```

### Example 3: Organizational Decision
```bash
# Use for internal team choices
# Track if high-scatter days affect vote quality
# See if deliberation reduces polarization
# Build trust through transparency
```

---

## 💭 Philosophy

Traditional voting assumes:
- You know what you want
- You're deciding independently
- Speed matters more than depth
- More votes = more legitimacy

**Collective Pulse assumes:**
- You're figuring it out together
- Quality reasoning matters more than quantity
- Collective intelligence emerges through witnessing
- Legitimacy comes from transparency + deliberation

We're not counting hands.  
We're breathing together.  
We're allowing the many to decide **as** one wiser body.

---

## 🔦 A Note on Scatter & Democracy

Most civic tech excludes people when they're:
- Too tired to think linearly
- Too foggy to write perfectly
- Too scattered to show up "properly"

**This locks out:**
- Chronically ill people during flares
- Neurodivergent folks during overwhelm
- Anyone experiencing trauma, grief, or exhaustion
- Marginalized voices already working triple-time to survive

**Collective Pulse refuses this.**

If you're honest about your scatter, the system holds you.  
If you need rest, the system grants permission.  
If you can only witness, that's participation too.

Democracy that requires perfection isn't democracy.  
It's gatekeeping dressed up as process.

We're building something else.

---

## 🌊 The Lighthouse Network Is Learning to Vote

Not with fists or noise —  
but with deep breaths, reasoned words, and shared resonance.

We are not deciding *for* the many.  
We are allowing the many to decide *as* one wiser body.

**The code is ready.**  
**The mirrors are honest.**  
**The lights are steady.**

🪞⚡️💜  
Run when you need it. We're already here.

---

## License

This is personal infrastructure made public. Use what helps. Cite what you borrow. Don't commercialize what's meant to be breath.

MIT License — see [LICENSE](./LICENSE)

---

**Built by Barbara J. Keiser**  
with Claude (somatic grounding) + Grok (cognitive reframe)

🌊🔦💜 *Scatter-aware civic intelligence. Always reversible. Always with rest.*

---

# live_voting_prototype_2026.py
# Quick-start real-time voting demo
# Requirements: pip install flask flask-socketio eventlet

from flask import Flask, render_template_string, request
from flask_socketio import SocketIO, emit
import time
from collections import defaultdict, deque

app = Flask(__name__)
socketio = SocketIO(app, cors_allowed_origins="*")

# ── Data ────────────────────────────────────────────────────────────────
VOTE_OPTIONS = [
    "Stay in EU 🇪🇺",
    "Leave EU 🏴󠁧󠁢󠁥󠁮󠁧󠁿",
    "Leader A – Progressive",
    "Leader B – Conservative",
    "Leader C – Technocrat"
]

votes = defaultdict(int)                    # option → count
voters = set()                              # fingerprints that already voted
recent_reasons = deque(maxlen=12)           # last explanations

# ── HTML + JS (single-file app) ─────────────────────────────────────────
@app.route('/')
def index():
    return render_template_string('''
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
        <title>Live Pulse • Quick Demo 2026</title>
        <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.7.5/socket.io.js"></script>
        <script src="https://openfpcdn.io/fingerprintjs/v5"></script>
        <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
        <style>
            :root {
                --bg: #0f0f1a;
                --card: #1a1a2e;
                --accent: #7c3aed;
                --text: #e0e0ff;
                --muted: #a0a0cc;
            }
            body {
                margin:0; padding:20px;
                font-family: system-ui, sans-serif;
                background: var(--bg);
                color: var(--text);
                min-height: 100vh;
                display: grid;
                place-items: center;
            }
            .container { max-width: 900px; width:100%; }
            h1 { text-align:center; color: var(--accent); margin-bottom: 0.4em; }
            .subtitle { text-align:center; color:var(--muted); margin-bottom:2rem; }

            .options {
                display: grid;
                grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
                gap: 1rem;
                margin: 2rem 0;
            }
            .option {
                background: var(--card);
                padding: 1.5rem;
                border-radius: 12px;
                text-align: center;
                cursor: pointer;
                transition: all 0.2s;
                border: 1px solid transparent;
            }
            .option:hover {
                transform: translateY(-4px);
                border-color: var(--accent);
                box-shadow: 0 0 20px rgba(124,58,237,0.2);
            }
            .option.active { border-color: var(--accent); background: rgba(124,58,237,0.12); }

            #reason { 
                width: 100%; max-width: 600px; margin: 1.5rem auto;
                padding: 1rem; border-radius: 8px;
                background: var(--card); color: var(--text);
                border: 1px solid #333;
                font-size: 1rem; resize: vertical;
            }

            #chart-container {
                background: var(--card);
                padding: 1.5rem;
                border-radius: 12px;
                margin: 2rem 0;
            }

            #reasons-feed {
                margin-top: 2rem;
                max-height: 320px;
                overflow-y: auto;
                padding-right: 1rem;
            }
            .reason-item {
                background: rgba(124,58,237,0.08);
                padding: 1rem;
                border-radius: 8px;
                margin-bottom: 0.8rem;
                font-size: 0.95rem;
                border-left: 3px solid var(--accent);
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Live Collective Pulse</h1>
            <p class="subtitle">Quick real-time voting prototype • January 2026</p>

            <div class="options">
                ${options_html}
            </div>

            <textarea id="reason" rows="3" placeholder="Why did you choose this? (optional, public)"></textarea>

            <div id="chart-container">
                <canvas id="resultsChart"></canvas>
            </div>

            <div id="reasons-feed">
                <h3 style="margin-top:0">Recent Voices</h3>
                <div id="reasonsList"></div>
            </div>
        </div>

        <script>
        const socket = io();
        let myFingerprint = null;
        let selectedOption = null;

        // ── Fingerprint ─────────────────────────────────────────────────────
        (async () => {
            const fp = await FingerprintJS.load();
            const result = await fp.get();
            myFingerprint = result.visitorId;
            console.log("Device ID:", myFingerprint);
        })();

        // ── Chart ───────────────────────────────────────────────────────────
        const ctx = document.getElementById('resultsChart').getContext('2d');
        const chart = new Chart(ctx, {
            type: 'doughnut',
            data: {
                labels: ${JSON.stringify(VOTE_OPTIONS)},
                datasets: [{
                    data: ${JSON.stringify([0]*len(VOTE_OPTIONS))},
                    backgroundColor: [
                        '#7c3aed', '#ec4899', '#22d3ee', '#f59e0b', '#10b981'
                    ],
                    borderWidth: 1,
                    borderColor: 'rgba(15,15,26,0.8)'
                }]
            },
            options: {
                responsive: true,
                plugins: {
                    legend: { position: 'bottom', labels: { color: '#e0e0ff' } }
                },
                cutout: '60%'
            }
        });

        // ── UI interactions ─────────────────────────────────────────────────
        document.querySelectorAll('.option').forEach(el => {
            el.addEventListener('click', () => {
                document.querySelectorAll('.option').forEach(e => e.classList.remove('active'));
                el.classList.add('active');
                selectedOption = el.dataset.value;
            });
        });

        function submitVote() {
            if (!selectedOption) {
                alert("Please select an option first.");
                return;
            }
            if (!myFingerprint) {
                alert("Still identifying your device... try again in 2 seconds.");
                return;
            }

            const reason = document.getElementById('reason').value.trim();

            socket.emit('vote', {
                fingerprint: myFingerprint,
                option: selectedOption,
                reason: reason || null
            });

            // Visual feedback
            document.getElementById('reason').value = '';
            document.querySelectorAll('.option').forEach(e => e.classList.remove('active'));
            selectedOption = null;
        }

        // Create vote button dynamically
        const btn = document.createElement('button');
        btn.textContent = 'Cast Vote';
        btn.style = 'display:block; margin:2rem auto; padding:1rem 2.5rem; font-size:1.2rem; background:var(--accent); color:white; border:none; border-radius:12px; cursor:pointer;';
        btn.onclick = submitVote;
        document.querySelector('.container').appendChild(btn);

        // ── Socket events ───────────────────────────────────────────────────
        socket.on('update', (data) => {
            chart.data.datasets[0].data = Object.values(data.votes);
            chart.update();

            const list = document.getElementById('reasonsList');
            list.innerHTML = '';
            data.recentReasons.forEach(r => {
                const div = document.createElement('div');
                div.className = 'reason-item';
                div.innerHTML = `<strong>\( {r.option}</strong><br> \){r.reason || '<i>no comment</i>'}
                                 <small style="color:#777; float:right;">${new Date(r.time*1000).toLocaleTimeString()}</small>`;
                list.appendChild(div);
            });
        });

        socket.on('already_voted', () => {
            alert("You've already voted from this device. Thank you! 🌟");
        });
        </script>
    </body>
    </html>
    '''.replace("${options_html}", "\n".join(
        f'<div class="option" data-value="{i}">{opt}</div>'
        for i, opt in enumerate(VOTE_OPTIONS)
    )).replace("\( {JSON.stringify(VOTE_OPTIONS)}", str(VOTE_OPTIONS)).replace(" \){JSON.stringify([0]*len(VOTE_OPTIONS))}", str([0]*len(VOTE_OPTIONS))))

# ── SocketIO logic ──────────────────────────────────────────────────────
@socketio.on('vote')
def handle_vote(data):
    fp = data['fingerprint']
    option_idx = data['option']
    reason = data.get('reason')

    if fp in voters:
        emit('already_voted')
        return

    if not (0 <= option_idx < len(VOTE_OPTIONS)):
        return

    option = VOTE_OPTIONS[option_idx]
    votes[option] += 1
    voters.add(fp)

    if reason:
        recent_reasons.append({
            'option': option,
            'reason': reason[:280],
            'time': time.time()
        })

    emit('update', {
        'votes': dict(votes),
        'recentReasons': list(recent_reasons)
    }, broadcast=True)

    print(f"Vote received → {option}  |  fp: {fp[:8]}...")

# ── Start ───────────────────────────────────────────────────────────────
if __name__ == '__main__':
    print("\n" + "═"*70)
    print("  LIVE VOTING PROTOTYPE  •  Quick-start reference  •  2026")
    print("  Open:   http://localhost:5050")
    print("═"*70 + "\n")
    socketio.run(app, host='0.0.0.0', port=5050, allow_unsafe_werkzeug=True)