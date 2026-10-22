# cloudnative-image-catalog

This repo defines workflows to fetch and deploy an updated imagecatalog for PostgreSQL for use with cloudnative-pg.

`mise/config.toml` defines the PostgreSQL versions, image type and Debian release.
The catalog uses PostgreSQL 18 **standard-trixie**, not minimal-trixie.

## Generation and validation

With Bash 4+, mise, yq v4 and Helm installed:

```sh
mise run clone-artifacts
mise run update-template
mise run test
```

The generator reads the PostgreSQL image from upstream's
[`image-catalogs/catalog-standard-trixie.yaml`](https://github.com/cloudnative-pg/artifacts/blob/main/image-catalogs/catalog-standard-trixie.yaml)
and copies the complete `postgis` and `timescaledb-oss` definitions for the same
major from
[`image-catalogs-extensions/catalog-minimal-trixie.yaml`](https://github.com/cloudnative-pg/artifacts/blob/main/image-catalogs-extensions/catalog-minimal-trixie.yaml).
The latter is only the source of extension definitions: its minimal PostgreSQL
image is not used. Versions, digests and extension settings remain upstream-owned.
Generation fails on missing/duplicate definitions, unsupported majors, mismatched
major/distribution tags or images without SHA256 digests, without replacing the template.

Tests compare the rendered catalog with upstream, check PostGIS settings and
digest pinning, exercise failure cases, run Helm lint/rendering and verify that
a second generation is byte-identical. CI runs these checks before packaging.

## Compatibility and prerequisites

- **CloudNativePG 1.29 or later**, including matching CRDs, is required for
  selecting extension images from an image catalog. ImageVolume support in CNPG
  1.27/1.28 alone is not sufficient for catalog-based selection.
- PostgreSQL 18 provides `extension_control_path`. All three images must match
  the PostgreSQL major version, Debian distribution and node architecture.
- Checked against GHCR on 2026-10-22: PostgreSQL 18.6 standard-trixie, PostGIS
  3.6.4 for PG18/Trixie and TimescaleDB OSS 2.30.2 for PG18/Trixie all publish
  `linux/amd64` and `linux/arm64` manifests. Extension labels confirm major 18
  and Trixie. Upstream's standard image extends its minimal image, so the
  extension catalog's minimal origin does not require switching image variant.
  This verifies metadata compatibility, not runtime loading; future upstream
  updates require the same checks.
- Kubernetes must support image volumes: 1.35+, or 1.33/1.34 with the
  `ImageVolume` feature gate enabled, according to the linked CNPG documentation.
  A compatible runtime is also required, such as containerd 2.1+ or CRI-O 1.31+.
- Nodes must be able to pull the pinned images from GHCR. Operator/CRD versions,
  feature gates, runtime support and registry access have **not** been verified
  in our running environments.

See upstream's [catalog compatibility](https://github.com/cloudnative-pg/cloudnative-pg/blob/main/docs/src/image_catalog.md)
and [ImageVolume requirements](https://cloudnative-pg.io/docs/1.28/imagevolume_extensions/).

## Using the catalog entries

Publishing an extension image does **not** mount it in every cluster or install
an SQL extension. A consumer such as pgrator must select the catalog entries in
the CNPG `Cluster`, then manage installation separately in each database.

For an operator supporting catalog extensions, the relevant Cluster fields are:

```yaml
spec:
  imageCatalogRef:
    apiGroup: postgresql.cnpg.io
    kind: ClusterImageCatalog
    name: cloudnative-image-catalog
    major: 18
  postgresql:
    extensions:
      - name: postgis
      - name: timescaledb-oss
```

Use the catalog name rendered by your Helm release if it differs from the
example. Select only the extensions needed by that cluster. CNPG resolves their
images and settings from the catalog; do not duplicate image references in
consumers. Preserve PostGIS' `ld_library_path: [system]` and
`GDAL_DATA=${image_root}/share/gdal` / `PROJ_DATA=${image_root}/share/proj`.
CNPG expands `${image_root}` to the extension's mount path.

For TimescaleDB, **after the extension image has rolled out**, add `timescaledb`
to `spec.postgresql.shared_preload_libraries`, preserving other preload entries:

```yaml
spec:
  postgresql:
    shared_preload_libraries:
      - timescaledb
```

These are successive configuration fragments, not complete manifests. Do not
replace existing `postgresql` configuration with the second fragment. Upstream
warns against changing extension images and preload configuration in the same
rollout. Test restarts, including changes to PostGIS library paths/environment,
which may require a manual restart.

Install SQL extensions in the target database using CNPG `Database` extension
management or an appropriately privileged migration:

```sql
CREATE EXTENSION postgis;
CREATE EXTENSION timescaledb;
```

The TimescaleDB catalog identifier is **`timescaledb-oss`**, but both the preload
library and SQL extension are **`timescaledb`**. This image contains only the
[Apache 2.0 edition](https://github.com/cloudnative-pg/postgres-extensions-containers/blob/main/timescaledb-oss/README.md):
features requiring the Timescale License are not included. Check required
application features against this edition rather than assuming full TimescaleDB
functionality.

## Follow-up outside this repository

- pgrator must map requested extensions to catalog identifiers, preserve catalog
  settings, configure TimescaleDB preload in the correct rollout order, and
  manage SQL installation/version upgrades in each database.
- The platform must verify that the installed CNPG operator and CRDs support
  catalog extensions, plus Kubernetes feature gates, runtime support and GHCR
  access on all relevant node pools.
- Test in a non-production cluster: mounts, resolved paths/environment, restarts,
  `CREATE EXTENSION`, `SELECT PostGIS_Full_Version()`, a PostGIS coordinate
  transformation, and TimescaleDB hypertable creation/inserts. Test failover,
  image updates, SQL extension upgrades and backup/restore before production use.

These runtime checks have not been performed by the local generation and Helm
tests. Updating catalog images can roll PostgreSQL pods; it does not perform
`ALTER EXTENSION ... UPDATE` in application databases.
