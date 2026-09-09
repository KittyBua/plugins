# Tuxbox Neutrino plugins

Deutsch: [README.de.md](README.de.md)

This repository is a superproject: every plugin lives in its own
repository and is linked here as a git submodule. The commits here are
almost exclusively pin bumps, so the tree of a superproject commit is a
known-good snapshot of all plugins together. A *pin* is the commit a
submodule entry points at; git calls that entry a *gitlink*.

**In one sentence:** change a plugin in its own repository, push it
there, and the pin here follows within the hour on its own.

## Contents

1. [Clone](#clone)
2. [Change something in a plugin](#change-something-in-a-plugin)
3. [Move only the pin](#move-only-the-pin)
4. [What happens on its own](#what-happens-on-its-own)
5. [Make it happen now](#make-it-happen-now)
6. [Commit subjects](#commit-subjects)
7. [Install the hook](#install-the-hook)
8. [Several plugins at once](#several-plugins-at-once)
9. [Add a plugin](#add-a-plugin)
10. [Which branch a pin follows](#which-branch-a-pin-follows)
11. [Move a pin without a checkout](#move-a-pin-without-a-checkout)

## Clone

With the submodules, or you get empty directories:

```bash
git clone --recursive https://github.com/tuxbox-neutrino/plugins.git
```

## Change something in a plugin

The everyday case. Work inside the submodule's checkout, commit and push
there. The pin here follows on its own within the hour — the second half
below is optional and only makes it land at once:

```bash
cd plugins                          # the superproject clone from above
cd scripts-lua                      # the submodule; any plugin directory works the same
git switch master && git pull       # a fresh clone leaves it detached on the pin
$EDITOR plugins/webtv/<script>.lua  # "plugins" again: the scripts' own subdirectory
git add . && git commit -m "fix (<script>): ..." && git push

cd ..                               # back in the superproject — optional from here
git diff --submodule=log            # the plugin commits the pin will move over
git add scripts-lua
git commit -m "update superproject: sample fix"
git push
```

The `git switch master` is not decoration: a fresh clone leaves every
submodule detached on its pin, a commit made there sits on no branch,
and `git push` refuses with "You are not currently on a branch". The
`--submodule=log` line lists the plugin commits the pin moves over —
that is the phrase for the subject. After the push, `git log -1
origin/master` shows your commit at the tip.

## Move only the pin

When the plugin was changed and pushed elsewhere — its own clone, another
machine, a colleague — and only the pin here is to follow:

```bash
git submodule update --remote scripts-lua   # fetch the plugin's master
git diff --submodule=log                    # what the pin moves over; empty = nothing to do
git add scripts-lua
git commit -m "update superproject: sample fix"
git push
```

`--remote` moves the checkout to the tip of the branch the pin follows,
which is exactly what the timer does. Without it, `git submodule update`
would put the checkout back on the **old** pin.

## What happens on its own

Nothing here has to be edited by hand for a plugin change. Two workflows
carry it:

1. **`plugin-scripts-lua`** (the Lua scripts, linked here as `scripts-lua`)
   is itself a superproject for the extracted Lua plugins
   (`plugins/logoupdater`, `plugins/neutrino-mediathek`,
   `plugins/stb_startup`, `plugins/webmin-setup`). Its workflow
   `update-submodule-pins.yml` runs **every hour at :17** and moves each of
   those gitlinks to the tip of the plugin's branch.
2. **This repository** runs the same workflow **every hour at :47** and
   moves every gitlink here, `scripts-lua` among them.

The half-hour offset is deliberate: a fix pushed to a Lua plugin at 09:00
is in `plugin-scripts-lua` by 09:17 and here by 09:47 (GitHub often
starts a scheduled run a few minutes late), without waiting for a second
full cycle. **Nothing triggers on push** — both workflows are
timers, so a pin is current within about an hour and a half at worst.
That is also why a push made by the workflow itself starts no further
workflow: GitHub does not chain workflows off pushes made with the
built-in token, and the timers do not need it to.

The workflow commits as `GitHub Actions <actions@github.com>`, the same
identity the other Tuxbox workflows use, and says what moved: with one
gitlink the subject is `update superproject: <path>: <subject of the
plugin's newest commit>`, with several it lists the paths. The body has every
moved gitlink as `<path> <old>..<new>` followed by the subjects of the
plugin commits in that range, so the log reads on its own. It refuses to
commit anything that is not a gitlink, and it never touches a branch
other than `master`.

## Make it happen now

Instead of waiting for the timer, start the workflow by hand in the
repository whose pin is to move: Actions → *Update submodule pins* →
*Run workflow*, or with the GitHub CLI (`gh`):

```bash
gh workflow run update-submodule-pins.yml -R tuxbox-neutrino/plugin-scripts-lua --ref master   # the Lua scripts
gh workflow run update-submodule-pins.yml -R tuxbox-neutrino/plugins --ref master              # this repository
```

For a Lua plugin, which passes both levels, run the first, wait for it
to finish, then the second. `gh run list -R tuxbox-neutrino/plugins`
shows the run and its result.

## Commit subjects

Subjects here start with `update superproject:`, and the phrase after
the colon says what moved:

```
update superproject: sample fix
```

Do say what moved. A log made of bare `update superproject` lines tells
nobody what changed in which plugin, and finding out means walking into
every submodule and reading its log there. One phrase — `sample fix`,
`<plugin> 0.9` — makes the history readable on its own.

The subject used to be plain `- update superproject`; that form still
passes. If you keep it, write the space after the dash. `-update
superproject`, as some old notes have it, is not what the history uses.
Nothing on GitHub checks any of this; it is a convention. The hook below
keeps you to it.

## Install the hook

`.githooks/commit-msg` rejects a subject that does not start with
`update superproject`, asks for the phrase after the colon when it is
missing, and warns about lines over 72 columns. Once per clone:

```bash
git config core.hooksPath .githooks
```

That points git at the directory for this repository only; the
submodules keep their own hooks. It also switches off whatever is in
this clone's `.git/hooks/` — if you keep hooks there, copy the file into
`.git/hooks/` instead of setting the option. To try it on a message before
committing:

```bash
printf 'update superproject: sample fix\n' > /tmp/msg && .githooks/commit-msg /tmp/msg
```

The file comes with the clone, so this is only needed to carry the hook
somewhere else — into another superproject, say. Save it as
`.githooks/commit-msg` and make it executable (`chmod +x`):

<details>
<summary><b>.githooks/commit-msg</b></summary>

```sh
#!/bin/sh
# commit-msg hook for the tuxbox-neutrino/plugins superproject.
#
# Keeps the subject of every commit here in the form the history has
# used for years:
#
#     update superproject: <what moved and why>
#
# The bare "update superproject" and the older "- update superproject"
# are let through as well, so old habits do not block anyone, but the
# hint below asks for the phrase. Merge, revert, fixup and squash
# commits are exempt. Subjects and body lines longer than 72 columns
# get a warning, not a rejection.
#
# Install (per clone, once):
#
#     git config core.hooksPath .githooks
#
# or copy the file into .git/hooks/. Either way it applies to this
# repository only; the submodules have hooks of their own.

msg="$1"

# Subject: the first line that is neither blank nor a comment.
subject=$(grep -v '^[[:space:]]*#' "$msg" | grep -m1 -v '^[[:space:]]*$' || true)
[ -n "$subject" ] || exit 0   # empty message: git aborts the commit itself

case "$subject" in
	"Merge "*|"Revert "*|"fixup! "*|"squash! "*|"amend! "*)
		;;
	"update superproject: "?*)
		;;
	"update superproject"|"- update superproject"*)
		printf 'hint: say what moved, e.g. "update superproject: sample fix"\n' >&2
		;;
	*)
		cat >&2 <<HINT
commit-msg: subject does not follow this repository's convention:

    $subject

Expected:

    update superproject: <what moved and why>

The commits here are pin bumps; the phrase after the colon is what
makes the log readable without walking into every submodule.
HINT
		exit 1
		;;
esac

# Length: warn, do not reject.
awk '
	NR == 1 && length($0) > 72 {
		printf("warning: subject is %d chars (limit 72)\n", length($0)) > "/dev/stderr"
	}
	NR > 1 && !/^[[:space:]]*#/ && length($0) > 72 {
		printf("warning: line %d is %d chars (limit 72)\n", NR, length($0)) > "/dev/stderr"
	}
' "$msg"
exit 0
```

</details>

## Several plugins at once

If you changed several plugins — each committed and pushed on its
`master` as above — let the superproject rebase every submodule checkout
onto its recorded pin first, so nothing you committed locally gets lost:

```bash
git submodule update --rebase --recursive
git diff --submodule=log                    # one block per moved pin
git add -u                                  # only the moved pins, no stray files
git commit -m "update superproject: <what moved>"
```

Push the submodules **before** the superproject, so the pins you publish
point at commits that exist upstream.

## Add a plugin

The plugin repository has to exist on GitHub under `tuxbox-neutrino/`
first; then:

```bash
git submodule add ../plugin-<name>.git <name>
git commit -m "update superproject: build (plugins): link <name>"
```

The URLs are relative on purpose (`../<repo>.git`), so the same
`.gitmodules` works over HTTPS and SSH and from a fork.

## Which branch a pin follows

The branch named in `.gitmodules` (`branch = …`), else the plugin
repository's default branch. Right now no entry names one, so every pin
follows the default branch, which is `master` for all of them. To pin a
plugin to another branch, add `branch = <name>` to its `.gitmodules`
entry; the next run moves it there.

## Move a pin without a checkout

If the workflow is down and you have no submodule checkout at hand, a pin
is one command per gitlink:

```bash
git update-index --cacheinfo 160000,<sha>,<path>
git commit -m "update superproject: <what moved and why>"
```
