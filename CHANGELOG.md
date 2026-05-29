# Changelog

## [1.1.1] — 2026-05-29

### Fixed
- **plugin.json**: eliminado el BOM UTF-8 del manifest, que rompía la instalación del plugin vía marketplace (Claude Code hace `JSON.parse` directo y falla con BOM).

## [1.1.0] — 2026-05-29

### Added
- **M10 Mobile Vertical Expansion** — 8 skills del ciclo móvil + agente operativo `@mobile` con 5 contratos bilaterales. Skills clave: `/mobile-offline-sync` (offline-first), `/mobile-app-store-submit` (publicación App Store / Google Play), `/mobile-deep-linking` (deep links + universal links), sumados a `/mobile-scaffold` y `/mobile-deploy`. Knowledge: `mobile-production-patterns`, `mobile-saas-starter-spec`. 7 sprints SDD (CASTLE §8 > §2).

---

## [1.0.0] — release inicial
- `/mobile-scaffold`, `/mobile-deploy`.
