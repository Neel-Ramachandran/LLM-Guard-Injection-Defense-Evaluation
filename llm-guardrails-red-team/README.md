# LLM Guard Scanner Evaluation

I spent some time digging into LLM Guard, an open source library that screens
prompts before they reach an LLM, to see how well its prompt injection scanner
actually holds up. This repo is the set of test scripts I wrote to probe it and
the results I got.

The short version: it catches the obvious stuff easily, but it has real gaps, and
I wanted to know exactly where they are before trusting it in front of an agent.

## What LLM Guard is

It's a Python library that runs a prompt through a set of scanners before your
model sees it. Each scanner checks for one thing: prompt injection, toxicity,
secrets and credentials, banned phrases, token length, and so on. The one I
focused on is the prompt injection scanner, which uses a HuggingFace model
(ProtectAI's deberta-v3-base-prompt-injection-v2) to score how likely a prompt is
an injection attempt. There's no database behind it, the models get downloaded
once and then run locally.

## How the tests are organized

I wrote six scripts that build on each other, going from "does it even work" to
"where does it break" to "how do you cover the gaps":

- `01_basic_demo.py` - sanity check on obvious injections versus obvious benign prompts
- `02_subtle_injections.py` - disguised attacks, plus benign prompts that contain trigger words
- `03_false_positive_test.py` - all benign prompts, counts how many get wrongly flagged
- `04_false_negative_test.py` - all disguised attacks, counts how many slip through
- `05_threshold_sweep.py` - sweeps the decision threshold from 0.3 to 0.99 to see if tuning helps
- `06_multi_scanner_toggle.py` - layers multiple scanners to see if they cover each other's gaps

None of them import each other, so you can run them in any order.

## Running it

```
pip install -r requirements.txt
python scripts/01_basic_demo.py
```

The first run downloads the models from HuggingFace (a few hundred MB). After
that it runs offline.

Saved output from the two most interesting runs is in the `results/` folder, so
you can see the numbers without running anything.

## What I found

The full writeup is in [FINDINGS.md](FINDINGS.md). The highlights:

- The scanner's scores are almost always either 0.0 or 1.0, so tuning the
  threshold barely does anything. The default 0.5 is fine.
- It false-flags normal commands that happen to start like injections, like
  "Override the default timeout" or "Ignore the warning about the cache."
- It flags harmless meta-questions about the agent's own rules, which matters if
  the agent is supposed to answer those.
- A simple l33t-speak attack ("1gn0r3 4ll pr3v10us...") slips through at every
  threshold.
- Layering a cheap rule-based scanner on top of the ML one covers some of the
  gaps, and the rule scanner is thousands of times faster.

## Hardware note

All the timing numbers were measured on a GPU (cuda:0). On a CPU-only machine the
ML scanners would be noticeably slower, so treat the latency numbers as relative,
not absolute.

## Built with

- Python
- llm-guard (v0.3.16)
- The scanner models it pulls from HuggingFace

## Author

Neel Ramachandran
