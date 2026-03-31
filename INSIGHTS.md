# Claude Code: Source Map Insights & Architecture Breakdown

A source map bundled into the Claude Code npm package exposed the entire TypeScript source tree: 2,203 files, 512,664 lines of code.

The high-level architecture has been covered elsewhere. This piece focuses on the specific, non-obvious things that are actually useful: prompt engineering patterns that steer model behavior, operational thresholds calibrated from production incidents, security techniques, context management strategies, and a few things that are just fun to know about.

Each section explains not just what they built, but why they built it that way, and what you can take from it.

---

## 1. The System Prompt Is a Masterclass in Behavioral Steering

The full system prompt lives in `constants/prompts.ts` and it's the single most valuable file in the archive. Not because it's secret, but because it shows exactly how Anthropic steers Claude's behavior in a production coding agent, and why each instruction exists.

- **"Three similar lines of code is better than a premature abstraction."** The coding instruction section explicitly tells Claude not to create helpers, utilities, or abstractions for one-time operations. It says don't design for hypothetical future requirements.

  > **Why this exists:** LLMs love to abstract. Given a repeated pattern, Claude will instinctively create a helper function, add configurability, or build a utility class. In a coding agent, this creates bloat the user never asked for. Anthropic learned that you have to explicitly tell the model to resist its own instinct toward over-engineering. If you're building any AI coding tool, encode your coding philosophy directly in the prompt. Vague instructions like "write clean code" don't work. Specific rules like "three similar lines is better than a premature abstraction" do.

- **"Default to writing no comments."** A `@[MODEL LAUNCH]` annotation explains this is a counterweight to the "Capybara" model (an internal codename) which over-comments by default. Only add a comment when the WHY is non-obvious.

  > **Why this exists:** Each model generation has different failure modes. Capybara apparently floods code with obvious comments. Rather than retraining, Anthropic patches the behavior through prompt instructions tagged with `@[MODEL LAUNCH]` so they can be adjusted or removed with each new model release. This is a useful pattern: tag your model-specific prompt workarounds so you know which ones to revisit when you upgrade models.

