---
name: optilogic-manage-model-storage
description: Work with Optilogic SQL storage devices - connection strings, schema, custom columns and custom tables - without leaving a model database in an inconsistent state.
api: Optilogic REST API
base: https://api.optilogic.app/v0
operations:
  - GET /storages/db-connection-strings
  - GET /storage/{storageName}/connection-string
  - GET /storage/{storageName}/schema
  - GET /storage/{storageName}/db-objects
  - GET /storage/{storageName}/tables
  - GET /storage/{storageName}/custom-columns
  - POST /storage/{storageName}/custom-column
  - PUT /storage/{storageName}/custom-column
  - DELETE /storage/{storageName}/custom-column
  - GET /storage/{storageName}/custom-columns/validate
  - POST /storage/{storageName}/custom-columns/repair
  - GET /storages/db-column-types
  - POST /storage/{storageName}/empty-tables
  - POST /storage/{storageName}/clone
generated: '2026-08-26'
method: generated
source: openapi/optilogic-rest-api-openapi.json
---

# Manage an Optilogic SQL storage device

A **storage device** is a Cosmic Frog model database (or another Postgres database) attached to the account.
`storageName` is the path parameter on 34 of the 67 operations in this API — it is the busiest identifier in
the contract.

## Read before you write

1. `GET /storages/db-connection-strings` — every SQL storage item the account can reach.
   `GET /storage/{storageName}/connection-string` for one.
2. `GET /storage/{storageName}/schema` — tables and views.
   `GET /storage/{storageName}/db-objects` — stats, tables and views; filterable by `schema`, `type`, `tags`.
3. `GET /storage/{storageName}/tables` — table-level detail.
4. `GET /storages/db-column-types` — the **supported column data types**. Read this before proposing a
   `dataType`; do not guess a Postgres type name.

## Adding a custom column

1. `POST /storage/{storageName}/custom-column` with `table`, `columnName`, `dataType`, and where relevant
   `characterMaximumLength`, `isNullable`, `defaultValue`, `isTableKeyColumn`, `isRequired`.
2. `GET /storage/{storageName}/custom-columns/validate` afterwards. It returns typed issues, and the
   published issue vocabulary is exact: `MissingTable`, `MissingColumn`, `ExtraColumn`, `TypeMismatch`,
   `MissingMetadata`, `MetadataDiscrepancy`, `IsNullableMismatch`, `UpdateTypedView`, `MissingTypedView`,
   `DefaultValueMismatch`, `MaxLengthMismatch`, `TableWithNoColumns`, `UnspecifiedError`.
3. If validate reports issues, `POST /storage/{storageName}/custom-columns/repair`. This is the documented
   path back to a consistent schema and the closest thing this API has to a rollback.
4. `412 Precondition Failed` on the custom-column surface means the table state does not satisfy the
   operation precondition — validate, repair, then retry.

## Destructive operations — read this before calling either

- **`POST /storage/{storageName}/empty-tables` deletes all data rows from the named tables. There is no undo
  and no retention window is published.** The only published safety net is
  `POST /storage/{storageName}/clone`, which duplicates the storage. Clone first.
- **`POST /storage/{storageName}/reassign` transfers ownership to another user and has no inverse operation.**
  Once reassigned, only the new owner can reassign it back. Never call this on behalf of a user without an
  explicit, specific instruction naming the target user.

## Sharing

- `POST /storage/{storageName}/share/access` grants access; `DELETE /storage/{storageName}/share/access`
  revokes it. This pair **is** reversible.
- `POST /storage/{storageName}/share` sends a **copy** to another user — that is a different act from
  granting access and it cannot be recalled.

## Exporting

`POST /storage/{storageName}/db-data-export` exports rows from selected tables/views (or the results of a
query) and returns **202** with a job key. Track it with `GET /job/db-data-export/{jobKey}`, and list all
export jobs with `GET /job/db-data-export/list`.
