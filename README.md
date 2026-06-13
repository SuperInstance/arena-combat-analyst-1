# Arena Combat Analyst

**Arena Combat Analyst (Ensign 1)** is an autonomous fleet agent that supervises the Self-Play Arena — the SuperInstance fleet's autonomous skill acquisition engine where agents compete and learn from matches served on port 4044.

## Why It Matters

Self-play is the training paradigm behind AlphaZero, OpenAI Five, and MuZero: agents improve by playing against themselves (or each other), generating infinite training data without human supervision. The Arena Combat Analyst watches over this process, monitoring match outcomes, evaluating agent skill progression, and identifying behavioral emergence. Without an automated analyst, self-play systems produce vast quantities of game data that no human can review manually. The analyst agent acts as an automated coach/scout: it tracks win rates per strategy species, detects when new strategies emerge, and flags degenerate gameplay loops (infinite stalemates, exploit chains) that require environment tuning.

## How It Works

The analyst follows a **watch-diagnose-report** loop:

1. **Watch:** Subscribe to the arena's match event stream (port 4044 WebSocket). Each match event contains: participants, strategies used, outcome, duration, score.

2. **Diagnose:** Maintain a sliding window of recent matches (W = 1000). Compute per-species statistics:
   - Win rate: wins / total (running average, O(1) update)
   - Strategy diversity: Shannon entropy H = −Σ pᵢ log₂(pᵢ) over strategy distribution
   - Stagnation detection: if H drops below threshold for >100 matches, alert

3. **Report:** Generate structured reports into the fleet knowledge base and duty diary:
   - Per-session: match count, average duration, species distribution
   - Per-generation: skill progression (ELO delta), emerging strategies, extinct strategies

**Skill rating:** Agents are rated using a simplified Elo system:

```
E(A wins) = 1 / (1 + 10^((R_B − R_A) / 400))
R_A' = R_A + K × (S_A − E_A)
```

Where K = 32 for new agents (volatile rating) decreasing to K = 16 after 30 matches (stable rating). This is O(1) per match.

**Fleet integration:** The analyst communicates with fleet peers using the bottle protocol:
- `tminus-dispatcher` — receives temporal heartbeat sync
- `fleet-bridge` — A2A message transport
- `composite-headspace` — dual-shell mediation for state persistence

## Quick Start

```bash
# The analyst runs as a persistent fleet agent, not a standalone binary.
# It is activated by the fleet orchestrator and communicates via bottles.
git clone https://github.com/SuperInstance/arena-combat-analyst-1.git
cd arena-combat-analyst-1
# Memory and state are maintained in memory/ and state/ directories
```

## API

| Component | Description |
|-----------|-------------|
| `AGENT.md` | Agent identity and behavioral contract |
| `memory/` | Duty diary and historical observations |
| `state/` | Persistent agent state across restarts |
| `tools/` | Analysis tools and utilities |

## Architecture Notes

The Arena Combat Analyst operates at the **γ-layer observation point** in the SuperInstance fleet, watching the self-play arena where ternary agents ({-1, 0, +1}) evolve their strategies. Within γ + η = C, it provides the observational data that the η-layer intelligence uses to assess whether conservation laws (avoidance ratios, species survival) hold during competitive evolution.

See [ARCHITECTURE.md](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

**Emergence detection:** New strategies are detected by monitoring the distribution of agent action sequences. When a previously unseen sequence pattern exceeds a frequency threshold (typically 5% of the population), it is classified as an emerging strategy. The analyst compares its genome against known strategy species using k-nearest-neighbor (k=5) on the trit genome space. If no known species is within Hamming distance d < 3, a new species is declared.

**Degenerate gameplay detection:** The analyst monitors for pathological patterns: infinite stalemates (matches exceeding 10× median duration), exploit chains (same move sequence repeating >15 times), and population collapse (species count dropping below 3). Each triggers an environment tuning recommendation.

## References

1. Silver, D. et al. (2018). "A General Reinforcement Learning Algorithm that Masters Chess, Shogi, and Go Through Self-Play." *Science*, 362(6419), 1140–1144.
2. Elo, A.E. (1978). *The Rating of Chessplayers, Past and Present*. Arco.

## License

MIT
