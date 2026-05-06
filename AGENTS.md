# Unrealmodding Project

Rust monorepo for Unreal Engine modding tools. Primarily works with `.uasset`, `.usmap`, and `.pak` files.

## Project Structure

| Crate | Purpose |
|-------|---------|
| `unreal_asset/` | Main crate for reading/writing Unreal assets |
| `unreal_asset/unreal_asset_base/` | Core types, readers, unversioned/mappings support |
| `unreal_asset/unreal_asset_properties/` | Property type implementations |
| `unreal_asset/unreal_asset_exports/` | Export type implementations |
| `unreal_asset/unreal_asset_kismet/` | Kismet bytecode |
| `unreal_asset/unreal_asset_registry/` | Asset registry |
| `unreal_helpers/` | Shared utilities (read/write extensions, FString, GUID) |
| `unreal_pak/` | `.pak` file reading/writing |
| `unreal_pak_cli/` | CLI for pak files |
| `unreal_mod_integrator/` | Mod integration |
| `unreal_mod_manager/` | Mod management |
| `unreal_mod_metadata/` | Mod metadata |
| `unreal_cpp_bootstrapper/` | C++ bootstrapper |
| `dll_injector/` | DLL injection |

## USMAP Parsing (Key Files)

The `.usmap` format stores Unreal Engine type mappings (struct schemas, enums, property types).

| File | Purpose |
|------|---------|
| `unreal_asset/unreal_asset_base/src/unversioned/mod.rs` | `Usmap` struct, `Usmap::new()`, `parse_data()` — main parsing logic |
| `unreal_asset/unreal_asset_base/src/unversioned/usmap_reader.rs` | `UsmapReader` — wraps archive reader with name map lookup |
| `unreal_asset/unreal_asset_base/src/unversioned/properties/mod.rs` | `EPropertyType` enum, `UsmapPropertyData` dispatch |
| `unreal_asset/unreal_asset_base/src/unversioned/properties/enum_property.rs` | `UsmapEnumPropertyData` |
| `unreal_asset/unreal_asset_base/src/unversioned/properties/array_property.rs` | `UsmapArrayPropertyData` (also used for OptionalProperty) |
| `unreal_asset/unreal_asset_base/src/unversioned/properties/struct_property.rs` | `UsmapStructPropertyData` |
| `unreal_asset/unreal_asset_base/src/unversioned/properties/map_property.rs` | `UsmapMapPropertyData` |
| `unreal_asset/unreal_asset_base/src/unversioned/properties/set_property.rs` | `UsmapSetPropertyData` |
| `unreal_asset/unreal_asset_base/src/unversioned/properties/shallow_property.rs` | `UsmapShallowPropertyData` (catch-all) |

## USMAP Format (v4 / ExplicitEnumValues)

Reference implementation: [CUE4Parse UsmapParser.cs](https://github.com/FabianFG/CUE4Parse/tree/master/CUE4Parse/MappingsProvider/Usmap)

```
Header:
  u16   magic (0x30C4, little-endian)
  u8    version (0-4)
  i32   has_versioning (4-byte bool, NOT 1-byte!)
  if has_versioning:
    i32   object_version
    i32   object_version_ue5
    i32   num_custom_versions
    [16-byte GUID + i32 version] * num_custom_versions
    u32   net_cl
  u8    compression_method (0=None, 1=Oodle, 2=Brotli, 3=ZStandard)
  u32   compressed_size
  u32   decompressed_size
  [compressed data]

Decompressed data:
  u32   name_count
  for each name:
    u16   name_length  (version >= LongFName)
    or u8 name_length  (version < LongFName)
    [name_bytes]

  u32   enum_count
  for each enum:
    i32   enum_name (name index, -1 = null)
    u16   value_count (version >= LargeEnums)
    or u8 value_count (version < LargeEnums)
    for each value:
      if version >= ExplicitEnumValues:
        u64   explicit_value
        i32   value_name (name index)
      else:
        i32   value_name (name index)

  u32   schema_count
  for each schema:
    i32   name (name index)
    i32   super_type (name index, -1 = none)
    u16   property_count
    u16   serializable_property_count
    for each serializable property:
      u16   schema_index
      u8    array_size
      i32   property_name (name index)
      u8    property_type (EPropertyType)
      [type-specific data — see CUE4Parse ParsePropertyType()]
```

### EPropertyType Type-Specific Data

- **StructProperty**: i32 struct_name
- **ArrayProperty/SetProperty/OptionalProperty**: recursive inner type
- **EnumProperty**: recursive inner type + i32 enum_name
- **MapProperty**: recursive key type + recursive value type
- All others (Byte, Bool, Int, Float, etc.): no extra data

### Name Index Convention

- `i32 >= 0`: index into name_map
- `i32 == -1`: null/empty (valid, not an error)

## Usmap Version History (EUsmapVersion)

| Value | Name | Changes |
|-------|------|---------|
| 0 | Initial | Base format |
| 1 | PackageVersioning | Adds i32 has_versioning flag + version data |
| 2 | LongFName | Name lengths become u16 instead of u8 |
| 3 | LargeEnums | Enum value counts become u16 instead of u8 |
| 4 | ExplicitEnumValues | Enum values are (u64, name) pairs |

## Debugging Usmap Parsing

### Standalone debug parser

`usmap_debug/parse_usmap.js` — Node.js parser that mirrors the Rust code. Useful for quick validation without compiling Rust:

```bash
node usmap_debug/parse_usmap.js path/to/file.usmap
```

### Testing Usmap::new

`usmap_test/` — minimal Rust crate that calls `Usmap::new()` and prints results:

```bash
cargo run -p usmap_test -- path/to/file.usmap
```

### Key gotchas when debugging

1. **Magic is little-endian**: `0x30C4`, not `0xC430`. The bytes in the file are `C4 30`.
2. **has_versioning is 4 bytes (i32)**: CUE4Parse's `ReadBoolean()` reads `int`, not `byte`. This is NOT the same as `reader.read_bool()` which reads 1 byte.
3. **Extension section may be absent**: Some mapping tools add non-standard data after schemas (CEXT/PPTH). Extension parsing is best-effort.
4. **Oodle compression**: Requires the `oodle` feature flag. Without it, Oodle-compressed files return an error.

## Prior Work / References

- **CUE4Parse** (C#): Reference implementation for usmap parsing
  - [EUsmapVersion.cs](https://github.com/FabianFG/CUE4Parse/blob/master/CUE4Parse/MappingsProvider/Usmap/EUsmapVersion.cs)
  - [UsmapParser.cs](https://github.com/FabianFG/CUE4Parse/blob/master/CUE4Parse/MappingsProvider/Usmap/UsmapParser.cs)
  - [FUsmapReader.cs](https://github.com/FabianFG/CUE4Parse/blob/master/CUE4Parse/MappingsProvider/Usmap/FUsmapReader.cs)
  - [UsmapProperties.cs](https://github.com/FabianFG/CUE4Parse/blob/master/CUE4Parse/MappingsProvider/Usmap/UsmapProperties.cs)
- **UAssetGUI**: Another reference for usmap parsing
- **uesave-rs** (Rust): Related project for `.sav` files (not usmap, but same UE patterns)

## Building

```bash
# Check compilation (no OpenSSL needed)
cargo check -p unreal_asset_base

# Full build (requires OpenSSL for reqwest in other crates)
cargo build
```
