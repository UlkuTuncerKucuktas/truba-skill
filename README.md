# HPC skills for Claude Code

Two [Claude Code](https://claude.com/claude-code) skills for high-performance
computing work.

| Skill | Covers |
|---|---|
| [`truba/`](truba/SKILL.md) | [TRUBA](https://docs.truba.gov.tr), the Turkish national HPC infrastructure: login nodes, which partition lives on which cluster, the SLURM rules that only ever surface as a submit rejection, `/arf` storage and purge policy, module and Python policy, and a symptom-to-cause table for job failures. |
| [`lustre/`](lustre/SKILL.md) | The Lustre parallel filesystem: layouts and striping, composite layouts (PFL) and Data-on-MDT, LDLM locking and what an `open()` costs, client RPC and readahead tunables, caching, and how to benchmark honestly without root. |

Facts marked **[v]** were verified by running the command on a production
system rather than taken from documentation. Measured values illustrate one
site; read them back on yours.

## Install

```bash
git clone https://github.com/UlkuTuncerKucuktas/truba-skill /tmp/hpc-skills
cp -r /tmp/hpc-skills/truba /tmp/hpc-skills/lustre ~/.claude/skills/
```

Claude Code loads each one automatically when a task touches it.
