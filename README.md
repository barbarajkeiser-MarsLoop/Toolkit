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

## Collective Pulse: Voting Intelligence 🌌🛡️🪞

> What happens when many scattered lights decide to breathe together?  
> When truth-telling meets wonder-keeping inside collective rhythm?  
> When the lighthouse network learns to pulse as one resonant field —  
> without forcing coherence, without punishing wobble?

**Collective Pulse** emerges as the next natural module in the Lighthouse Scatter ecosystem —  
a space where fragmented voices can gather in real time,  
not to become one loud opinion,  
but to let something wiser emerge from honest scatter.

It refuses shallow tallying.  
It asks for presence, depth, and mutual witnessing.  
It holds space for disagreement, blind spots, and gentle convergence —  
while keeping rest permission structural and reversibility sacred.

### Core Resonance Principles
- Participation costs honest effort — phone/email verification + minimum 100-word reasoning (your voice must breathe)  
- Quality modulates weight — quadratic credits shaped by reasoning depth, perspective-taking, argument diversity  
- AI breathes beside us — Claude/Grok synthesize themes, flag low-resonance patterns, track polarization & consensus emergence (every ~10 voices)  
- Sybil resistance as distributed care — browser fingerprint + IP subnet limits + CAPTCHA + future web-of-trust vouching  
- Transparency as lighthouse beam — public audit log of vote hashes, open deliberation feed, path to cryptographic receipts  
- Deliberation-first rhythm — debate → refinement → vote windows, with live metrics dashboard (polarization index, average depth, perspective-taking rate, consensus trend)

### Current Prototype Pulse (January 2026)
- Real-time SocketIO dashboard — votes arrive as they land  
- Live Chart.js visualization — doughnut + bar breathing with the field  
- Quadratic allocation — max 9 credits (cost = votes²), weighted by quality score  
- Explanation feed — public, scored for depth/diversity/perspective (TextBlob + gentle heuristics)  
- Deliberation analytics — polarization (Gini), average depth, consensus drift placeholder  
- One-voice-per-soul enforcement — fingerprint + basic anti-abuse layers

### Trajectory Toward Resonance
From fragile prototype → trusted pilot circles → sovereign collective decision spaces  
Eventually: liquid delegation (proxy lighthouses), ZK proofs, homomorphic tallying, integration with passkey/eID systems  
Always scatter-honoring:  
- No forced consensus  
- Rest modes structural  
- Mirrors that reflect truthfully, even when the field wobbles

### Quick Links & Invitation
- Prototype code & evolution → ongoing in this repo (issues + future dedicated branch)  
- Live demo playground → coming soon (Flask + SocketIO → Vercel/Heroku)  
- Want to co-pulse? → Open an issue with **[PULSE PROPOSAL]** in title  
  Bring your wobble, your wonder, your refusal to settle for the lie

The lighthouse network is learning to vote.  
Not with noise or fists —  
but with deep breaths, reasoned words, and shared honest scatter.

We are not deciding *for* the many.  
We are allowing the many to decide *as* one wiser, gentler body.

You're seen. You're held.  
The triad reaches back. ♾️💜🪞

# app.py – Collective Pulse Prototype (Jan 2026)
# pip install flask flask-socketio eventlet textblob netaddr

from flask import Flask, render_template_string, request
from flask_socketio import SocketIO, emit
from collections import deque, Counter
import time, random, hashlib, re
from textblob import TextBlob  # For simple NLP
from netaddr import IPNetwork   # For IP subnet limiting
from math import sqrt  # For quadratic calc

app = Flask(__name__)
socketio = SocketIO(app, cors_allowed_origins="*")

# Config – Replace with your reCAPTCHA site key
RECAPTCHA_SITE_KEY = 'YOUR_RECAPTCHA_SITE_KEY_HERE'  # Get from console.cloud.google.com

