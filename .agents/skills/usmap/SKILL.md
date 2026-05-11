---
name: usmap
description: Parse and debug .usmap files (Unreal Engine type mapping format). Use when reading, writing, or troubleshooting usmap binary parsing — including name maps, enum maps, schema maps, compression, and version handling (v0–v4).
metadata:
  project: unrealmodding
  language: rust
---

# Usmap Parsing Skill

Parse and debug `.usmap` files (Unreal Engine type mapping format).

## When to use

- User needs to parse, read, or write `.usmap` files
- Debugging usmap parsing failures
- Adding support for new usmap versions
- Understanding the usmap binary format

## Quick reference

The usmap format is a compressed binary file containing:
1. **Name map**: lookup table for string names (indexed by i32)
2. **Enum map**: enum definitions with value names
3. **Schema map**: struct/property definitions

### Format overview

```
Header: magic(u16 LE=0x30C4) + version(u8) + has_versioning(i32 bool) + compression(u8) + sizes(u32+u32) + data
Data: name_map + enum_map + schema_map + [extensions]
```

### Current codebase locations

| What | Where |
|------|-------|
| Main parsing | `unreal_asset/unreal_asset_base/src/unversioned/mod.rs` → `Usmap::new()` / `parse_data()` |
| Name map reader | `unreal_asset/unreal_asset_base/src/unversioned/usmap_reader.rs` → `UsmapReader::read_name()` |
| Property types | `unreal_asset/unreal_asset_base/src/unversioned/properties/mod.rs` → `EPropertyType` |
| Enum property | `unreal_asset/unreal_asset_base/src/unversioned/properties/enum_property.rs` |

### Version history

| Version | Key change |
|---------|------------|
| 0 (Initial) | Base format |
| 1 (PackageVersioning) | i32 has_versioning flag |
| 2 (LongFName) | u16 name lengths (was u8) |
| 3 (LargeEnums) | u16 enum value counts (was u8) |
| 4 (ExplicitEnumValues) | Enum values are (u64, name) pairs |

### Critical implementation details

1. **Magic**: `0x30C4` little-endian. File bytes are `C4 30`.
2. **has_versioning is i32**: CUE4Parse's `ReadBoolean()` reads 4 bytes (`int`), NOT 1 byte. The Rust `read_bool()` reads 1 byte — do NOT use it for the usmap header.
3. **Name index -1 is valid**: Means null/empty, not an error. Return empty string.
4. **Extension section is optional**: Some tools (e.g. mapping generators) add non-standard data after schemas. Extension parsing should be best-effort.
5. **Property data is recursive**: StructProperty reads a name, ArrayProperty/SetProperty/OptionalProperty read an inner type recursively, MapProperty reads key+value types recursively, EnumProperty reads inner type + name.

### Reference implementations

- **CUE4Parse** (C#, authoritative): `CUE4Parse/MappingsProvider/Usmap/`
  - `EUsmapVersion.cs` — version enum
  - `UsmapParser.cs` — main parser
  - `FUsmapReader.cs` — reader wrapper
  - `UsmapProperties.cs` — property type parsing
  - `EPropertyType.cs` — property type enum
  - `EUsmapCompressionMethod.cs` — compression methods

### Debugging

Standalone parser (no Rust compilation needed):
```bash
node usmap_debug/parse_usmap.js path/to/file.usmap
```

Test `Usmap::new()` directly:
```bash
cargo run -p usmap_test -- path/to/file.usmap
```

### Common issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| "File is not a valid usmap file" | Magic check fails | Ensure magic is `0x30C4` not `0xC430` |
| All fields after version are garbage | has_versioning read as 1 byte | Read i32 (4 bytes) for has_versioning |
| Enum parsing fails | Missing LargeEnums/ExplicitEnumValues support | Add version checks for u16 counts and (u64,name) pairs |
| "Invalid extension version" | Non-standard data after schemas | Make extension parsing best-effort |
| Name entries are garbage | LongFName not supported | Use u16 name lengths for version >= 2 |
