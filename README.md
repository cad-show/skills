# CAD.show skills

Drive a live CAD.show tab from your coding agent with the `cad` CLI: build, inspect, edit and export 3D geometry through the Scene Actions the in-app agent uses.

The skill needs the `cad` CLI (`npm i -g @cadshow/cad`) and a signed-in
CAD.show window: the desktop app, or a browser tab with `cad broker` running.

## Install

| Agent | Command |
| ----- | ------- |
| Claude Code | `/plugin marketplace add cad-show/skills`, then `/plugin install cad-show@cad-show` |
| any agent, via [skills.sh](https://skills.sh) | `npx skills add cad-show/skills --skill cad-show` |
| no marketplace | `cad skill install <dir>` |

## Source

This repository is generated. Every file is written from the CAD.show product
repository on each release, so a pull request here is overwritten by the next
sync; report issues at https://cad.show instead.

Version 0.1.1, the version of `@cadshow/cad` that ships the same
`SKILL.md`.
