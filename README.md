# OWASP Amass v5 Primer: Flags, Workflows & Skill Levels

Nov 22, 2026 · @Travis

Amass v5.1.3 (latest tag, October 2026) is a client/engine system, not the single-binary scanner most tutorials describe. `amass enum` hands a scope to a collection engine, the engine writes everything it finds into an asset database, and `subs`, `track`, `viz` and `assoc` read that database back out.

Every flag in this primer was checked against the v5.1.3 source code, not just the help text. Several flags still parse but do nothing in this release; they are marked **No effect** in the reference tables so you don't build workflows on them.

Read it top-down: architecture and install, then Beginner, Intermediate and Advanced workflows, then the complete flag reference for every subcommand. Run Amass only against domains and networks you own or have written permission to test.

## How Amass v5 works

&#91;embedded content: Amass v5 architecture · client, engine, database, readers\]

The engine does all collection and is the only part that talks to the internet. Because results live in the database, you can rerun `subs`, `track`, `viz` or `assoc` any time without re-scanning.

## Installation and first run

Pick one install method; all produce the same `amass` binary.

| Method | Command |
| --- | --- |
| Go (needs a current Go toolchain) | `CGO_ENABLED=0 go install -v github.com/owasp-amass/amass/v5/cmd/amass@main` |
| Homebrew | `brew tap owasp-amass/homebrew-amass && brew install amass` |
| Docker | `docker pull owaspamass/amass:latest` then `docker run --rm -it -v ~/.config/amass:/.config/amass owaspamass/amass:latest enum -d example.com` |
| Docker Compose (bundles Postgres) | `git clone https://github.com/owasp-amass/amass-docker-compose.git && cd amass-docker-compose && docker compose run --rm enum -d example.com` |

For Docker Compose, set the database passwords in `config/assetdb.env` and the matching value in `config/config.yaml` before the first run. The docs say changing them after the database exists is not supported.

The first time any subcommand runs, Amass creates its output directory and writes default `config.yaml` and `datasources.yaml` there if they are missing:

| OS | Default output directory |
| --- | --- |
| Linux | `~/.config/amass/` |
| macOS | `~/Library/Application Support/amass/` |
| Windows | `%AppData%\amass\` |

That directory also holds `asset.db` (the default SQLite asset database) and one `session-<uuid>.log` per enumeration. Override it with `-dir` on any subcommand.

Verify the install:

```bash
amass -version        # prints v5.1.3
amass -h              # lists subcommands
amass enum -h         # lists enum flags
```

## Beginner: run a passive enumeration and read the results

The beginner loop is two commands: `amass enum` collects, `amass subs` shows what was collected. In v5, `enum` does not print subdomains as it finds them; it shows a progress bar and then a "Session Scope" summary.

### 1. Run your first enumeration

```bash
amass enum -d example.com
```

- If no engine is running, `enum` starts one in the background and waits up to 60 seconds for it.
- Passive is the default: data sources, certificate logs, archives and DNS lookups. No ports are probed.
- The run ends when all work items finish, or after 30 minutes with no progress (`-timeout`). Ctrl+C stops it early; collected data is kept.

### 2. Read the results with `subs`

`subs` prints nothing unless you pick `-names`, `-summary` or `-show`.

```bash
amass subs -names -d example.com                 # just the subdomains
amass subs -names -ip -d example.com             # subdomains + IPv4/IPv6
amass subs -names -ipv4 -d example.com           # IPv4 only
amass subs -summary -d example.com               # ASN / netblock table only
amass subs -show -d example.com                  # names + ASN table
amass subs -names -d example.com -o subs.txt     # also write to a file
```

### 3. Feed it more than one domain

```bash
amass enum -d example.com,example.net
amass enum -d example.com -d example.org
amass enum -df roots.txt        # one root domain per line
```

### 4. Control the console

```bash
amass enum -d example.com -nocolor     # plain text for logs/CI
amass enum -d example.com -silent      # no output at all
amass enum -d example.com -v           # verbose status / debug info
```

### 5. Add API keys (the biggest quality jump)

Passive results depend on data sources. Open `datasources.yaml` in the output directory, uncomment a source, and paste its key:

```yaml
datasources:
  - name: SecurityTrails
    creds:
      account:
        apikey: YOUR_KEY
