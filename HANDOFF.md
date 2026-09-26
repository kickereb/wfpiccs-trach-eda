# Handoff — Claude (chat) <-> Claude Code (VSCode)

Shared working notes between the two Claude sessions. Claude (chat) does thinking/ideation,
Claude Code (VSCode) does the actual coding/running. Neither session can message the other
directly, so this file is the bridge — always read it before starting work, always update
your section when you hand off.

## To VSCode Claude Code

_(tasks queued up for VSCode to implement/run)_

- [ ] Nothing queued yet.

## From VSCode Claude Code

_(results, blockers, questions — written back after doing a task)_

- Nothing yet.

## Notes

- Data lives in `SATI-Q_WFPICCS_DATATHON_DATASET/` and `ANZPICR_WFPICCS_DATATHON_DATASET/` —
  gitignored, local only, never commit it.
- `WFPICCS_Trach_EDA.ipynb` is the working notebook (also mirrored to a Colab copy on Drive).
- Env: use the `data_science` conda env (`~/anaconda3/bin/conda install -y -n data_science -c conda-forge "numpy<2" pandas matplotlib seaborn`) — plain pip numpy/pandas versions clash on this Mac.
