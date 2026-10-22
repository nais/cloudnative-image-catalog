# PostGIS and TimescaleDB extension rollout

This procedure covers verification of the catalog changes before merging and
enabling application use. It does not authorize a deployment or changes to
existing PostgreSQL clusters.

## 1. Start with a draft PR

- [x] Create the `feat/postgis-timescaledb-extensions` branch.
- [x] Commit the generator, generated catalog, tests, CI and documentation.
- [x] Push the branch and create a draft PR against `main`.
- [ ] Confirm the PR checks pass.
- [ ] Keep the PR in draft until the isolated runtime tests below are complete.

The generated catalog retains **standard-trixie**, but also updates PostgreSQL
from 18.4 to 18.6. Merging into `main` triggers the existing deployment workflow;
updating the shared catalog can roll existing clusters. Do not merge just to
make the extension images available for testing.

## 2. Confirm the target environment

The following evidence was provided for the inspected dev cluster on 2026-10-22:

| Check | Observed result |
| --- | --- |
| CNPG operator Pod image | `ghcr.io/cloudnative-pg/cloudnative-pg:1.30.0` |
| ClusterImageCatalog CRD | Extension `env` fields and placeholder support present |
| Cluster CRD | Extension image, paths and environment fields present |
| Kubernetes API server | `v1.36.4-gke.2046003` |
| Inspected PostgreSQL node kubelet | `v1.36.4-gke.2046003` |
| Inspected node runtime | `containerd://2.2.7` |
| Inspected node architecture | `amd64` |

These observations meet the version prerequisites for catalog-based extensions:
CNPG 1.29+, PostgreSQL 18 and an ImageVolume-capable Kubernetes/runtime setup.
ImageVolume is stable from Kubernetes 1.36; it is enabled by default in 1.35,
and must be explicitly enabled in 1.33/1.34.

This is evidence for the inspected environment and node, not every deployment
target or node pool. Repeat the checks wherever the catalog will be deployed.
Runtime mounting and registry access are not established by version checks.

## 3. Deploy an isolated smoke test

Obtain approval for the test deployment and select an explicit Kubernetes
context and a dedicated test namespace. Do not use an existing application
database cluster.

1. Render the chart from the PR with a unique catalog name, such as
   `extension-smoke-catalog`. Confirm the rendered resource name and digest-pinned
   images. Apply only the test catalog, never the existing shared catalog.
2. Create a disposable CNPG PostgreSQL 18 cluster referencing the test catalog.
   Retain `standard-trixie` and use the relevant PostgreSQL node pool,
   scheduling constraints and security settings.
3. Select `postgis` and `timescaledb-oss` by name under
   `spec.postgresql.extensions`. Do not specify extension image references in
   the Cluster: this test must exercise catalog resolution.
4. Wait for the cluster to become healthy. Inspect
   `status.pgDataImageInfo.extensions` for the resolved digests and PostGIS
   `ld_library_path`, `GDAL_DATA` and `PROJ_DATA` settings.
5. Inspect Pod image volumes and mounts. Check events for image pull or mount
   failures. Use `pullPolicy: Always` for the test extension image volumes to
   exercise registry resolution, and confirm the volumes reference the expected
   digests. A laptop image pull does not prove node registry access.
6. After the extension images have rolled out, add `timescaledb` to
   `spec.postgresql.shared_preload_libraries`, preserving other entries. Wait
   for the configuration rollout to finish before installing SQL extensions.
7. Install `postgis` and `timescaledb` in the test database using an appropriately
   privileged connection or CNPG Database extension management.

Available images are not installed SQL extensions. The catalog identifier is
`timescaledb-oss`; the preload library and SQL extension are `timescaledb`.

Run the following in the disposable test database:

```sql
CREATE EXTENSION postgis;
CREATE EXTENSION timescaledb;

SELECT PostGIS_Full_Version();
SELECT ST_AsText(
  ST_Transform(ST_SetSRID(ST_MakePoint(10, 60), 4326), 3857)
);

SHOW shared_preload_libraries;
SELECT extversion FROM pg_extension WHERE extname = 'timescaledb';

CREATE TABLE measurements (time timestamptz NOT NULL, value integer);
SELECT create_hypertable('measurements', 'time');
INSERT INTO measurements VALUES (now(), 42);
SELECT * FROM measurements;
```

If Database resources already installed the extensions, omit the two
`CREATE EXTENSION` statements. Configure the SQL client to stop on errors.

Also verify `SHOW extension_control_path`, `SHOW dynamic_library_path`, and the
actual PostgreSQL process environment. PostGIS should resolve its library path
to `/extensions/postgis/system`, GDAL data to `/extensions/postgis/share/gdal`
and PROJ data to `/extensions/postgis/share/proj`; confirm these paths exist.
Check GDAL data access separately with an applicable raster/GDAL operation:
the coordinate transformation above primarily exercises PROJ.

Repeat the mount and pull tests for each relevant node pool, runtime and
architecture. The TimescaleDB image includes only the Apache 2.0 edition;
do not assume features requiring the Timescale License are available.

## 4. Record results before merge

- [ ] Record the context, namespace, test resource names and test date in the PR.
- [ ] Record operator version, CRD release, node pool, kubelet, runtime and architecture.
- [ ] Record the actual PostgreSQL and extension image digests.
- [ ] Attach catalog-resolution, mount, environment and SQL test results.
- [ ] Review compatibility across all deployment targets, not only the inspected dev cluster.
- [ ] Explicitly assess the PostgreSQL 18.4 to 18.6 update and resulting rollout impact.
- [ ] Remove the disposable test resources and test catalog after testing.
- [ ] Check PVC ownership and contents before deleting any test PVCs.
- [ ] Mark the PR ready for review and merge only after the required checks pass.

Local generation, upstream-equality, failure-case, Helm lint/render/package and
regeneration-stability checks have passed. These checks do not replace the
runtime tests above. No runtime smoke test has yet been recorded in this guide.

## 5. Follow up in pgrator and the platform

Keep these changes outside the catalog PR:

- Map application requests to catalog identifiers, including `timescaledb-oss`
  versus the SQL name `timescaledb`.
- Select extensions by name and inherit their complete catalog settings.
- Handle TimescaleDB preload and SQL installation in the correct rollout order.
- Manage SQL extension versions and upgrade paths separately from image updates.
- Test restart, failover, backup/restore and extension upgrades before production use.
- Record and address any missing platform capabilities in other target environments.

Updating an extension image can roll PostgreSQL Pods, but does not execute
`ALTER EXTENSION ... UPDATE` in application databases.