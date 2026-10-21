# pos-evidence

The validation evidence for [OmarSweiti/pos](https://github.com/OmarSweiti/pos) pull requests: the
screenshots and summary each pull request's **Validation evidence** comment shows. The owner approves
every pull request on this evidence (`docs/implementation/03-github-workflow.md` §6 in that
repository).

## Layout

```
pr-<number>/<UTC time>-<head commit>/
  summary.md        what was run, expected and seen, and what was not validated
  01-<step>.png     one screenshot per step
```

`just evidence <pr> <directory>` in the pos repository writes a new folder here, pushes it, and posts
the comment. The comment embeds each image **at the commit that stored it**, so a link can never be
repointed, and names the pull request's exact head commit as the one validated.

## Rules

- **Append-only.** A ruleset refuses force-pushes and deletions on every branch, with no bypass, so
  the record only grows. A correction is a new folder, never an edit.
- **Synthetic data only.** Every image is inspected before it is published: no real customer data, no
  PIN, card value, token or key, nothing from another window.
- **Public on purpose.** GitHub shows an image inside a pull request comment only when the image is
  publicly reachable. The pos repository is public too; if it ever goes private, this storage must
  change with it.
- **Nothing here is code.** Keeping the images in their own repository keeps every clone of the code
  free of them.

## Migrated

Until 21 October 2026, evidence was stored on `refs/evidence/pr-<n>` inside the pos repository. Those
sets were copied here unchanged from these commits, which stay in place so every existing comment
keeps working:

| Folder | Original evidence commit |
|---|---|
| [`pr-260`](pr-260/) | [`b34ac59`](https://github.com/OmarSweiti/pos/tree/b34ac598c8f3483c08ac45db5a1acd0eb71e6eb3) |
| [`pr-267`](pr-267/) | [`36beec4`](https://github.com/OmarSweiti/pos/tree/36beec449c4d22cea1f619842e9b9f10e946e2bd) |
| [`pr-268`](pr-268/) | [`8e43518`](https://github.com/OmarSweiti/pos/tree/8e435183b052d363525547cc7d108d87b4b9bd7a) |
| [`pr-269`](pr-269/) | [`165e62c`](https://github.com/OmarSweiti/pos/tree/165e62c3a28d6c518c12fe81a278cc7a3d722941) |
| [`pr-271`](pr-271/) | [`d8c8a16`](https://github.com/OmarSweiti/pos/tree/d8c8a1633b98089886892463f3ecbf88f0831616) |

