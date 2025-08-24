# Semantic Commit Messages with Emojis

Commit format: `<emoji_type> <commit_type>(<scope>): <subject>. <issue_reference>`

## Example

```
:sparkles: feat(Component): Add a new feature. Closes: #
^--------^ ^--^ ^-------^   ^---------------^  ^------^
|          |    |           |                  |
|          |    |           |                  +--> (Optional) Issue reference: if the commit closes or fixes an issue
|          |    |           |
|          |    |           +---------------------> Commit summary
|          |    |
|          |    +---------------------------------> (Optional) Commit scope in the project
|          |
|          +--------------------------------------> Commit type: feat, fix, docs, refactor, test, style, chore, build, perf or ci
|
+-------------------------------------------------> (Optional) Emoji type. See: https://gitmoji.carloscuesta.me/
```

**The commit message will be:**

> feat: Add a new feature

**With optional features emoji, scope and issue reference:**

> :sparkles: feat(Component): Add a new feature. Closes: #..

## Commit Message Types

- **feat**: introducing a new feature to the codebase
- **fix**: fixing a bug in the codebase
- **docs**: adding or updating the documentation
- **refactor**: refactoring the production code
- **build**/**conf**: changes related to the build system (involving scripts, configurations) and package dependencies
- **test**: adding tests (no production code change)
- **ci**: changes related to the continuous integration and deployment system
- **style**: improving structure/format of the code e.g. missing semi colons (no production code change)
- **chore**: updating grunt tasks etc. (no production code change)
- **perf**: changes related to backward-compatible performance improvements

## Supported Emojis by Commit Message Types

| Type     | Emoji                                         |
| -------- | --------------------------------------------- |
| feat     | :sparkles: `:sparkles:`                       |
| fix      | :bug: `:bug:`                                 |
| docs     | :memo: `:memo:`                               |
| refactor | :recycle: `:recycle:`                         |
| build    | :construction_worker: `:construction_worker:` |
| test     | :white_check_mark: `:white_check_mark:`       |
| ci       | :green_heart: `:green_heart:`                 |
| style    | :art: `:art:`                                 |
| chore    | :wrench: `:wrench:`                           |
| perf     | :zap: `:zap:`                                 |

Besides the emojis of these commit types, other related emojis can also be used in the commit messages. For example:

`:construction_worker: build(Electron): Bump version 7 to 9 :arrow_up:`

> :construction_worker: build(Electron): Bump version 7 to 9 :arrow_up:

## Issue Referencing

Keywords to close an related issue with the commit:

- close
- closes
- closed
- fix
- fixes
- fixed
- resolve
- resolves
- resolved

You can use the phrase: `Fixes: #1` or `Fixes #1`.
Once the branch is merged into the default branch, the issue will close.

## References

- https://gitmoji.carloscuesta.me/
- https://nitayneeman.com/posts/understanding-semantic-commit-messages-using-git-and-angular/
