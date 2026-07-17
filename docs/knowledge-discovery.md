# Knowledge Discovery and Topics

Lessons, notes, and rules are organized around topics such as `command-dispatch`, `slash-commands`, and `tui`. Every knowledge record has one primary topic and may have related topics.

## Find Knowledge

Search notes, lessons, and rules together when you do not know which type contains the answer:

```bash
gsc knowledge search "slash command"
```

Browse a known topic:

```bash
gsc knowledge list --topic command-dispatch
```

Or limit the search to one kind of record:

```bash
gsc knowledge search "tui" --type lessons
```

## Browse Topics

```bash
gsc topics list
gsc topics show command-dispatch
gsc topics search slash
```

## Migrate Older Records

If the repository has lessons created before topics were required, preview and apply the migration:

```bash
gsc topics migrate --dry-run
gsc topics migrate
```
