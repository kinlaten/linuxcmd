# Repo Writing Guide

Keep this repo easy to scan on the GitHub `README.md` page.

## Default location

- Put most notes in `README.md`.
- Use `UseCase/` only for long, narrow, or multi-step topics that would make `README.md` noisy.
- If a topic moves to `UseCase/`, keep a short pointer in `README.md` when it is still worth discovering from the main page.

## Writing style

## Section placement authority

- If the user explicitly says to add content to a specific file or section, follow that exactly.
- The user determines the outline and section placement when they specify it.
- Only choose the location or section yourself when the user did not explicitly specify one.

- Prefer `sh` code blocks with short `#` comments on the line above the command.
- Keep notes simple, concise, and searchable.
- Use short headings with obvious keywords.
- Avoid repeating the same note in both `README.md` and `UseCase/`.

## Command block rule

- Put `#` comments on the line above the command, not inline at the end of the command.
- Do not add a blank line between the comment and its command.

Example:

```sh
# refresh package metadata
sudo apt update

# install tmux
sudo apt install tmux
```

## ADB notes

- For `adb` notes, include example output when useful.
- Add a short note about what to look for in that output.
