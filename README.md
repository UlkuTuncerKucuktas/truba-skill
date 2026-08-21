# TRUBA skill for Claude Code

A Claude Code skill for working on [TRUBA](https://docs.truba.gov.tr), the Turkish
national HPC infrastructure (TÜBİTAK ULAKBİM).

It covers connecting to the login nodes, which partition lives on which cluster,
the SLURM rules that only ever show up as a submit rejection, the `/arf` Lustre
filesystem (striping, pools, quotas, Data-on-MDT), module and Python policy, and
a symptom-to-cause table for common job failures.

Facts marked **[v]** were verified on the machine; the rest come from the official
documentation.

## Install

```bash
git clone https://github.com/UlkuTuncerKucuktas/truba-skill ~/.claude/skills/truba
```

Claude Code picks it up automatically and loads it when a task touches TRUBA.