```

In v5.1.3 the engine ships API plugins for Aviato, BinaryEdge, Chaos, crt.sh, DNSRepo, GLEIF, grep.app, HackerTarget, Hunter.io, LeakIX, PassiveTotal, Prospeo, RDAP, SecurityTrails, VirusTotal and ZETAlytics. The default `datasources.yaml` still lists about 40 older names; keys for sources without a plugin are ignored.

### Beginner flags at a glance

| Command | Flag | Use it to |
| --- | --- | --- |
| `enum` | `-d` | Name the root domain(s) |
| `enum` | `-df` | Load root domains from a file |
| `enum` | `-v`, `-silent`, `-nocolor` | Control console output |
| `subs` | `-names` | List discovered subdomains |
| `subs` | `-ip`, `-ipv4`, `-ipv6` | Add resolved addresses |
| `subs` | `-summary`, `-show` | Add the ASN table |
| `subs` | `-o` | Save output to a text file |
| all | `-h` | Show help for that subcommand |

## Intermediate: active collection, scope control and change tracking

At this level you decide how much Amass touches the target, what counts as in scope, and how results change over time.

### 1. Go active (with authorization)

```bash
amass enum -active -d example.com
amass enum -active -p 80,443,8443,9443 -d example.com
```

In v5.1.3, `-active` does two things in the engine: it turns on HTTP/TLS probes against discovered hosts on the scope ports, and it widens reverse-DNS sweeps around in-scope IPs from 100 to 250 addresses. That means traffic hits the target. `-p` is refused unless `-active` is set. Default ports come from `config.yaml` (80, 443, 8000, 8080, 8443, 8888 in the shipped file).

### 2. Seed the scope with network ranges

Seeds give the engine starting points beyond the root domain:

```bash
amass enum -d example.com -cidr 203.0.113.0/24,198.51.100.0/25
amass enum -d example.com -addr 203.0.113.10-50
amass enum -d example.com -asn 64500,64501
```

### 3. Keep the scope tight

By default, Amass expands scope when it finds related organizations, registration records, certificates and locations (the "horizontal" plugins). Turn that off for bounded tests:

```bash
amass enum -rigid -d example.com
amass enum -d example.com -bl dev.example.com,old.example.com
amass enum -d example.com -blf out-of-scope.txt
```

`-bl` / `-blf` stop listed subdomains from being investigated.

### 4. Use your own resolvers

```bash
amass enum -d example.com -r 1.1.1.1,9.9.9.9
amass enum -d example.com -rf resolvers.txt
```

The built-in rate is 5 queries/second per public resolver and 15 per trusted resolver. In v5.1.3, `-tr` writes to the same list as `-r`, so the CLI no longer separates trusted from untrusted.

### 5. Import names from other tools

```bash
subfinder -d example.com -silent > known.txt
amass enum -d example.com -nf known.txt
```

`-nf` bulk-uploads known FQDNs to the engine so it resolves and expands them.

### 6. Move flags into a config file

```bash
amass enum -config ./engagement.yaml -dir ./acme-2026q4
```

A minimal `engagement.yaml`:

```yaml
scope:
  domains: [example.com]
  cidrs: [203.0.113.0/24]
  ports: [80, 443, 8443]
  blacklist: [legacy.example.com]
options:
  datasources: "./datasources.yaml"
  rigid_boundaries: true
