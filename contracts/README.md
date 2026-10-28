# Contracts

D-4 (revised 28 Oct 2026). One OpenAPI file per service, named `<Service>.openapi.json` — a **generated
snapshot of each service's own real, live contract**, not a hand-authored spec maintained independently
of the code. This exists so someone can see every service's actual API in one place without cloning and
running six separate repos — valuable for an open-source project specifically — while staying accurate
by construction: every file here is produced directly from that service's own compiled code via
Swashbuckle, the same tool that generates its live `/swagger` endpoint.

**Nobody edits these files directly, ever — not builders, not the coordinator.** A contract changes in
the service's own code first (same human-only-gate process as any other API contract change — see
`AGENTS.md`); this folder is only ever regenerated *from* that change afterward, never the other way
around. If a file here and a service's real behavior disagree, the service's code is correct and this
folder is stale — regenerate it, don't hand-edit it to match.

## Regenerating a service's snapshot

```bash
dotnet tool install -g Swashbuckle.AspNetCore.Cli   # once per machine
export PATH="$PATH:$HOME/.dotnet/tools"

cd /path/to/<Repo>/src/<Repo>
dotnet build -c Release
ASPNETCORE_ENVIRONMENT=Development swagger tofile \
  --output /path/to/garbati-platform/contracts/<Repo>.openapi.json \
  bin/Release/net8.0/<Repo>.dll v1
```

`ASPNETCORE_ENVIRONMENT=Development` only affects which code path registers Swagger in `Program.cs`
(every service gates it there — see each one's own comment, "always in Development; elsewhere only when
`Swagger:Enabled` is true, the public showcase") — it does not require a live database connection; the
Swashbuckle CLI loads the built assembly's route/DI metadata directly.

No automation regenerates these on every contract change yet — there's no CI step wired up for it. Treat
"this snapshot looks stale" as a real but low-urgency finding, not a blocker, until that's built.
