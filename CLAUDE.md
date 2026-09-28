This is a Claude plugin that contains a model-agnostic agent skill for interacting with the Boomi Data Integration product (aka BDI - Former Rivery)

## Scripts

`scripts/bdi-common.sh` is sourced by every other script and owns `.env` loading, the `bdi_api` curl wrapper, pagination, and operation polling. The rest are per-domain subcommand scripts (`bdi-flow.sh`, `bdi-connection.sh`, `bdi-cdc.sh`, `bdi-dataframe.sh`, `bdi-variable.sh`, `bdi-logicode.sh`, `bdi-env.sh`, `bdi-env-check.sh`).

### Credential handling conventions

- **`xtrace_off` / `xtrace_on` wrap every credential expansion.** Under `xtrace` bash prints each command to stderr *after* expansion, so a traced `source ./.env`, `${!var}` test, or presigned-URL interpolation prints the secret — and an agent harness typically captures stderr into its session transcript. The two helpers save, clear, and conditionally restore the caller's tracing state, and are depth-counted so one guarded helper can call another. A guard reads as redundant if you don't know this, so don't delete one; each call site carries a one-line comment naming what it protects. Bash 3.2 offers no way to test a variable's value without expanding it into a traced position, so the guard is the only remedy.
- **Response bodies go to a temp file under `$_BDI_TMPDIR`**, a `mktemp -d` created when the library is sourced and removed by an `EXIT` trap. Never reintroduce a bare `mktemp` — an orphaned response body can hold environment-variable values.
- **Call sites still `rm` their own response file on the success path.** The trap is there for the interrupted path, not instead of cleaning up.
- **That `EXIT` trap is the only one in the skill.** A second `trap ... EXIT` anywhere would replace it and silently restore the leak, so extend the existing one rather than adding another.
- **Auth reaches curl on stdin** (`curl -K -` via `curl_cfg`), never in argv, so stdin is reserved: no `@-`, no `-T -`, no piping into `bdi_api`. Inline `-d` and `--data-binary @file` are both fine.