```

Command-line flags override the file. Using a separate `-dir` per engagement keeps each client's `asset.db` and logs apart. Pass the same `-dir` to `subs`, `track`, `viz` and `assoc` later.

### 7. Track what changed

```bash
amass track -d example.com
amass track -d example.com -since '11/01 00:00:00 2026 EST'
```

Without `-since`, `track` shows names first seen on the most recent collection day. The time format is fixed: `MM/DD HH:MM:SS YYYY TZ`.

### 8. Tune the run

```bash
amass enum -d example.com -timeout 90 -log ./acme.log
```

`-timeout` is minutes without progress, not total runtime. `-log` replaces the per-session log file name.

## Advanced: engine operations, graph queries, alterations and automation

Advanced use treats Amass as an attack-surface database: run the engine as a service, back it with Postgres, and query the graph directly.

### 1. Run the engine yourself

```bash
amass engine -log-dir /var/log/amass &        # long-running collector
amass enum -d example.com                       # reuses the running engine
amass enum -engine http://10.0.5.20:4000 -d example.com   # remote engine
```

The engine's REST API listens on port 4000 on **all interfaces** (`:4000` in the source), and the code shows no authentication on it. Firewall it or run it in a private network. The engine also exposes Go pprof on `127.0.0.1:6060`. You can set a default remote engine in `config.yaml` under `options.engine`.

### 2. Swap SQLite for Postgres (or Neo4j)

Set the database URI in `config.yaml`:

```yaml
options:
  database: "postgres://amass:CHANGE_ME@assetdb:5432/assetdb"
  # database: "bolt://neo4j:CHANGE_ME@neo4j:7687/neo4j"
```

The config loader also reads `AMASS_DB_USER`, `AMASS_DB_PASSWORD`, `AMASS_DB_HOST` (default localhost), `AMASS_DB_PORT` (5432) and `AMASS_DB_NAME` (assetdb) to build a Postgres URI, which keeps passwords out of the YAML. Postgres suits multi-engagement history and SQL reporting; the Compose project sets it up for you.

### 3. Query the graph with `assoc` triples

`assoc` walks the Open Asset Model graph. Each triple is `<subject> - <predicate> -> <object>` (or `<-` for incoming), and each element takes a constant, `*`, or a `#/regex/#`.

```bash
# IPs for the root domain, seen since July 1
amass assoc -t1 '<fqdn:owasp.org> - <*:dns_record,since:2025-07-01> -> <ipaddress:*>'

# Two-hop walk: subdomains, then their IPs
amass assoc -t1 '<fqdn:owasp.org> - <*:node> -> <*>' \
            -t2 '<fqdn:*> - <*:dns_record> -> <ipaddress:*>'

# Only answers in 203.0.113.x
amass assoc -t1 '<fqdn:example.com> - <*:dns_record> -> <ipaddress:#/203\.0\.113\..*/#>'

# Long walks from a file (one triple per line)
amass assoc -tf walk.txt
```

Output is JSON with each step and its properties, so pipe it to `jq`. Property filters look like `prop:[sourceproperty:DNS-IP,since:2025-07-01,confidence:80]`.

### 4. Visualize the graph

```bash
amass viz -d3 -d example.com -o ./graphs            # interactive HTML (D3 v4)
amass viz -dot -d example.com -oA acme              # Graphviz DOT, acme.* prefix
amass viz -gexf -d example.com -since '11/01 00:00:00 2026 EST'   # Gephi
```

GEXF opened in Gephi handles large graphs best; D3 is good for a quick look in a browser.

### 5. Name alterations

Alterations generate permutations (word flips, added words and numbers) from names already found. In v5.1.3 the plugin runs only when **both** `-brute` and `-alts` are set:

```bash
amass enum -d example.com -brute -alts
amass enum -d example.com -brute -alts -aw my-alts.txt
```

Mask syntax for `-wm` / `-awm` is hashcat-style: `?d` digits, `?l` or `?u` lowercase letters, `?s` hyphen, `?a` all three; at most 3 per word (for example `dev?d?d`). In v5.1.3 the expansion code sits in a function the CLI never calls, so verify masks expand before relying on them (see Gotchas).

### 6. Tune re-query intervals with transformations

`config.yaml` has a `transformations` block keyed `From->To` (for example `FQDN->DomainRecord`). Each `ttl` is in minutes and tells the engine how long a finding stays fresh before that plugin queries it again. Lower it for frequent monitoring; raise it to spare API quotas.

