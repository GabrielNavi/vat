<div align="center">
  <img src="assets/logo.svg" alt="VAT logo" width="100"/>
  <h1>VAT - Versatile Autoregistration Transformer</h1>
</div>

[![en](https://img.shields.io/badge/lang-en-blue.svg)](README.md)
[![es](https://img.shields.io/badge/lang-es-green.svg)](README.es.md)

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Debian package](https://img.shields.io/badge/package-versatile--autoreg--vat-brightgreen)](https://github.com/GabrielNavi/vat/releases)
[![Bash](https://img.shields.io/badge/shell-bash-89e051.svg)](https://www.gnu.org/software/bash/)
[![Platform: Linux](https://img.shields.io/badge/platform-Linux-lightgrey.svg)]()

VAT es el transformador de inventarios JSON del ecosistema VA*. Funciona como un filtro Unix estandar: lee un inventario `{"clients": [...]}` desde stdin, aplica un pipeline definido en un preset YAML y escribe el JSON transformado en stdout. Puede renombrar, filtrar, truncar, anonimizar, validar o reestructurar el inventario antes de entregarlo a hooks de VAL, a un VAS superior o a cualquier consumidor externo.

---

## Tabla de contenidos

- [Ecosistema](#ecosistema)
- [Instalacion rapida](#instalacion-rapida)
- [Archivos instalados](#archivos-instalados)
- [Configuracion](#configuracion)
- [Integracion en VAC](#integracion-en-vac)
- [Presets](#presets)
- [Pasos del pipeline](#pasos-del-pipeline)
- [Modos de fallo](#modos-de-fallo)
- [Integracion en VAL](#integracion-en-val)
- [Integracion en VAF](#integracion-en-vaf)
- [Informacion del paquete](#informacion-del-paquete)
- [Construccion](#construccion)
- [Referencia](#referencia)
- [Licencia](#licencia)

---

## Ecosistema

```
VAC / VAF / VAL  ── inventario JSON ──►  VAT  ──►  hooks, VAS superior, consumidores externos
```

| Paquete | Repositorio | Descripcion |
|---------|-------------|-------------|
| `versatile-autoreg-vas` | [vas](https://github.com/GabrielNavi/vas) | Servidor de inventario |
| `versatile-autoreg-vac` | [vac](https://github.com/GabrielNavi/vac) | Cliente de autoregistro |
| `versatile-autoreg-val` | [val](https://github.com/GabrielNavi/val) | Consumidor generico con hooks |
| `versatile-autoreg-vaf` | [vaf](https://github.com/GabrielNavi/vaf) | Servidor VAS federado |
| `versatile-autoreg-vat` | [vat](https://github.com/GabrielNavi/vat) ← *este* | Transformador de inventarios |

---

## Instalacion rapida

```bash
# Instalar
sudo dpkg -i versatile-autoreg-vat_0.3-4_all.deb
sudo apt-get -f install

# Preparar un preset de ejemplo
sudo mkdir -p /etc/vat/presets.d
sudo cp /usr/share/vat/presets/example.yaml /etc/vat/presets.d/centro_downstream_detallado.yaml

# Ejecutar sobre un inventario
echo "$INVENTORY" | vat-operate \
    --source-component VAL \
    --direction downstream \
    --preset centro_downstream_detallado
```

> **Dependencias:** `jq`, `yq` · `python3-jsonschema` (opcional, solo si se usa el paso de validacion de esquema)
> Los presets se buscan primero en `/etc/vat/presets.d/` y despues en `/usr/share/vat/presets/`.

---

## Archivos instalados

| Ruta | Descripcion |
|------|-------------|
| `/usr/bin/vat-operate` | CLI principal del transformador |
| `/etc/vat/presets.d/` | Directorio de presets del administrador |
| `/usr/share/vat/presets/example.yaml` | Preset de referencia con todos los pasos documentados |
| `/usr/share/man/man8/vat-operate.8.gz` | Pagina de manual |

---

## Configuracion

VAT no necesita fichero de configuracion propio. Su comportamiento depende del preset elegido y del componente que lo invoca.

```bash
# Activacion en el componente
USE_VAT=true
VAT_PRESET=centro_downstream_detallado

# Override opcional del nivel de log
VAT_LOG_LEVEL=debug
```

Los presets del administrador viven en `/etc/vat/presets.d/` y los de referencia distribuidos con el paquete en `/usr/share/vat/presets/`.

---

## Presets

Un preset es una definicion YAML de un pipeline de transformacion. Se selecciona con `--preset <nombre>` y puede estar en cualquier fichero `.yaml` dentro de los directorios de presets.

**Precedencia de busqueda:**

1. `/etc/vat/presets.d/*.yaml` - presets del administrador, con mayor precedencia
2. `/usr/share/vat/presets/*.yaml` - presets distribuidos con el paquete

Un preset puede filtrar por `source_component` (`VAS`, `VAC`, `VAL`, `VAF`) y por `direction` (`upstream`, `downstream`, `both`). Si el preset no coincide con la invocacion actual, VAT hace passthrough.

---

## Pasos del pipeline

El pipeline se ejecuta en el orden definido en el preset. El orden recomendado es:

1. `select_clients` - reducir el inventario cuanto antes.
2. `filter_keys` - mantener o eliminar claves concretas de los extras.
3. `transform` - renombrar o descartar claves.
4. `truncate_lists` - limitar el tamano de listas dentro de los extras.
5. `anonymize` - hashear o redactar campos sensibles.
6. `limit_depth` - limitar la profundidad JSON.
7. `json_schema_validation` - validar el payload final.
8. `jq_arbitrary` - aplicar un filtro jq libre como escape hatch.

El preset de ejemplo en `/usr/share/vat/presets/example.es.yaml` documenta cada paso con detalle.

---

## Modos de fallo

| Valor | Comportamiento ante error |
|---|---|
| `passthrough` | Devuelve el JSON original sin modificar (recomendado en produccion) |
| `abort` | Sale con codigo 1 y deja que el caller gestione el fallo |
| `drop` | Devuelve `{"clients": []}` |

---

## Integracion en VAC

```bash
# /etc/vac/vac.conf
SYNC_CLIENTS=true
USE_VAT=true
VAT_PRESET=clients_local_filtered
```

VAC aplica VAT en `--direction downstream` para transformar `clients.json` tras cada descarga, antes de que cualquier consumidor downstream lo lea.

```bash
# Fragmento de la descarga de VAC (vac-common.sh:354-365)
if [[ "$USE_VAT" == "true" && -n "$VAT_PRESET" ]]; then
    if command -v vat-operate &>/dev/null; then
        vat_out="$(vat-operate --source-component VAC --direction downstream \
            --preset "$VAT_PRESET" < "$CLIENTS_FILE" 2>/dev/null)" \
        && echo "$vat_out" > "${CLIENTS_FILE}.tmp" \
        && mv "${CLIENTS_FILE}.tmp" "$CLIENTS_FILE" \
        && log "[VAT] clients.json saneado con preset '$VAT_PRESET'"
    fi
fi
```

---

## Integracion en VAL

VAL aplica VAT en dos puntos distintos:

**Upstream (al llegar):** Normaliza clients.json al recibirlo de la capa superior (val-common.sh:196-206).

```bash
# /etc/val/val.conf
USE_VAT=true
VAT_PRESET=normalize_on_arrival

# Aplicado con --direction upstream
if [[ "$USE_VAT" == "true" && -n "$VAT_PRESET" ]]; then
    vat_out="$(vat-operate --source-component VAL --direction upstream \
        --preset "$VAT_PRESET" < "$CLIENTS_FILE" 2>/dev/null)" \
    && echo "$vat_out" > "${CLIENTS_FILE}.tmp" && mv "${CLIENTS_FILE}.tmp" "$CLIENTS_FILE"
fi
```

**Downstream (al servir a hooks):** Filtra clientes antes de despacharse a los manejadores de hook locales (val-common.sh:231-241).

```bash
# Misma variable de preset; aplicada separadamente con --direction downstream
if [[ "$USE_VAT" == "true" && -n "$VAT_PRESET" ]]; then
    vat_out="$(vat-operate --source-component VAL --direction downstream \
        --preset "$VAT_PRESET" < "${key}_clients.json" 2>/dev/null)" \
    && echo "$vat_out" > "${out}.tmp" && mv "${out}.tmp" "$out"
fi
echo "$INVENTORY" | "$HOOK"
```

---

## Integracion en VAF

VAF aplica VAT en ambas direcciones:

**Upstream:** Normalizar o reducir el inventario local antes de registrarse en el VAS superior (vaf:202-206).

```bash
# /etc/vaf/vaf.conf
USE_VAT=true
VAT_PRESET=aula_upstream_minimal
```

```bash
# Fragmento del bucle de registro de VAF (upstream)
if [[ "$USE_VAT" == "true" && -n "$VAT_PRESET" ]] && command -v vat-operate &>/dev/null; then
    NEW_IMP="$(echo "$NEW_IMP" | vat-operate \
        --source-component VAF --direction upstream \
        --preset "$VAT_PRESET" 2>/dev/null)"
fi
register_client "$LOC_HOST" "$LOC_IP" "$LOC_MAC" "$NEW_IMP" ""
```

**Downstream:** Filtrar la réplica de la base de datos antes de almacenarla localmente (vaf-common.sh:624-630).

```bash
# /etc/vaf/vaf.conf
USE_VAT=true
VAT_PRESET=aula_downstream_detallado
```

```bash
# Fragmento del procesamiento de la réplica BD en VAF (downstream)
if [[ "$USE_VAT" == "true" && -n "$VAT_PRESET" ]] && command -v vat-operate &>/dev/null; then
    vat_out="$(vat-operate --source-component VAF --direction downstream \
        --preset "$VAT_PRESET" < "$TMP_UPPER_CLIENTS" 2>/dev/null)" \
    && echo "$vat_out" > "$TMP_UPPER_CLIENTS"
fi
```

---

## Informacion del paquete

- Nombre: `versatile-autoreg-vat`
- Version: `0.3-4`
- Arquitectura: `all`
- Mantenedor: Gabriel Navia <correos@gabrielnav.es>
- Licencia: Apache 2.0
- Repositorio: https://gitlab.vitalinux.educa.aragon.es/gabriel/versatile-autoreg-vat

---

## Construccion

```bash
dpkg-buildpackage -us -uc -b
```

---

## Referencia

- `man vat-operate`
- `/usr/share/vat/presets/example.yaml`

---

## Licencia

[Apache License 2.0](LICENSE)
