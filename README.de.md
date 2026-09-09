# Tuxbox Neutrino plugins

English: [README.md](README.md)

Dieses Repository ist ein Superprojekt: jedes Plugin lebt in einem
eigenen Repository und ist hier als Git-Submodul eingebunden. Die Commits
hier sind fast ausschließlich Pin-Bumps, der Baum eines
Superprojekt-Commits ist also ein bekannt guter Schnappschuss aller
Plugins zusammen. Ein *Pin* ist der Commit, auf den ein Submodul-Eintrag
zeigt; Git nennt diesen Eintrag *Gitlink*.

**In einem Satz:** Du änderst ein Plugin in seinem eigenen Repository,
pushst es dort, und der Pin hier zieht innerhalb einer Stunde von allein
nach.

## Inhalt

1. [Klonen](#klonen)
2. [Etwas in einem Plugin ändern](#etwas-in-einem-plugin-ändern)
3. [Nur den Pin nachziehen](#nur-den-pin-nachziehen)
4. [Was von allein passiert](#was-von-allein-passiert)
5. [Sofort statt warten](#sofort-statt-warten)
6. [Commit-Subjects](#commit-subjects)
7. [Den Hook installieren](#den-hook-installieren)
8. [Mehrere Plugins auf einmal](#mehrere-plugins-auf-einmal)
9. [Ein Plugin hinzufügen](#ein-plugin-hinzufügen)
10. [Welchem Branch ein Pin folgt](#welchem-branch-ein-pin-folgt)
11. [Einen Pin ohne Checkout setzen](#einen-pin-ohne-checkout-setzen)

## Klonen

Mit den Submodulen, sonst bekommst du leere Verzeichnisse:

```bash
git clone --recursive https://github.com/tuxbox-neutrino/plugins.git
```

## Etwas in einem Plugin ändern

Der Alltagsfall. Du arbeitest im Checkout des Submoduls, committest und
pushst dort. Der Pin hier zieht innerhalb einer Stunde von allein nach —
die zweite Hälfte unten ist optional und lässt ihn nur sofort landen:

```bash
cd plugins                          # der Superprojekt-Klon von oben
cd scripts-lua                      # das Submodul; jedes Plugin-Verzeichnis geht genauso
git switch master && git pull       # ein frischer Klon lässt es losgelöst auf dem Pin
$EDITOR plugins/webtv/<script>.lua  # nochmal "plugins": das Unterverzeichnis der Skripte
git add . && git commit -m "fix (<script>): ..." && git push

cd ..                               # zurück im Superprojekt — ab hier optional
git diff --submodule=log            # die Plugin-Commits, über die der Pin springt
git add scripts-lua
git commit -m "update superproject: sample fix"
git push
```

Das `git switch master` ist keine Zierde: ein frischer Klon lässt jedes
Submodul losgelöst auf seinem Pin stehen, ein Commit dort liegt auf
keinem Branch, und `git push` bricht ab mit „Sie befinden sich im Moment
auf keinem Branch". Die `--submodule=log`-Zeile führt die Plugin-Commits
auf, über die der Pin springt — das ist der Halbsatz für das Subject.
Nach dem Push zeigt `git log -1 origin/master` deinen Commit an der
Spitze.

## Nur den Pin nachziehen

Wenn das Plugin woanders geändert und gepusht wurde — eigener Klon,
anderer Rechner, ein Kollege — und nur der Pin hier folgen soll:

```bash
git submodule update --remote scripts-lua   # holt den master des Plugins
git diff --submodule=log                    # worüber der Pin springt; leer = nichts zu tun
git add scripts-lua
git commit -m "update superproject: sample fix"
git push
```

`--remote` setzt den Checkout auf die Spitze des Branches, dem der Pin
folgt, also genau das, was der Timer tut. Ohne `--remote` würde `git
submodule update` den Checkout auf den **alten** Pin zurücksetzen.

## Was von allein passiert

Für eine Plugin-Änderung muss hier nichts von Hand editiert werden. Zwei
Workflows tragen das:

1. **`plugin-scripts-lua`** (die Lua-Skripte, hier als `scripts-lua`
   eingebunden) ist selbst ein Superprojekt für die herausgelösten
   Lua-Plugins (`plugins/logoupdater`, `plugins/neutrino-mediathek`,
   `plugins/stb_startup`, `plugins/webmin-setup`). Sein Workflow
   `update-submodule-pins.yml` läuft **jede Stunde um :17** und setzt jeden
   dieser Gitlinks auf die Spitze des Plugin-Branches.
2. **Dieses Repository** lässt denselben Workflow **jede Stunde um :47**
   laufen und bewegt jeden Gitlink hier, `scripts-lua` eingeschlossen.

Der Versatz um eine halbe Stunde ist Absicht: ein Fix, der um 09:00 in ein
Lua-Plugin gepusht wird, ist um 09:17 in `plugin-scripts-lua` und um 09:47
hier (GitHub startet geplante Läufe oft ein paar Minuten verspätet), ohne
auf eine zweite volle Runde zu warten. **Auf einen Push
reagiert nichts** — beide Workflows sind Timer, ein Pin ist also
schlimmstenfalls nach etwa anderthalb Stunden aktuell. Deshalb stört es
auch nicht, dass ein Push des Workflows selbst keinen weiteren Workflow
anstößt: GitHub kettet Workflows nicht an Pushes mit dem eingebauten
Token, und die Timer brauchen das nicht.

Der Workflow committet als `GitHub Actions <actions@github.com>`, mit
derselben Identität wie die übrigen Tuxbox-Workflows, und sagt, was sich
bewegt hat: bei einem Gitlink lautet das Subject
`update superproject: <pfad>: <Subject des neuesten Plugin-Commits>`, bei
mehreren nennt es die Pfade. Der Body
führt jeden bewegten Gitlink als `<pfad> <alt>..<neu>` auf, gefolgt von
den Subjects der Plugin-Commits in diesem Bereich, damit das Log für sich
allein lesbar ist. Er weigert sich, etwas anderes als Gitlinks zu
committen, und er fasst nie einen anderen Branch als `master` an.

## Sofort statt warten

Statt auf den Timer zu warten, startest du den Workflow von Hand in dem
Repository, dessen Pin sich bewegen soll: Actions → *Update submodule
pins* → *Run workflow*, oder mit der GitHub-CLI (`gh`):

```bash
gh workflow run update-submodule-pins.yml -R tuxbox-neutrino/plugin-scripts-lua --ref master   # die Lua-Skripte
gh workflow run update-submodule-pins.yml -R tuxbox-neutrino/plugins --ref master              # dieses Repository
```

Bei einem Lua-Plugin, das beide Ebenen durchläuft, erst den ersten
Befehl, auf das Ende des Laufs warten, dann den zweiten. `gh run list -R
tuxbox-neutrino/plugins` zeigt den Lauf und sein Ergebnis.

## Commit-Subjects

Subjects beginnen hier mit `update superproject:`, und der Halbsatz nach
dem Doppelpunkt sagt, was sich bewegt hat:

```
update superproject: sample fix
```

Schreib dazu, was sich bewegt hat. Ein Log aus lauter nackten
`update superproject`-Zeilen sagt niemandem, was sich in welchem Plugin
geändert hat, und um es herauszufinden, muss man in jedes Submodul
hineinnavigieren und dort das Log lesen. Ein Halbsatz — `sample fix`,
`<plugin> 0.9` — macht die Historie für sich allein lesbar.

Das Subject hieß früher schlicht `- update superproject`; diese Form geht
weiterhin durch. Wenn du bei ihr bleibst, schreib das Leerzeichen nach
dem Strich. `-update superproject`, wie es in alten Notizen steht, ist
nicht das, was die Historie benutzt. Auf GitHub prüft das alles nichts,
es ist eine Konvention. Der Hook unten hält dich daran.

## Den Hook installieren

`.githooks/commit-msg` lehnt ein Subject ab, das nicht mit
`update superproject` beginnt, bittet um den Halbsatz nach dem
Doppelpunkt, wenn er fehlt, und warnt bei Zeilen über 72 Zeichen. Einmal
je Klon:

```bash
git config core.hooksPath .githooks
```

Das zeigt Git das Verzeichnis nur für dieses Repository; die Submodule
behalten ihre eigenen Hooks. Es schaltet aber auch alles ab, was in
`.git/hooks/` dieses Klons liegt — wer dort eigene Hooks hat, kopiert die
Datei stattdessen nach `.git/hooks/`, statt die Option zu setzen. Zum Ausprobieren an
einer Message, bevor du committest:

```bash
printf 'update superproject: sample fix\n' > /tmp/msg && .githooks/commit-msg /tmp/msg
```

Die Datei kommt mit dem Klon; der Block unten wird nur gebraucht, um den
Hook woanders hin mitzunehmen — in ein weiteres Superprojekt etwa. Als
`.githooks/commit-msg` speichern und ausführbar machen (`chmod +x`):

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

## Mehrere Plugins auf einmal

Hast du mehrere Plugins geändert — jedes wie oben auf seinem `master`
committet und gepusht —, lass das Superprojekt zuerst jeden
Submodul-Checkout per Rebase auf seinen eingetragenen Pin setzen, damit
nichts verloren geht, was du lokal committet hast:

```bash
git submodule update --rebase --recursive
git diff --submodule=log                    # ein Block je bewegtem Pin
git add -u                                  # nur die bewegten Pins, keine liegen gebliebenen Dateien
git commit -m "update superproject: <was sich bewegt hat>"
```

Push die Submodule **vor** dem Superprojekt, damit die veröffentlichten
Pins auf Commits zeigen, die es upstream gibt.

## Ein Plugin hinzufügen

Das Plugin-Repository muss zuerst auf GitHub unter `tuxbox-neutrino/`
existieren; dann:

```bash
git submodule add ../plugin-<name>.git <name>
git commit -m "update superproject: build (plugins): link <name>"
```

Die URLs sind absichtlich relativ (`../<repo>.git`), damit dieselbe
`.gitmodules` über HTTPS und SSH und aus einem Fork funktioniert.

## Welchem Branch ein Pin folgt

Dem Branch, der in `.gitmodules` steht (`branch = …`), sonst dem
Standard-Branch des Plugin-Repositories. Derzeit nennt kein Eintrag
einen, alle Pins folgen also dem Standard-Branch, und das ist bei allen
`master`. Um ein Plugin auf einen anderen Branch zu pinnen, trägst du
`branch = <name>` in seinen `.gitmodules`-Eintrag ein; der nächste Lauf
zieht es dorthin.

## Einen Pin ohne Checkout setzen

Wenn der Workflow ausfällt und du keinen Submodul-Checkout zur Hand hast,
ist ein Pin ein Befehl je Gitlink:

```bash
git update-index --cacheinfo 160000,<sha>,<pfad>
git commit -m "update superproject: <was sich bewegt hat und warum>"
```