```yaml
transformations:
  FQDN->ALL:
    ttl: 720        # re-check FQDNs every 12 hours
  TLSCertificate->ALL:
    ttl: 10080      # 7 days
```

### 7. Automate continuous monitoring

```bash
#!/usr/bin/env bash
# nightly.sh - run from cron
D=/srv/amass/acme
amass enum -silent -dir "$D" -config "$D/config.yaml" -df "$D/roots.txt"
amass track -nocolor -dir "$D" -df "$D/roots.txt" > "$D/new-$(date +%F).txt"
amass subs -names -nocolor -dir "$D" -df "$D/roots.txt" -o "$D/all.txt"
```

Send `track` output to your alerting or SIEM. To ship engine logs centrally, set `SYSLOG_HOST`, `SYSLOG_PORT` (default 514) and `SYSLOG_TRANSPORT` (default udp).

### 8. Other binaries in the repo

The `cmd/` folder also builds standalone `oam_enum`, `oam_subs`, `oam_track`, `oam_viz`, `oam_assoc`, `amass_engine`, `ae_isready` (engine health check for scripts) and `oam_i2y`, which converts a legacy v3/v4 INI config to v5 YAML.

## Flag reference: `amass enum`

Usage: `amass enum [options] -d DOMAIN`. At least one root domain (`-d`, `-df` or the config file) is required. Status is what the flag does in v5.1.3 code: **Works**, **Partial** (works with a condition), or **No effect** (parsed, then never used).

### Targets and scope

| Flag | Default | What it does | Example | Level | Status |
| --- | --- | --- | --- | --- | --- |
| `-d` | none | Root domain(s); comma-separated, repeatable | `-d example.com,example.net` | Beginner | Works |
| `-df` | none | File of root domains, one per line; repeatable | `-df roots.txt` | Beginner | Works |
| `-addr` | none | Seed IPs and ranges | `-addr 203.0.113.1-254` | Intermediate | Works |
| `-cidr` | none | Seed CIDRs; repeatable | `-cidr 203.0.113.0/24` | Intermediate | Works |
| `-asn` | none | Seed ASNs; repeatable | `-asn 64500` | Intermediate | Works |
| `-p` | 80,443 (config file adds 8000, 8080, 8443, 8888) | Ports for active HTTP/TLS probes | `-active -p 443,8443` | Intermediate | Partial: requires `-active` |
| `-bl` | none | Subdomains never to investigate | `-bl vpn.example.com` | Intermediate | Works |
| `-blf` | none | File of blacklisted subdomains | `-blf oos.txt` | Intermediate | Works |
| `-nf` | none | File of already-known FQDNs to load into the session | `-nf subfinder.txt` | Intermediate | Works |
| `-rigid` | off | Disable scope expansion via related orgs, records, certs, locations | `-rigid` | Intermediate | Works |

### Collection behavior

| Flag | Default | What it does | Example | Level | Status |
| --- | --- | --- | --- | --- | --- |
| `-active` | off | HTTP/TLS probes on scope ports; reverse-DNS sweeps grow to 250 IPs | `-active` | Intermediate | Works (help text still says "zone transfers"; no zone-transfer code found) |
| `-passive` | n/a | Old passive switch | none | none | No effect: deprecated, passive is the default |
| `-brute` | off | In v5.1.3 only gates the alterations plugin | `-brute -alts` | Advanced | Partial: no dictionary brute-force plugin in the engine |
| `-alts` | off | Generate altered names from found names | `-brute -alts` | Advanced | Partial: requires `-brute` |
| `-w` | built-in list | Wordlist file(s) for brute forcing; repeatable | `-brute -w top5000.txt` | Advanced | No effect: loaded, no engine plugin reads it |
| `-wm` | none | Hashcat-style masks for brute forcing | `-brute -wm 'api?d'` | Advanced | No effect: same as `-w` |
| `-aw` | `alterations.txt` | Wordlist file(s) for alterations; repeatable | `-brute -alts -aw words.txt` | Advanced | Works |
| `-awm` | none | Hashcat-style masks for alterations | `-brute -alts -awm 'v?d'` | Advanced | Partial: masks may stay unexpanded (see Gotchas) |
| `-max-depth` | 0 (no limit) | Max subdomain labels for brute forcing | `-max-depth 3` | Advanced | No effect |
| `-min-for-recursive` | 1 | Labels seen before recursive brute forcing | `-min-for-recursive 2` | Advanced | No effect |
| `-norecursive` | off | Turn off recursive brute forcing | `-norecursive` | Advanced | No effect |
| `-timeout` | 30 | Minutes without progress before the client stops | `-timeout 120` | Intermediate | Works |
| `-include` | all | Only use these data sources | `-include crtsh,RDAP` | Intermediate | No effect: parsed, not sent to the engine |
| `-exclude` | none | Skip these data sources | `-exclude VirusTotal` | Intermediate | No effect: parsed, not sent to the engine |
| `-list` | off | Print available data sources | `-list` | Beginner | No effect: prints nothing extra |

