# zq — a small jq for JSON and YAML, in Zell

`zq` reads a JSON or YAML document (a file, stdin, or a newline-delimited
JSON stream), and gets, sets, lists, counts, validates, pretty-prints or
converts it to JSON, YAML or TOML. It is a showcase of writing an ordinary
command-line tool in [Zell](https://github.com/zell-lang/zell).

What it shows about Zell:

- subcommands, typed flags and generated `--help` from the standard `cli`
  package (`@cli.flag` attributes on flag structs, `cli.Bind<T>`);
- `T!` errors with `?`, `?("context")`, `try … catch` and the block form
  `try { … } catch`; a typed error (`Failure`) carrying the exit status,
  found through added context with `errors.As`;
- `encoding.json` / `encoding.yaml` dynamic values read into a small
  mutable tree (`Node`, an `enum Kind` and slices of `*Node`), and
  hand-written JSON / YAML / TOML emitters on `strings.Builder`;
- streaming NDJSON with `io.NewBufferedReader(...).ReadLine()` until
  `io.EOF`;
- testable I/O: the commands write to `io.Writer`s and read an
  `io.Reader`, so `zell test` runs the whole tool in-process against
  golden files.

## Install Zell

```sh
curl -fsSL https://raw.githubusercontent.com/zell-lang/zell/master/scripts/install.sh | sh
```

## Build and run

```sh
zell build --release -o zq       # ./zq
zell run -- get server.port testdata/app.json
```

## Usage

```text
zq [flags] <command>

  get <path> [file]           print the value at a path (-r: raw strings)
  set <path> <value> [file]   set a value (JSON literal, else a string), print the document
  keys [path] [file]          keys of an object, indexes of an array
  count [path] [file]         number of members of an object or array
  convert --to json|yaml|toml [file]
  validate --schema spec.json [file]
  fmt [--indent N] [--sort-keys] [file]

  -i, --input FILE   read FILE instead of stdin (or give it as the last argument)
  --from json|yaml   input format (default: by extension, else json)
  --ndjson           run the command on every line of a JSON-lines stream
  -h, --help         help for zq or any subcommand (zq get --help)
```

Paths: `a.b[0].c`, `servers[1]`, `["key.with.dots"]`, `.` for the whole
document. `set` creates missing object keys and appends with
`[len]`.

Validation schema (`testdata/schema.json`):

```json
{"required": ["name", "server.port", "owner"],
 "types": {"name": "string", "server.port": "integer", "tags": "array"}}
```

Types: `string`, `number`, `integer`, `boolean`, `array`, `object`, `null`.

Exit status: `0` ok, `1` usage error, `2` input error (unreadable or
malformed input, wrong type, failed validation), `3` path not found.
Diagnostics go to stderr as `zq: <command>: <key path>: <problem>`; in
NDJSON mode with the line number, the run continues and the worst status
wins.

## Example session

A real transcript (`testdata/` is in this repository):

```text
$ echo '{"a": {"b": [10, {"c": "x"}]}}' | zq get 'a.b[1].c'
"x"
$ zq get -r server.host testdata/app.json
127.0.0.1
$ zq get server.nope testdata/app.json
zq: get: server.nope: no such key
[exit 3]
$ zq keys users testdata/app.json
0
1
$ zq count tags testdata/app.json
2
$ echo '{"a": 1}' | zq set b.c '[1, "x"]' --indent 0
{"a":1,"b":{"c":[1,"x"]}}
$ echo '{"b": 1, "a": {"y": true}}' | zq fmt --sort-keys --indent 4
{
    "a": {
        "y": true
    },
    "b": 1
}
$ zq convert --to yaml testdata/app.json | head -8
name: shop
version: 3
debug: false
server:
  host: "127.0.0.1"
  port: 8080
tags:
  - api
$ echo '{"name": "shop", "server": {"port": 8080}, "users": [{"id": 1}, {"id": 2}]}' | zq convert --to toml
name = "shop"

[server]
port = 8080

[[users]]
id = 1

[[users]]
id = 2
$ zq validate -s testdata/schema.json testdata/app.json
zq: validate: owner: required
zq: validate: ratio: expected string, got number
[exit 2]
$ zq get user.name --ndjson -r testdata/events.ndjson
ann
zq: get: line 2: user: no such key
cid
zq: get: line 5: json: invalid character 'n' at offset 0
[exit 3]
$ zq get
zq: get: missing <path>
[exit 1]
```

## Test

```sh
zell test        # 15 tests: every subcommand against testdata/golden, NDJSON, one test per exit status
```

## Layout

| File | What |
|---|---|
| `src/main.zl` | `main`: real streams, `os.Exit` |
| `src/app.zl` | the `cli.App`, error → exit status |
| `src/commands.zl` | one function per subcommand |
| `src/input.zl` | reading files / stdin / NDJSON, `RunOp` |
| `src/node.zl` | the document tree, from `json.Value` / `yaml.Value` |
| `src/path.zl` | key paths: parse, lookup, assign |
| `src/emit_*.zl` | JSON, YAML, TOML writers |
| `src/schema.zl` | the validate schema |
| `src/errors.zl` | `Failure` and exit statuses |

## Notes

- Arguments are parsed with the standard `cli` package rather than
  [zconf](https://github.com/zell-lang/zconf): zconf binds configuration
  layers, `cli` is the tool for subcommands with per-command help.
- `cli.App.Run` knows only 0/1/2, so commands report their own errors and
  keep the status aside (`src/app.zl`); its help is printed straight to
  stdout.

## License

MIT
