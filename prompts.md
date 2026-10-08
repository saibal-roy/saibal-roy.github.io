# How I build: prompts

*By [Saibal Roy](https://www.saibalroy.com/) · [LinkedIn](https://www.linkedin.com/in/roysaibal) · [GitHub](https://github.com/saibal-roy)*

I build open source from **real production requirements**: a problem a business actually has, with a budget, a deadline, confidential data and a team that has to keep it running. The aim isn't a demo; it's a **reusable business solution** that someone else can deploy, trust and afford. That means designing the whole system, not just the code:

- **Cost first.** Start from the unit economics (cost per page, per document, per server hour) and size the solution down to the smallest machine that measurably works.
- **Reliability you can prove.** Every requirement gets an acceptance check, and nothing ships until a go-ahead gate rehearses a fresh server on the supported platform and target size.
- **Evidence before decisions.** Benchmarks, not opinions. When there's a trade-off, the numbers go to the owner, and the decision and its evidence are written down.
- **Secure and confidential by default.** Services bound to localhost, no stored credentials, client data kept out of the repository by mechanism rather than memory.
- **Built for the people who maintain it.** Pinned versions, one supported platform, scripts for setup, demo and cleanup, upgrades only through the gate.
- **Grow in stages.** One small server first; the next stage is written up as a proposal with the triggers that would justify it.

These are the prompts I use with Claude to work this way. There is **one copy**, kept in the project where every change is validated, so it always describes the current working state:

- **[Build docling-batch-extract from scratch](https://saibal-roy.github.io/docling-batch-extract/prompts/#part-1-build-from-scratch)**: batch PDF to Markdown for RAG on one CPU-only 2 vCPU / 8 GB server, with every validated decision built in ([documentation](https://saibal-roy.github.io/docling-batch-extract/), [how it was built](https://saibal-roy.github.io/docling-batch-extract/STORY/)).
- **[The reusable template](https://saibal-roy.github.io/docling-batch-extract/prompts/#part-7-reusable-template-for-any-solution-like-this)**: the first prompt I start any similar solution with.
- **[All the prompts](https://saibal-roy.github.io/docling-batch-extract/prompts/)**, including operating, maintaining, investigating and extending the project. Source: [`prompts.md`](https://github.com/saibal-roy/docling-batch-extract/blob/main/prompts.md).

The prompts improve themselves: after every run that succeeds, whatever had to be corrected or added is folded back into the prompt, and each change is logged at the end of the file. How I publish the result on GitHub is on [a page of its own](https://saibal-roy.github.io/github-practices/).
