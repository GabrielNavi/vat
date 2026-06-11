<div align="center">
  <img src="assets/logo.svg" alt="VAT logo" width="100"/>
  <h1>VAT - Versatile Autoregistration Transformer</h1>
</div>

[![en](https://img.shields.io/badge/lang-en-blue.svg)](README.md)
[![es](https://img.shields.io/badge/lang-es-green.svg)](README.es.md)

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Debian package](https://img.shields.io/badge/package-versatile--autoreg--vat-brightgreen)](https://github.com/GabrielNavi/versatile-autoreg-vat)
[![Bash](https://img.shields.io/badge/shell-bash-89e051.svg)](https://www.gnu.org/software/bash/)
[![Platform: Linux](https://img.shields.io/badge/platform-Linux-lightgrey.svg)]()

VAT is the JSON inventory transformer for the VA* ecosystem. It behaves like a standard Unix filter: it reads an inventory `{"clients": [...]}` from stdin, applies a YAML preset pipeline, and writes the transformed JSON to stdout. It can rename, filter, truncate, anonymize, validate, or otherwise reshape inventory data before it reaches VAL hooks, an upper VAS, or any external consumer.

> Version in Spanish: [README.es.md](README.es.md)

---

## Table of Contents

- [Ecosystem](#ecosystem)
- [Quick Start](#quick-start)
- [Installed Files](#installed-files)
- [Configuration](#configuration)
- [Integration in VAC](#integration-in-vac)
- [Presets](#presets)
- [Pipeline Steps](#pipeline-steps)
- [Fail Modes](#fail-modes)
- [Integration in VAL](#integration-in-val)
- [Integration in VAF](#integration-in-vaf)
- [Package Information](#package-information)
- [Build](#build)
- [Reference](#reference)
- [License](#license)

---

## Ecosystem

```
VAC / VAF / VAL  ── JSON inventory ──►  VAT  ──►  hooks, upstream VAS, external consumers
```

| Package | Repository | Description |
|---------|------------|-------------|
| `versatile-autoreg-vas` | [vas](https://github.com/GabrielNavi/vas) | Inventory server |
| `versatile-autoreg-vac` | [vac](https://github.com/GabrielNavi/vac) | Autoregistration client |
| `versatile-autoreg-val` | [val](https://github.com/GabrielNavi/val) | Generic consumer with hooks |
| `versatile-autoreg-vaf` | [vaf](https://github.com/GabrielNavi/vaf) | Federated VAS server |
| `versatile-autoreg-vat` | [vat](https://github.com/GabrielNavi/vat) ← *this* | Inventory transformer |

---

## Quick Start

```bash
# Install
sudo dpkg -i versatile-autoreg-vat_0.3-4_all.deb
sudo apt-get -f install

# Inspect the available preset reference
less /usr/share/vat/presets/example.yaml

# Run a preset from the command line
echo "$INVENTORY" | vat-operate \
    --source-component VAL \
    --direction downstream \
    --preset centro_downstream_detallado
```

> **Dependencies:** `jq`, `yq`
> See the man page with `man vat-operate`.

To activate VAT inside a component, set:

```bash
USE_VAT=true
VAT_PRESET=centro_downstream_detallado
```

---

## Installed Files

| Path | Description |
|------|-------------|
| `/usr/bin/vat-operate` | Main transformer CLI |
| `/etc/vat/presets.d/` | Administrator preset directory |
| `/usr/share/vat/presets/example.yaml` | Reference preset with all pipeline steps documented |

**Reference documentation in the source tree:**

| Path | Description |
|------|-------------|
| `usr/share/vat/presets/example.en.yml` | English reference copy of the preset example |

---

## Configuration

VAT does not use a daemon configuration file. Presets are discovered from these locations, in order:

1. `/etc/vat/presets.d/*.yaml` - administrator presets
2. `/usr/share/vat/presets/*.yaml` - package-distributed presets

`vat-operate` matches a preset by name inside the discovered preset set. A preset may also be guarded by `match.source_component` and `direction`.

Recommended activation variables for the calling component:

```bash
USE_VAT=true
VAT_PRESET=centro_downstream_detallado
```

---

## Presets

A preset is a YAML document that defines one or more named pipelines. The source tree ships `example.yaml` as the canonical reference and `example.en.yml` as an English companion copy.

Search precedence:

1. `/etc/vat/presets.d/` - administrator presets
2. `/usr/share/vat/presets/` - package presets

A preset may define:

- `description` for logs and documentation.
- `direction` as `upstream`, `downstream`, or `both`.
- `enabled` as a global switch.
- `fail_mode` at preset or step level.
- `match.source_component` to constrain the calling component.
- `pipeline` as the ordered list of transformations.

Example invocation:

```bash
echo "$INVENTORY" | vat-operate \
    --source-component VAF \
    --direction upstream \
    --preset aula_upstream_minimal
```

---

## Pipeline Steps

| Step | Purpose |
|------|---------|
| `select_clients` | Filter which clients stay in the result set. |
| `filter_keys` | Whitelist or blacklist keys inside `extra_imperative` and `extra_informative`. |
| `transform` | Rename extra keys and optionally drop unmapped ones. |
| `truncate_lists` | Limit array sizes inside extras. |
| `anonymize` | Hash or redact selected fields in each client. |
| `limit_depth` | Restrict JSON nesting depth, with optional stringify or truncate strategies. |
| `json_schema_validation` | Validate clients against a JSON Schema. |
| `jq_arbitrary` | Apply an arbitrary `jq` filter to the whole inventory. |

The pipeline runs strictly in order. A common layout is: select first, filter and transform second, then truncate, anonymize, and validate at the end.

---

## Fail Modes

| Value | Behavior on error |
|---|---|
| `passthrough` | Return the original JSON unchanged. This is the default and recommended production mode. |
| `abort` | Exit with code 1 and let the caller decide. |
| `drop` | Return `{"clients": []}`. |

`fail_mode` can be set globally in `defaults.fail_mode`, overridden per preset, and overridden again by `json_schema_validation` when needed.

---

## Integration in VAC

```bash
# /etc/vac/vac.conf
SYNC_CLIENTS=true
USE_VAT=true
VAT_PRESET=centro_downstream_detallado
```

When `USE_VAT=true`, VAC applies VAT to `clients.json` after each inventory download and before any downstream consumer reads it. The transform runs with `--source-component VAC --direction downstream`.

```bash
# Fragment of the VAC download path when USE_VAT=true
if [[ "$USE_VAT" == "true" && -n "$VAT_PRESET" ]]; then
    CLIENTS_JSON="$(vat-operate \
        --source-component VAC \
        --direction downstream \
        --preset "$VAT_PRESET" < /var/lib/vac/clients.json)"
fi
```

---

## Integration in VAL

```bash
# /etc/val/val.conf
USE_VAT=true
VAT_PRESET=centro_downstream_detallado
```

```bash
# Fragment of the VAL loop when USE_VAT=true
if [[ "$USE_VAT" == "true" ]]; then
    INVENTORY="$(echo "$INVENTORY" | vat-operate \
        --source-component VAL \
        --direction downstream \
        --preset "$VAT_PRESET")"
fi

echo "$INVENTORY" | "$HOOK"
```

VAT runs before hooks are dispatched, so the hooks receive the transformed inventory.

---

## Integration in VAF

```bash
# /etc/vaf/vaf.conf
USE_VAT=true
VAT_PRESET=aula_upstream_minimal
```

```bash
# Fragment of the VAF loop when USE_VAT=true
if [[ "$USE_VAT" == "true" ]]; then
    NEW_IMP="$(echo "$NEW_IMP" | vat-operate \
        --source-component VAF \
        --direction upstream \
        --preset "$VAT_PRESET")"
fi

register_client "$LOC_HOST" "$LOC_IP" "$LOC_MAC" "$NEW_IMP" ""
```

VAT runs before the payload is sent to the upper VAS, so the published extras can be normalized or reduced in place.

---

## Package Information

- Name: `versatile-autoreg-vat`
- Version: `0.3-4`
- Architecture: `all`
- Maintainer: Gabriel Navia `<correos@gabrielnav.es>`
- License: Apache 2.0
- Repository: https://gitlab.vitalinux.educa.aragon.es/gabriel/versatile-autoreg-vat

---

## Build

```bash
dpkg-buildpackage -us -uc -b
```

---

## Reference

- `man vat-operate`
- `/usr/share/vat/presets/example.yaml`

---

## License

[Apache License 2.0](LICENSE)
