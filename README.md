# lftp FTPS Deploy

Incremental FTPS deploy for GitHub Actions, built on **lftp**.

## Why another FTP deploy action?

Many shared hostings (typically ProFTPD, e.g. Czech **Hukot.net**) require
**TLS session reuse on the data channel**. The FTP library used by popular
deploy actions doesn't support it, so deploys die with:

```
Error: Client is closed because read ECONNRESET (data socket)
```

`lftp` handles TLS session reuse natively. On top of that, this action keeps
**both certificate-chain and hostname verification enabled** even on hostings
with imperfect TLS setups:

- **Leaf-only chains**: servers that don't send their intermediate certificate
  fail verification with "issuer unknown". The bundled CA file (ISRG Root X1 +
  Let's Encrypt intermediates R10–R14) completes the chain.
- **Wildcard one level up**: when the cert covers `*.hosting.tld` but the FTP
  host is `ftp.srv1.hosting.tld` (which a wildcard cannot match per RFC 6125),
  set `cert-host: srv1.hosting.tld` — the action aliases the server IP to that
  name in `/etc/hosts` and connects to it, so hostname verification stays ON
  instead of being switched off.

Sync is **incremental**: `mirror --only-newer` plus `git restore-mtime`
(so files keep their last-commit date instead of the checkout time — without
this, every CI run would re-upload everything).

The mirror deliberately runs **without `--delete`** — it will never remove
remote files, so deploying into a directory that also contains an existing
application (e.g. a PrestaShop installation) is safe.

If lftp gives up with `mirror: Fatal error: max-retries exceeded` (flaky
shared hosting), the action restarts the mirror — up to 3 attempts total.
Thanks to `--only-newer` each rerun continues where the previous one died.

## Usage

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # required for incremental sync (restore-mtime)

      - uses: PilarJ/lftp-ftps-deploy@v1
        with:
          server: ${{ secrets.FTP_SERVER }}        # ftp.example.com (no protocol)
          username: ${{ secrets.FTP_USERNAME }}
          password: ${{ secrets.FTP_PASSWORD }}
          server-dir: /www/                        # must end with /
          # optional:
          # local-dir: ./dist/
          # cert-host: srv1.hosting.tld
          # exclude: |
          #   ^\.git
          #   ^\.github/
          #   ^node_modules/
```

### Example: Hukot.net

Hukot's FTP host is `ftp.whX.hukot.net`, but the certificate is a
`*.hukot.net` wildcard (one level), so:

```yaml
      - uses: PilarJ/lftp-ftps-deploy@v1
        with:
          server: ${{ secrets.FTP_SERVER }}        # ftp.wh1.hukot.net
          username: ${{ secrets.FTP_USERNAME }}
          password: ${{ secrets.FTP_PASSWORD }}
          server-dir: /www/
          cert-host: wh1.hukot.net
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `server` | ✅ | | FTP host, no protocol prefix |
| `username` | ✅ | | FTP login |
| `password` | ✅ | | FTP password |
| `server-dir` | ✅ | | Remote dir, **must end with `/`** |
| `local-dir` | | `.` | Local dir to mirror |
| `cert-host` | | | Hostname the server cert actually covers (see above) |
| `ca-file` | | bundled LE | CA bundle for chain verification |
| `exclude` | | `.git`, `.github` | Newline-separated lftp `-x` regexes (relative paths) |
| `restore-mtime` | | `true` | Per-file commit dates for incremental sync (needs `fetch-depth: 0`) |
| `parallel` | | `4` | Parallel transfers |
| `max-retries` | | `2` | lftp `net:max-retries`; on "max-retries exceeded" the whole mirror restarts (max 3 attempts) |
| `insecure-tls` | | `false` | ⚠️ Disable cert verification (MITM risk, last resort) |
| `debug` | | `false` | Verbose lftp protocol log |

## Security notes

- Credentials are passed via `~/.netrc`, never on a command line.
- The password is additionally masked via `::add-mask::`.
- Certificate **and** hostname verification stay enabled by default;
  `insecure-tls` exists only as an explicit, loudly-warned escape hatch.
- Pin this action to a commit SHA if you want supply-chain guarantees:
  `uses: PilarJ/lftp-ftps-deploy@<sha> # v1`.

## License

[MIT](LICENSE)