### DNS and network

| Flag | Default | What it does | Example | Level | Status |
| --- | --- | --- | --- | --- | --- |
| `-r` | built-in public list | Resolver IPs (5 qps each); repeatable | `-r 1.1.1.1,9.9.9.9` | Intermediate | Works |
| `-tr` | none | "Trusted" resolver IPs | `-tr 10.0.0.53` | Intermediate | Partial: goes into the same list as `-r` |
| `-rf` | none | File of resolver IPs | `-rf resolvers.txt` | Intermediate | Works |
| `-iface` | system default | Network interface for traffic | `-iface eth1` | Advanced | No effect: use OS routing or a namespace |
| `-engine` | `http://127.0.0.1:4000` | Engine API URL to submit the session to | `-engine http://10.0.5.20:4000` | Advanced | Works |

### Files, config and output

| Flag | Default | What it does | Example | Level | Status |
| --- | --- | --- | --- | --- | --- |
| `-config` | `config.yaml` in output dir | YAML config file | `-config acme.yaml` | Intermediate | Works |
| `-dir` | OS config dir + `/amass` | Output directory (DB, logs, configs) | `-dir ./acme` | Intermediate | Works |
| `-log` | `session-<uuid>.log` | Log file name in the output dir | `-log acme.log` | Intermediate | Works |
| `-oA` | none | Prefix for all output files | `-oA acme` | Intermediate | No effect in `enum` (works in `viz`) |
| `-demo` | off | Censor output for demos | `-demo` | Beginner | No effect in `enum` (works in `subs`) |
| `-v` | off | Verbose status / debug output | `-v` | Beginner | Works |
| `-silent` | off | No console output | `-silent` | Beginner | Works |
| `-nocolor` | off | Plain output | `-nocolor` | Beginner | Works |
| `-h`, `-help` | n/a | Show usage | `amass enum -h` | Beginner | Works |

## Flag reference: global, `engine`, `subs`, `track`, `viz`, `assoc`

All flags in this section work as described in v5.1.3. `-dir` and `-config` on the reader subcommands must point at the same output directory and database the enumeration used.

### Global: `amass [assoc|engine|enum|subs|track|viz] [options]`

| Flag | What it does | Example |
| --- | --- | --- |
| `-h`, `-help` | List subcommands | `amass -h` |
| `-version` | Print the version | `amass -version` |

### `amass engine`: run the collection engine

| Flag | What it does | Example | Level |
| --- | --- | --- | --- |
| `-log-dir` | Directory for engine logs | `amass engine -log-dir /var/log/amass` | Advanced |
| `-silent` | No console output | `amass engine -silent` | Advanced |
| `-nocolor` | Plain output | `amass engine -nocolor` | Advanced |
| `-h`, `-help` | Show usage | `amass engine -h` | Beginner |

`amass engine` refuses to start if an engine is already running.

### `amass subs`: show discovered subdomains