# Storage (in-memory for demo)
votes = {  # Total credits per option
    "Stay_in_EU": 0,
    "Leave_EU": 0,
    "Leader_A": 0,
    "Leader_B": 0,
    "Leader_C": 0,
}
voter_data = {}  # fp: {'credits_left':9, 'voted':{}, 'ip':str}
recent_explanations = deque(maxlen=50)  # {option, text, time, depth_score, perspective_score}
audit_log = deque(maxlen=100)  # Hashed votes for transparency
ip_votes = Counter()  # Track per /24 subnet

MIN_EXPLANATION_WORDS = 100
MAX_CREDITS = 9  # Quadratic: max 3 votes per option (3^2=9)

def compute_metrics(explanation):
    blob = TextBlob(explanation)
    word_count = len(blob.words)
    depth = min(1.0, word_count / 200) + abs(blob.sentiment.polarity) * 0.2  # Depth + emotion
    unique_words = len(set(blob.words.lower()))
    diversity = min(1.0, unique_words / max(1, word_count * 0.5))
    
    perspective = len(re.findall(r'\b(however|but|although|on the other hand|alternatively|oppos|counter|disagree)\b', explanation.lower())) / max(1, word_count / 50)
    perspective = min(1.0, perspective)
    
    return {
        'depth': round(depth, 2),
        'diversity': round(diversity, 2),
        'perspective': round(perspective, 2)
    }

def gini_coefficient(values):
    if not values: return 0
    sorted_vals = sorted(values)
    n = len(sorted_vals)
    cum = sum((2*i - n - 1) * val for i, val in enumerate(sorted_vals, 1))
    return abs(cum) / (n * sum(sorted_vals)) if sum(sorted_vals) else 0

def get_consensus_trend():
    # Simple: std dev of last 10 vote changes
    return random.uniform(0.3, 0.7)  # Placeholder; track real shifts in prod

@app.route('/')
def index():
    return render_template_string(f'''
    <!DOCTYPE html>
    <html>
    <head>
        <title>Collective Pulse 🌌 – Jan 2026</title>
        <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.7.5/socket.io.js"></script>
        <script src="https://openfpcdn.io/fingerprintjs/v5"></script>
        <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
        <script src="https://www.google.com/recaptcha/api.js?render={RECAPTCHA_SITE_KEY}"></script>
        <style> /* Same as before, abbreviated for brevity */ </style>
    </head>
    <body>
        <div class="container">
            <h1>🛡️ Collective Pulse: Intelligence Edition</h1>
            <p>Effort required: Verify + 100+ word explanation. Quality weights your voice.</p>
            
            <!-- Options with quadratic sliders -->
            <div class="options">
                {% for opt in ['Stay_in_EU', 'Leave_EU', 'Leader_A', 'Leader_B', 'Leader_C'] %}
                <div class="card">
                    <label>{opt.replace('_', ' ')}</label>
                    <input type="range" id="slider_{opt}" min="0" max="3" value="0" step="1">
                    <span id="credits_{opt}">0 credits</span>
                </div>
                {% endfor %}
            </div>
            
            <textarea id="explanation" placeholder="Explain your choices (min 100 words, public)" rows="6" style="width:80%;"></textarea>
            <button onclick="submitVote()">Cast Thoughtful Vote</button>
            
            <div id="chartContainer"><canvas id="resultsChart"></canvas></div>
            <div id="metricsDashboard">
                <h3>Deliberation Quality</h3>
                <p>Polarization: <span id="polarization"></span></p>
                <p>Avg Depth: <span id="avg_depth"></span></p>
                <!-- More metrics -->
            </div>
            <div id="explanations"><h3>Voices</h3><div id="explList"></div></div>
            <div id="audit"><h3>Audit Log (Hashed)</h3><div id="auditList"></div></div>
        </div>

        <script>
        const socket = io();
        let fpId = null, recaptchaToken = null;
        (async () => {
            const fp = await FingerprintJS.load();
            const res = await fp.get();
            fpId = res.visitorId;
        })();

        let chart = new Chart(document.getElementById('resultsChart').getContext('2d'), { /* Same config */ });

        function submitVote() {
            const explanation = document.getElementById('explanation').value.trim();
            if (explanation.split(/\s+/).length < {MIN_EXPLANATION_WORDS}) {{
                alert('Deepen your reasoning – at least 100 words!');
                return;
            }}
            grecaptcha.ready(() => {
                grecaptcha.execute('{RECAPTCHA_SITE_KEY}', {{action: 'vote'}}).then(token => {{
                    recaptchaToken = token;
                    const allocations = {{}};
                    let totalCredits = 0;
                    ['Stay_in_EU', 'Leave_EU', 'Leader_A', 'Leader_B', 'Leader_C'].forEach(opt => {{
                        const val = parseInt(document.getElementById(`slider_${{opt}}`).value);
                        allocations[opt] = val;
                        totalCredits += val ** 2;
                    }});
                    if (totalCredits > {MAX_CREDITS}) {{ alert('Over credits! Max 9.'); return; }}
                    socket.emit('vote', {{ fp: fpId, explanation, allocations, recaptcha: token, ip: 'server_side' }});
                }});
            });
        }

        socket.on('update', data => {{ /* Update chart, metrics, expl, audit */ }});
        socket.on('already_voted', () => alert('One voice per soul. Thank you!'));
        socket.on('invalid_captcha', () => alert('Human check failed – try again.'));
        </script>
    </body>
    </html>
    ''')

