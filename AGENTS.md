# AGENTS.md

Guidance for agents working on Stillpage. Use [`docs/goal.md`](docs/goal.md) to understand the experience the product is aiming for.

## Make decisions that age well

- Prefer code that stays easy to understand and change, not just the quickest patch.
- Keep solutions simple. Don't add abstractions or features for hypothetical needs.
- Before a major choice, consider how hard it will be to change later and whether it affects saved user data.
- If the goal or an important choice is unclear, ask rather than guessing.

## Keep changes easy

- Give each module or component one clear responsibility.
- Keep product rules separate from UI, storage, platform APIs, and content extraction.
- Make platform-specific or replaceable behavior easy to swap, but don't add layers that have no current purpose.
- Keep shared behavior and settings in one place. Avoid repeated values and hidden global state.
- When changing stored data, preserve existing user data or provide a migration.

## Make visual choices configurable

Colors, fonts, spacing, layout, and reader styles should be easy to adjust without editing many unrelated files.

- Use shared design tokens for visual values instead of scattered hard-coded styles.
- Keep saved content separate from how it is displayed.
- Make meaningful style options easy to switch, with sensible defaults and readable contrast.
- Keep user preferences distinct from developer-controlled defaults.
- Don't build a theme framework until the app needs one.

## Working process

1. Inspect the relevant code and the project goal; follow the project's existing conventions.
2. Make the smallest complete change. Avoid unrelated cleanup and dependencies without a clear need.
3. Add or update tests for changed behavior.
4. Run the relevant tests and checks. Say what could not be run.
5. Summarize the change and any important trade-offs briefly.