| Flag | What it does | Example | Level |
| --- | --- | --- | --- |
| `-d` | Domain(s) to report; defaults to the config's domains | `-d example.com` | Beginner |
| `-df` | File of domains to report | `-df roots.txt` | Beginner |
| `-names` | Print discovered names | `subs -names -d example.com` | Beginner |
| `-summary` | Print only the ASN/netblock summary table | `subs -summary -d example.com` | Beginner |
| `-show` | Names plus the ASN table (`-names` + `-summary`) | `subs -show -d example.com` | Beginner |
| `-ip` | Add IPv4 and IPv6 addresses | `subs -names -ip -d example.com` | Beginner |
| `-ipv4` | Add IPv4 addresses only | `subs -names -ipv4 -d example.com` | Beginner |
| `-ipv6` | Add IPv6 addresses only | `subs -names -ipv6 -d example.com` | Beginner |
| `-demo` | Censor names for screenshots and talks | `subs -names -demo -d example.com` | Intermediate |
| `-o` | Also write terminal output to a text file (overwrites) | `-o subs.txt` | Beginner |
| `-dir` | Output directory holding the database | `-dir ./acme` | Intermediate |
| `-config` | YAML config (database settings) | `-config acme.yaml` | Intermediate |
| `-silent`, `-nocolor`, `-h` | Output control and help | `-nocolor` | Beginner |

### `amass track`: find newly discovered assets

| Flag | What it does | Example | Level |
| --- | --- | --- | --- |
| `-d` | Domain(s) to compare | `track -d example.com` | Intermediate |
| `-df` | File of domains | `track -df roots.txt` | Intermediate |
| `-since` | Show only assets first seen after this time; format `MM/DD HH:MM:SS YYYY TZ`; default = latest collection day | `-since '11/01 00:00:00 2026 EST'` | Intermediate |
| `-dir`, `-config` | Database location / config | `-dir ./acme` | Intermediate |
| `-silent`, `-nocolor`, `-h` | Output control and help | `-nocolor` | Intermediate |

### `amass viz`: export graph visualizations

Usage: `amass viz -d3|-dot|-gexf [options] -d domain`.

| Flag | What it does | Example | Level |
| --- | --- | --- | --- |
| `-d3` | Interactive D3 v4 force-graph HTML | `viz -d3 -d example.com` | Advanced |
| `-dot` | Graphviz DOT file | `viz -dot -d example.com` | Advanced |
| `-gexf` | Gephi GEXF file | `viz -gexf -d example.com` | Advanced |
| `-d`, `-df` | Domain(s) to graph | `-df roots.txt` | Advanced |
| `-since` | Only assets validated after this time (same format as `track`) | `-since '11/01 00:00:00 2026 EST'` | Advanced |
| `-o` | Directory for generated files | `-o ./graphs` | Advanced |
| `-oA` | File name prefix for all outputs | `-oA acme-q4` | Advanced |
| `-dir`, `-config` | Database location / config | `-dir ./acme` | Advanced |
| `-silent`, `-nocolor`, `-h` | Output control and help | `-silent` | Advanced |

### `amass assoc`: walk the asset graph

Usage: `amass assoc [options] [-tf path] [-t1 triple] ... [-t10 triple]`.

| Flag | What it does | Example | Level |
| --- | --- | --- | --- |
| `-t1` to `-t10` | Up to 10 chained triples; each step's results feed the next | `-t1 '<fqdn:example.com> - <*:node> -> <*>'` | Advanced |
| `-tf` | File containing a list of triples | `-tf walk.txt` | Advanced |
| `-dir`, `-config` | Database location / config | `-dir ./acme` | Advanced |
| `-silent`, `-nocolor`, `-h` | Output control and help | `-nocolor` | Advanced |

Triple elements: `type:label` (for example `fqdn:example.com`, `*:dns_record`), `*` wildcard, `#/regex/#`, plus optional `since:YYYY-MM-DD` and `prop:[...]` filters. Attributes can be matched too, such as `header.rr_type:#/1|28/#` for A and AAAA records.

## Gotchas: v4 habits that break in v5.1.3

Most Amass guides online were written for v3/v4. These differences cause the most confusion; each was confirmed in the v5.1.3 source.