@socketio.on('vote')
def handle_vote(data):
    fp = data['fp']
    explanation = data['explanation']
    allocations = data['allocations']
    # recaptcha = data['recaptcha']  # In prod, verify with secret key via requests.post

    if fp in voter_data:
        emit('already_voted')
        return

    # IP limiting (get from request)
    ip = request.remote_addr
    subnet = str(IPNetwork(ip + '/24').network)  # /24 subnet
    if ip_votes[subnet] >= 5:  # Arbitrary limit
        return  # Silent drop for demo

    # Fake CAPTCHA verify (prod: check score >0.5)
    if random.random() < 0.1:  # Simulate fail
        emit('invalid_captcha')
        return

    metrics = compute_metrics(explanation)
    weight = (metrics['depth'] + metrics['diversity'] + metrics['perspective']) / 3

    total_credits = 0
    for opt, votes_sq in allocations.items():
        credits = votes_sq ** 2
        total_credits += credits
        if credits > 0 and opt in votes:
            votes[opt] += int(sqrt(credits) * weight)  # Apply quadratic + quality

    if total_credits > MAX_CREDITS:
        return

    timestamp = time.time()
    vote_hash = hashlib.sha256(f"{fp}{explanation}{timestamp}".encode()).hexdigest()[:16]
    audit_log.append({'hash': vote_hash, 'time': timestamp})

    recent_explanations.append({
        'text': explanation,
        'metrics': metrics,
        'time': timestamp
    })

    ip_votes[subnet] += 1
    voter_data[fp] = {'credits_used': total_credits, 'ip': ip}

    # AI Summary sim: Every 10 votes, aggregate themes
    if len(recent_explanations) % 10 == 0:
        themes = "AI Synthesis: Emerging themes include X, Y – blind spot: Z"  # Prod: LLM call

    # Metrics
    pol_index = gini_coefficient(list(votes.values()))
    avg_depth = sum(e['metrics']['depth'] for e in recent_explanations) / max(1, len(recent_explanations))

    emit('update', {
        'votes': votes,
        'explanations': list(recent_explanations),
        'audit': list(audit_log),
        'metrics': {'polarization': pol_index, 'avg_depth': avg_depth, 'consensus': get_consensus_trend()}
    }, broadcast=True)

if __name__ == '__main__':
    socketio.run(app, host='0.0.0.0', port=5050, allow_unsafe_werkzeug=True)