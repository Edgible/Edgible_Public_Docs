# Kbase corpus moved

Edgible’s RAG kbase markdown now lives in the dedicated **`Edgible_Kbase`** repository (sibling to `Edgible_Public_Docs` in the monorepo).

- **References:** `Edgible_Kbase/references/` (e.g. `references/awesome-selfhosted/`)
- **Index:** `Edgible_Kbase/INDEX.md`

Update Progento bind mounts to point at **`Edgible_Kbase`** instead of this folder, then rescan.

This directory is kept only as a **pointer** to avoid silent breakage for old paths; do not add new kbase content here.