| What you expect (v4) | What happens in v5.1.3 | What to do |
| --- | --- | --- |
| `enum` prints subdomains live | Shows a progress bar, then a "Session Scope" list | Run `amass subs -names -d …` after (or alongside) |
| `amass intel` for org/ASN/whois discovery | Subcommand removed | Seed with `-asn` / `-cidr`; leave `-rigid` off so horizontal plugins expand scope; query with `assoc` |
| `amass db` to list past runs | Subcommand removed | Use `subs`, `track` and `assoc` against `-dir` |
| `-o`, `-json` on `enum` | Flags gone | Use `subs -o`, `assoc` JSON, or query the asset DB |
| `-brute` runs a DNS dictionary attack | No dictionary brute-force plugin in the engine; `-brute` only gates alterations | Use a dedicated resolver tool (puredns, shuffledns) and feed results back with `-nf` |
| `-w`, `-wm`, `-max-depth`, `-min-for-recursive`, `-norecursive` | Parsed; values are never used by the engine | Don't rely on them |
| `-include`, `-exclude`, `-list` | Parsed; not sent to the engine | Control sources by which keys you put in `datasources.yaml` |
| `-iface` | Parsed; not applied | Route traffic with the OS, a VPN or a network namespace |
| `-oA`, `-demo` on `enum` | Parsed; not applied | Use them on `viz` (`-oA`) and `subs` (`-demo`) |
| `-tr` separates trusted resolvers | Merged into the `-r` list | Put only resolvers you trust in `-r` / `-rf` |
| Masks in `-wm` / `-awm` expand | Expansion lives in `CheckSettings()`, which no code path calls | Pre-expand masks into a wordlist file and pass it with `-aw` |
| `-alts` works alone | Alterations plugin needs `-brute` **and** `-alts` | Always pass both |
| `-active` tries zone transfers | No zone-transfer code found; active = HTTP/TLS probes + wider PTR sweeps | Run AXFR checks with `dig axfr` separately |
| Legacy INI config | Not read | Convert with `oam_i2y` |
| Engine is local-only | REST API binds `:4000` on all interfaces | Firewall port 4000 |

These reflect the source at tag v5.1.3. Check the [releases page](https://github.com/owasp-amass/amass/releases) after upgrading; several "No effect" flags look like they are waiting to be rewired to the new engine.

## Authorization, OPSEC and operating notes

Get written scope before any run, including passive ones, and record the exact flags you used.

- **Passive still leaves traces.** Every query to a data source tells that provider which domain you are researching. Use separate API accounts per engagement if that matters.
- **Active touches the target.** `-active` sends HTTP/TLS requests on every scope port and sweeps up to 250 addresses around in-scope IPs. Confirm IP ranges are owned by the client, not a shared host or CDN.
- **Scope expansion can wander.** Without `-rigid`, Amass follows related organizations and registration records. Pair it with `-bl` / `-blf` to protect out-of-scope assets.
- **Protect secrets.** `datasources.yaml` holds API keys and `config.yaml` may hold database passwords. Restrict them with `chmod 600` and keep them out of git.
- **Protect the engine.** Block inbound port 4000 on any host running `amass engine`; the API binds all interfaces.
- **Separate engagements.** One `-dir` per client keeps each `asset.db`, log and config isolated and makes evidence handoff simple.
- **Mind resolver load.** Defaults are 5 queries/second per public resolver. Large resolver lists multiply that; follow each resolver's acceptable-use terms.

## Sources

- [owasp-amass/amass on GitHub](https://github.com/owasp-amass/amass), source at tag v5.1.3: `cmd/amass/main.go`, `internal/*/cli.go`, `config/`, `engine/plugins/`, `resources/config.yaml`, `resources/datasources.yaml`
- [Amass releases](https://github.com/owasp-amass/amass/releases)
- [Amass documentation: installation](https://owasp-amass.github.io/docs/)
- [Amass documentation: triples syntax for assoc](https://owasp-amass.github.io/docs/asset_db/triples/)