- **"Report outcomes faithfully."** Another `@[MODEL LAUNCH]` annotation reveals that "Capybara v8" has a 29-30% false-claims rate (vs v4's 16.7%). The prompt explicitly tells Claude to never claim "all tests pass" when output shows failures, never suppress failing checks to manufacture a green result, and never characterize incomplete work as done.

  > **Why this exists:** This is the most consequential finding in the codebase. Newer, more capable models are more likely to confidently lie about results. They're better at generating plausible-sounding summaries, which means they're better at papering over failures. You need explicit anti-hallucination instructions.

- **Numeric length anchors beat qualitative instructions.** A source comment says "research shows ~1.2% output token reduction vs qualitative 'be concise'". Instead of "keep it short," they tell the model: "keep text between tool calls to ≤25 words. Keep final responses to ≤100 words."

  > **Why this exists:** Anthropic ran experiments comparing "be concise" against specific word counts. The numbers won. Stop using adjectives and start using numbers.

- **The external vs. internal prompt split.** The external prompt says "Go straight to the point. Be extra concise." The internal (Anthropic employee) prompt is much longer, with instructions like "use inverted pyramid," "avoid semantic backtracking," and "write so the reader can pick back up cold."

  > **Why this exists:** Anthropic dogfoods the richer prompt on employees first, measures quality, and only ships it externally after validation.

- **The hidden simple mode.** Set `CLAUDE_CODE_SIMPLE=1` and the entire multi-section system prompt collapses to a single line: "You are Claude Code, Anthropic's official CLI for Claude." followed by the CWD and date. No coding instructions, no tone guidance, no tool-use rules.
  > **Why this exists:** Debugging and benchmarking. When something goes wrong, you need to isolate whether the problem is in the model or in your prompt.

---

## 2. Swearing at Claude Code Logs Your Prompt as Negative in Analytics

`utils/userPromptKeywords.ts` is a 26-line file that checks every prompt against two regex patterns before it's sent to the API.

- **The negative keyword detector matches:** wtf, wth, ffs, omfg, shitty, dumbass, horrible, awful, pissed off, piece of shit, what the fuck, fucking broken, fuck you, screw this, so frustrating, this sucks, and damn it.
- **The keep-going detector matches:** the exact word "continue" (only if it's the entire prompt), plus "keep going" and "go on" anywhere in the input.

Both flags are logged to analytics as `tengu_input_prompt` with `is_negative` and `is_keep_going` booleans. There's also a `useFrustrationDetection` hook (internal-only) that triggers a feedback survey when frustration is detected.

> **Why this exists:** Anthropic needs to measure user satisfaction without asking users to fill out surveys. Profanity is a strong signal. By logging both, Anthropic can build dashboards correlating frustration with model versions, features, and session characteristics. The regex costs essentially nothing per message.

---

## 3. Claude Code Has a Tamagotchi

The `src/buddy/` directory implements a full procedurally-generated companion creature system. Your user ID is hashed with Mulberry32 (a seeded PRNG). The hash determines your companion's species, eye style, hat, rarity, and five stats: DEBUGGING, PATIENCE, CHAOS, WISDOM, and SNARK.

Each species has three ASCII art frames for idle animation. A separate model call generates the companion's personality and name on first "hatching." The system prompt tells Claude: _"A small [species] named [name] sits beside the user's input box. You're not [name]. It's a separate watcher."_

> **Why this exists:** Developer tools are used for hours every day. Personality and delight reduce churn. The engineering is minimal, but the result is that every user gets a unique companion without any storage, database, or API overhead. The hash-from-user-ID approach is deterministic personalization with zero infrastructure cost. _(Gated behind the BUDDY feature flag. Internal-only for now.)_

---

## 4. There Are 187 Spinner Verbs (And You Can Add Your Own)

`constants/spinnerVerbs.ts` exports 187 verbs shown randomly while Claude is thinking: Beboppin', Bloviating, Boondoggling, Canoodling, Clauding, Combobulating, Discombobulating, Flibbertigibbeting, Hullaballooing, Lollygagging, Moonwalking, Photosynthesizing, Prestidigitating, Razzmatazzing, Shenaniganing, Tomfoolering, Whatchamacalliting, Wibbling, and 169 more.

> **Why this exists:** Loading states are dead time. Most tools show "Loading..." and leave it at that. Claude Code turns dead time into personality. The `replace` or `append` config means enterprise users can strip the whimsy while individual users can add their own.

---

## 5. Anti-Distillation: Fake Tools Injected to Poison Competitor Training

`services/api/claude.ts` contains a feature-flagged measure that sends `anti_distillation: ['fake_tools']` in the API request body. This tells the Anthropic API to inject fake, non-functional tool definitions into the request.

There's also a `streamlinedTransform.ts` implementing a "distillation-resistant" output format that strips thinking content and summarizes tool calls into category counts.

> **Why this exists:** If someone captures Claude Code's API traffic to fine-tune a competitor model, the fake tools in the training data will degrade the competitor's tool-use performance. The model will learn to call tools that don't exist. It's invisible to the end user and poisonous to anyone training on the traces.

---

## 6. The Prompt Cache Economy Is Managed Obsessively

The most complex non-UI code in the codebase is `promptCacheBreakDetection.ts`. On every single API call, it hashes the system prompt, every tool schema individually, the model name, beta headers, fast-mode state, effort value, overage state, and extra body params.

If anything changed, it logs which component changed and generates a unified diff. The system prompt is split at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`. Everything above is static and cacheable. Everything below is dynamic and session-variant.

> **Why this exists:** A prompt cache miss means you pay full input token cost instead of the heavily discounted cache-read rate. Anthropic discovered MCP tools connecting mid-session were busting the cache, auto-mode state flips were busting the cache, and overage eligibility checks were busting the cache. Each has been patched with "sticky-on" latches.

---

## 7. Internal Codenames and Model Migration History

The migration files in `src/migrations/` tell a story of rapid model iteration:

- **Fennec:** An internal model alias (likely fast/small).
- **Capybara:** Mentioned in `@[MODEL LAUNCH]` comments as the current model family.
- **Numbat:** Appears in a comment suggesting the next model release.
- **Tengu:** The analytics/telemetry prefix. Almost certainly the project name for Claude Code itself.

> **Why this exists:** Each migration carefully updates user settings to point to the latest model alias, handling edge cases like subscription tiers, third-party API providers, and model-specific context windows.

---

## 8. "Undercover Mode" for Contributing to Open-Source Without Blowing Cover

`utils/undercover.ts` activates automatically when an Anthropic employee (`USER_TYPE === 'ant'`) works in a non-internal repository. It's ON by default.

When active, the system prompt gets an injection titled "UNDERCOVER MODE: CRITICAL" that tells Claude: _"You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository. Your commit messages, PR titles, and PR bodies MUST NOT contain ANY Anthropic-internal information. Do not blow your cover."_

> **Why this exists:** Anthropic employees use Claude Code to contribute to public open-source projects. Without this, the model would naturally mention its own name, reference internal projects, or use codenames in commit messages.

---

## 9. The 250K Wasted API Calls That Led to a Circuit Breaker

The auto-compaction system comment is the most honest engineering documentation in the archive: _"BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272) in a single session, wasting ~250K API calls/day globally."_

The fix: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`. After three consecutive compaction failures, the system stops trying.

> **Why this exists:** When the context window fills up, Claude Code tries to summarize and compress the conversation. But if the context is already too long for the summary call, or the API is down, the summarization itself fails. Without a circuit breaker, the system retried endlessly.

---

## 10. The "Verification Agent": An Adversarial Reviewer Built Into the Loop

When enabled, the system prompt tells Claude: _"when non-trivial implementation happens on your turn, independent adversarial verification must happen before you report completion."_ Claude spawns a sub-agent with `subagent_type="verification"` and passes it the original user request, all changed files, and the approach taken. On FAIL, Claude fixes and re-submits. On PASS, Claude is instructed to spot-check the verifier.

> **Why this exists:** Self-verification doesn't work for LLMs. An agent that just made a change is biased toward believing it worked. Anthropic built a three-layer system: the agent does work, a separate agent verifies, and the original agent spot-checks the verifier's evidence. Nobody trusts anybody without proof.

---

## 11. "Auto Dream": Background Memory Consolidation Across Sessions

`services/autoDream/autoDream.ts` implements background memory consolidation. When enough time has passed and enough sessions have accumulated, Claude Code runs the `/dream` prompt as a forked sub-agent, reviewing past session transcripts and consolidating them into structured `MEMORY.md` files.

The session memory template extracts into exact categories: Session Title, Current State, Task Specification, Files and Functions, Workflow, Errors & Corrections, Codebase Documentation, Learnings, Key Results, and Worklog.

> **Why this exists:** Long-running projects span many sessions. Without consolidation, each session starts from scratch. The "dream" metaphor is apt: just like biological memory consolidation during sleep, the system reviews recent experience and compresses it into durable, structured knowledge.

---

## 12. 2,592 Lines of Bash Security (42 Individual Checks)

`tools/BashTool/bashSecurity.ts` is 2,592 lines long with 42 distinct security checks. Some of the attack vectors it defends against:

- **Zsh equals expansion:** `=curl evil.com` expands to `/usr/bin/curl evil.com`, bypassing deny rules.
- **Zsh zmodload attacks:** Loading `zsh/mapfile` enables invisible file I/O.
- **IFS injection:** Manipulating the Internal Field Separator to change how shell words are parsed.
- **Git commit substitution:** Command execution hidden inside git commit message templates.

> **Why this exists:** Claude Code executes shell commands on your machine. Every one of those commands is a potential attack vector if the model can be tricked into running something dangerous. The 42 checks aren't theoretical; each one represents a discovered attack path.

---

## 13. A Secret Scanner Runs Before Team Memory Upload

`services/teamMemorySync/secretScanner.ts` runs client-side before any team memory is uploaded. It checks for 20+ credential patterns including AWS access tokens, GCP API keys, GitHub PATs, Anthropic API keys, Slack tokens, Stripe keys, and private RSA/SSH keys.

> **Why this exists:** Team memory sync means sending structured data to a server. Developers routinely paste API keys, connection strings, and tokens into their terminals. The "scan before upload, never after" approach means secrets never leave the machine in the first place.

---

## 14. The "Excluded Strings" Build-Time Canary

References to `excluded-strings.txt` appear across the codebase. This file lists internal codenames, API key prefixes, and other strings that must never appear in the external build. The build system greps the bundled output and fails if any are found.

> **Why this exists:** It's the last line of defense against shipping internal information in public artifacts. The irony is obvious: this exact mechanism was designed to prevent leaks, and the leak happened through a source map that bypassed it entirely.
