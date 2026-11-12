# Publishing checklist

The public repo holds **curated writeups, not raw engagement archives**.

## Before each article goes public

- [ ] Verify the target is **retired or expressly eligible for publication** under its current platform rules.
- [ ] Keep CPTS capstone/exam and most Pro Lab solutions private.
- [ ] Write explanations in your own words; distinguish observation, hypothesis, and proof.
- [ ] Select only command-output excerpts establishing important facts.
- [ ] Remove plaintext passwords, hashes, tokens, flags, personal records and private account metadata.
- [ ] Crop/redact screenshots; don't rely on reversible blur.
- [ ] Confirm links and assets render correctly on GitHub.
- [ ] Review staged changes with `git diff --cached`.
- [ ] Confirm no raw notebooks, credential stores, VPN files or evidence archives are tracked (`git ls-files`).
- [ ] Change **Draft outline** to **Published** only after the author completes and reviews the narrative.

> [!WARNING]
> `.gitignore` cannot protect content explicitly force-added or previously committed. Removing a committed secret later doesn't erase Git history. Keep raw materials outside this repository.

## Workflow

**Private CherryTree / Obsidian / terminal logs → private evidence map → public outline → your prose → technical review → publish.**

Sources: [HTB sharing guidelines](https://help.hackthebox.com/en/articles/5188925-streaming-writeups-walkthrough-guidelines), [GitHub Markdown](https://docs.github.com/en/get-started/writing-on-github).
