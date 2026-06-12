# Entire Plugin Index

The plugin catalog for the [Entire CLI](https://github.com/entireio/cli). `entire plugin search`, `install <name>`, `info`, and `browse` resolve bare plugin names through this repository.

```sh
entire plugin search            # browse the catalog
entire plugin install upgrade   # install by name
entire plugin index update      # force a refresh
```

## How it works

There is no hosted service. The CLI shallow-clones this repository into its user cache and re-pulls it on a TTL (default 24 hours) — any git server can host an index, and the CLI keeps working offline from its cached copy. The catalog is the single `index.json` file at the repository root.

Plugins themselves are standalone executables named `entire-<name>`. Installing one resolves the newest semver tag of its repository over the git protocol (`git ls-remote`), downloads the platform's release asset over HTTPS, verifies it against the release's `checksums.txt`, and places it in the CLI's managed plugin directory. See the [external commands documentation](https://github.com/entireio/cli/blob/main/docs/architecture/external-commands.md) for the full contract.

## index.json schema (version 1)

```json
{
  "version": 1,
  "plugins": [
    {
      "name": "upgrade",
      "repo_url": "https://github.com/entireio/entire-upgrade",
      "description": "Upgrade the system-installed Entire binary",
      "official": true,
      "platforms": ["darwin", "linux", "windows"]
    }
  ]
}
```

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | Bare plugin name (`upgrade` for `entire-upgrade`). Must be a valid dispatcher name: no path separators, no leading `-`, and the `agent-` prefix is reserved for the external agent protocol. |
| `repo_url` | yes | Full `https://` git URL of the plugin repository. Any git host works — GitHub, GitLab, Gitea/Forgejo, self-hosted. |
| `description` | no | One line, shown by `search`, `info`, and `browse`. |
| `official` | no | Marks Entire-maintained plugins. Defaults to `false`. |
| `platforms` | no | Supported GOOS values when the plugin doesn't ship the full matrix. Empty means all. |

Entries with invalid names or missing repo URLs are filtered by the CLI on load, not fatal — but don't rely on that.

## Adding a plugin

1. **Name and host your repository.** Convention: the repository basename is `entire-<name>`. Releases must be downloadable without authentication (public repository).
2. **Publish releases with conventional assets.** Tag with semver (`v0.1.0`) and attach per-platform assets named `entire-<name>_<version>_<os>_<arch>.tar.gz` (`.zip` on Windows) containing the `entire-<name>` binary, plus a `checksums.txt` (sha256sum format). goreleaser's defaults produce exactly this shape — see [entire-upgrade's `.goreleaser.yaml`](https://github.com/entireio/entire-upgrade/blob/main/.goreleaser.yaml) for a working template. Hosts with non-standard release URLs can declare a `download_url` template in `entire-plugin.yml` instead.
3. **Optionally commit `entire-plugin.yml`** at the repository root:

   ```yaml
   name: myplugin
   description: One-line description
   requires:                # other plugins this one needs (optional)
     - name: sem
       repo_url: https://github.com/entireio/entire-sem
       min_version: v0.2.0  # minimum only; no range syntax
   ```

4. **Open a pull request** adding your entry to `index.json`. Keep the list sorted by name.

Before submitting, verify the end-to-end flow works against your repository directly:

```sh
entire plugin install https://github.com/you/entire-myplugin --yes
entire myplugin
entire plugin remove myplugin
```

## Private and corporate catalogs

Organizations can host their own index (any git repository with an `index.json`) and point the CLI at it. Precedence, highest first:

1. `--index <url>` flag on the plugin subcommands
2. `ENTIRE_PLUGIN_INDEX_URL` environment variable
3. `plugins.index_url` in `.entire/settings.local.json` / `.entire/settings.json` — commit it to a repository to point every contributor at an internal catalog
4. This repository (the built-in default)

`plugins.index_ttl_hours` tunes refresh frequency (default 24).
