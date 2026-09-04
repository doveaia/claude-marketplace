# Changelog

## 0.4.0

- Renamed from `mattpocock-skills-override` to `mattpocock-skills-addons`: nothing in it overrides anything, it extends upstream through its per-repo config. The install id and every qualified skill invocation change with it (`mattpocock-skills-addons:setup-triage-board`, `mattpocock-skills-addons:sync-triage-board`).
- Moved into the `doveaia` marketplace.

## 0.3.0

Last release as `mattpocock-skills-override` in `doveaia/mattpocock-override` (history there).

- `setup-triage-board`: creates a GitHub Projects v2 board for `mattpocock-skills:triage`, named after the repo, six columns on the built-in Status field; writes `docs/agents/github-project.md` and the board rule into `docs/agents/issue-tracker.md`.
- `sync-triage-board`: moves an item's card when a triage label changes; audits drift and backfills the backlog on approval. Labels are the source of truth.
- Extends upstream through its per-repo config. Upstream's `triage` is user-invoked and cannot be wrapped; an earlier wrapper design was dropped.
