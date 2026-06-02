# versatile-autoreg-vat — Versatile Autoregistration Transformer

Transformador de inventarios JSON del ecosistema VA*. Opera como filtro Unix estándar: lee un inventario `{"clients": [...]}` desde stdin, aplica un pipeline de transformaciones definido en un preset YAML y escribe el resultado en stdout.

No es un daemon. No tiene servicio systemd. Se invoca puntualmente desde los bucles de VAL, VAF o cualquier componente que necesite transformar el inventario antes de entregarlo a su destino.

## Ecosistema

```
versatile-autoreg-vas          → servidor de registro canónico
versatile-autoreg-vac          → cliente de autoregistro
versatile-autoreg-val          → consumidor genérico (hooks) ← usa VAT antes del dispatch
versatile-autoreg-vaf          → federación de VAS en jerarquía ← usa VAT antes de upstream
versatile-autoreg-vat          → transformador de inventarios (este paquete)
```

## Instalación

```bash
dpkg -i versatile-autoreg-vat_0.1-1_all.deb
```

No requiere configuración para operar. Los presets se colocan en `/etc/vat/presets.d/`.

## Uso

```bash
# Desde línea de comandos
echo "$INVENTORY" | vat-operate \
    --source-component VAL \
    --direction downstream \
    --preset centro

# Desde el bucle de VAL (val.conf)
USE_VAT=true
VAT_PRESET=centro
```

Cuando `USE_VAT=true` está activo en el componente, el inventario pasa por `vat-operate` antes de llegar a los hooks (VAL) o antes de enviarse al VAS superior (VAF).

## Argumentos

| Argumento | Descripción |
|---|---|
| `--source-component` | Componente que invoca la transformación: `VAS` · `VAC` · `VAL` · `VAF` |
| `--direction` | Sentido del flujo: `upstream` · `downstream` · `both` |
| `--preset` | Nombre del preset a aplicar (sin extensión `.yaml`) |

## Presets

Un preset es un fichero YAML que define un pipeline de pasos de transformación. Se identifica por nombre (`--preset centro` carga `centro.yaml`).

**Precedencia de búsqueda:**

1. `/etc/vat/presets.d/<nombre>.yaml` — presets del administrador (mayor precedencia)
2. `/usr/share/vat/presets/<nombre>.yaml` — presets distribuidos con el paquete

## Rutas instaladas

| Ruta | Descripción |
|---|---|
| `/usr/bin/vat-operate` | Binario principal |
| `/etc/vat/presets.d/` | Directorio de presets del administrador |
| `/usr/share/vat/presets/example.yaml` | Preset de referencia con todos los pasos documentados |

## Estructura de un preset

```yaml
version: 1

defaults:
  fail_mode: passthrough   # passthrough | abort | drop
  log_level: normal        # no | normal | debug
  enabled: true

presets:

  mi_preset:
    description: "Descripción del preset"
    direction: downstream  # upstream | downstream | both
    enabled: true
    match:
      source_component: [VAL]   # VAS | VAC | VAL | VAF
    pipeline:
      - step: select_clients    # filtrar qué clientes pasan
        config:
          status: active
          limit: 100

      - step: filter_keys       # filtrar qué campos de cada cliente
        config:
          keep_metadata: true
          mode: whitelist       # whitelist | blacklist
          paths:
            imperative: ["clave_a"]
            informative: ["clave_b"]

      - step: transform         # renombrar claves en extras
        config:
          imperative:
            - from: "nombre_original"
              to:   "nombre_nuevo"
          defaults:
            - from: "*"
              drop: true        # eliminar claves no listadas

      - step: truncate_lists    # limitar tamaño de arrays en extras
        config:
          informative:
            - path: "printers"
              max_items: 5

      - step: anonymize         # anonimizar campos sensibles
        config:
          mode: hash            # hash | redact
          hash_algo: sha256
          deterministic: true
          fields:
            - "hostname"
            - "extra_imperative.user"

      - step: limit_depth       # limitar profundidad de anidamiento
        config:
          max_depth: 3
          strategy: stringify   # truncate | stringify

      - step: json_schema_validation
        config:
          schema: "/usr/share/vat/schemas/clients_v1.json"
          fail_mode: drop

      - step: jq_arbitrary      # filtro jq libre (escape hatch)
        config:
          filter: '.clients |= map(select(.ip != null))'
```

Ver `/usr/share/vat/presets/example.yaml` para la referencia completa con todos los pasos documentados.

## fail_mode

| Valor | Comportamiento ante error |
|---|---|
| `passthrough` | Devuelve el JSON original sin modificar (recomendado en producción) |
| `abort` | Sale con código 1; el componente decide qué hacer |
| `drop` | Devuelve `{"clients": []}` |

## Integración en VAL

```bash
# /etc/val/val.conf
USE_VAT=true
VAT_PRESET=centro
```

El daemon VAL aplica automáticamente la transformación antes de despachar los hooks:

```bash
# Fragmento del bucle de VAL (cuando USE_VAT=true)
if [[ "$USE_VAT" == "true" ]]; then
    INVENTORY="$(echo "$INVENTORY" | vat-operate \
        --source-component VAL \
        --direction downstream \
        --preset "$VAT_PRESET")"
fi
echo "$INVENTORY" | "$HOOK"
```

## Integración en VAF

```bash
# /etc/vaf/vaf.conf
USE_VAT=true
VAT_PRESET=aula_upstream_minimal
```

VAF aplica la transformación sobre el extra antes de enviarlo al VAS superior:

```bash
# Fragmento del bucle de VAF (cuando USE_VAT=true)
if [[ "$USE_VAT" == "true" ]]; then
    NEW_IMP="$(echo "$NEW_IMP" | vat-operate \
        --source-component VAF \
        --direction upstream \
        --preset "$VAT_PRESET")"
fi
register_client "$LOC_HOST" "$LOC_IP" "$LOC_MAC" "$NEW_IMP" ""
```

## Información del paquete

- Nombre: `versatile-autoreg-vat`
- Versión: 0.1-1
- Arquitectura: all
- Mantenedor: Gabriel Navia \<correos@gabrielnav.es\>
- Licencia: Apache 2.0
- Repositorio: https://gitlab.vitalinux.educa.aragon.es/gabriel/versatile-autoreg-vat

## Construcción del paquete

```bash
dpkg-buildpackage -us -uc -b
```
