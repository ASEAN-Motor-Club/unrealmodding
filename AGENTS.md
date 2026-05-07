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

## Asset Parsing (Key Files)

| File | Purpose |
|------|---------|
| `unreal_asset/src/asset.rs` | `Asset` struct, `Asset::new()` — main entry point |
| `unreal_asset/src/asset_data.rs` | `AssetData`, `read_export()`, `read_export_no_raw()` |
| `unreal_asset/unreal_asset_exports/src/normal_export.rs` | `NormalExport::from_base()` — reads properties + ObjectGuid |
| `unreal_asset/unreal_asset_exports/src/data_table_export.rs` | `DataTableExport::from_base()` — reads DataTable rows |
| `unreal_asset/unreal_asset_properties/src/lib.rs` | `Property::new()` — unversioned property dispatch |
| `unreal_asset/unreal_asset_properties/src/struct_property.rs` | `StructProperty::custom_header()` — nested struct reading |
| `unreal_asset/unreal_asset_properties/src/enum_property.rs` | `EnumProperty::new()` — enum value reading |
| `unreal_asset/unreal_asset_properties/src/int_property.rs` | `BoolProperty`, `IntProperty`, `FloatProperty`, etc. |
| `unreal_asset/unreal_asset_properties/src/str_property.rs` | `TextProperty`, `StrProperty`, `NameProperty` |
| `unreal_asset/unreal_asset_base/src/unversioned/header.rs` | `UnversionedHeader`, `UnversionedHeaderFragment` |
| `unreal_asset/unreal_asset_base/src/engine_version.rs` | `EngineVersion` enum (includes VER_UE5_3/4/5) |

## USMAP Parsing (Key Files)

| File | Purpose |
|------|---------|
| `unreal_asset/unreal_asset_base/src/unversioned/mod.rs` | `Usmap` struct, `Usmap::new()`, `parse_data()` |
| `unreal_asset/unreal_asset_base/src/unversioned/usmap_reader.rs` | `UsmapReader` — wraps archive reader with name map lookup |
| `unreal_asset/unreal_asset_base/src/unversioned/properties/mod.rs` | `EPropertyType` enum, `UsmapPropertyData` dispatch |

## USMAP Format (v4 / ExplicitEnumValues)

Reference: [CUE4Parse UsmapParser.cs](https://github.com/FabianFG/CUE4Parse/tree/master/CUE4Parse/MappingsProvider/Usmap)

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
    [name_bytes]  // length does NOT include null; no null terminator stored

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
      [type-specific data]
```

### EPropertyType Type-Specific Data

- **StructProperty**: i32 struct_name
- **ArrayProperty/SetProperty/OptionalProperty**: recursive inner type
- **EnumProperty**: recursive inner type + i32 enum_name
- **MapProperty**: recursive key type + recursive value type
- All others (Byte, Bool, Int, Float, etc.): no extra data

### Usmap Schema Property Key

Properties are stored in `IndexedMap` keyed by `(name: String, array_index: u32)`. The `array_index` is 0 for non-array properties, 0..N for array properties. This matches CUE4Parse's `PropertyInfo.Index`.

## Usmap Version History

| Value | Name | Changes |
|-------|------|---------|
| 0 | Initial | Base format |
| 1 | PackageVersioning | Adds i32 has_versioning flag + version data |
| 2 | LongFName | Name lengths become u16 instead of u8 |
| 3 | LargeEnums | Enum value counts become u16 instead of u8 |
| 4 | ExplicitEnumValues | Enum values are (u64, name) pairs |

## UE5.4+ Asset Parsing

### NormalExport Layout (UE5.4+)

Reference: [UAssetAPI NormalExport.cs](https://github.com/atenfyr/UAssetAPI/blob/master/UAssetAPI/ExportTypes/NormalExport.cs)

```
[leading 4 null bytes check — only for DATA_RESOURCES < ObjectVersionUE5 < ASSETREGISTRY_PACKAGEBUILDDEPENDENCIES, not CDO]:
  i32   leading_check  // if non-zero, seek back (no-op); if zero, consume 4 bytes

[unversioned properties]:
  UnversionedHeader  // fragments + zero mask
  property data...

[if not CDO]:
  i32   has_object_guid  // 0 = no GUID, non-zero = has GUID
  if has_object_guid != 0:
    16 bytes  GUID
```

Key differences from CUE4Parse:
- **ObjectGuid presence is i32 (4 bytes)**, not `ReadBoolean()` (1 byte). UAssetAPI writes `writer.Write((int)0)` for no GUID.
- **Leading 4 null bytes check**: Range-guarded: `ObjectVersionUE5::DATA_RESOURCES < version < ObjectVersionUE5::ASSETREGISTRY_PACKAGEBUILDDEPENDENCIES`. Reads 4 bytes; if zero, consumes them; if non-zero, seeks back.

### UnversionedHeader

Reference: [CUE4Parse FUnversionedHeader.cs](https://github.com/FabianFG/CUE4Parse/blob/master/CUE4Parse/UE4/Assets/Objects/Unversioned/FUnversionedHeader.cs)

```
Fragment word (u16 LE):
  bits 0-6:   skip_num (7 bits)
  bit 7:      has_zeros
  bit 8:      is_last
  bits 9-15:  value_num (7 bits)

Fragment reading loop:
  first_num = 0
  loop:
    read fragment word
    fragment.first_num = first_num + skip_num
    first_num = fragment.first_num + value_num
    if has_zeros: zero_mask_num += value_num
    if is_last: break

Zero mask loading (CUE4Parse LoadZeroMaskData):
  if num_bits <= 8:  read 1 byte
  elif num_bits <= 16: read 2 bytes
  else: read ceil(num_bits/32) * 4 bytes

  Then: BitVec::from_vec(data).truncate(num_bits)
```

`first_num` is `u16` (not `u8`) — matches CUE4Parse's `ushort FirstNum`.

### Zero Mask Semantics

CUE4Parse: `HasNonZero(i) = HasNonZeroValues && !ZeroMask[i]`

- `ZeroMask[i] = true (1)` → property IS zero → skip reading
- `ZeroMask[i] = false (0)` → property is non-zero → read data

### Property Reading (Unversioned)

Reference: [CUE4Parse UObject.cs DeserializePropertiesUnversioned](https://github.com/FabianFG/CUE4Parse/blob/master/CUE4Parse/UE4/Assets/Exports/UObject.cs)

For unversioned properties, `Property::new`:
1. Walks fragments to find current property index
2. Skips fragments with `value_num == 0`
3. Returns `Ok(None)` when past all fragments
4. Looks up schema by parent name, walks up inheritance chain
5. Returns `Ok(None)` if `super_type` is empty (dead end)
6. Reads zero mask bit for current fragment
7. Calls `Property::from_type` with `effective_include_header = false`

`effective_include_header = false` for unversioned — no property GUID is read from the stream.

### BoolProperty (Unversioned)

BoolProperty ALWAYS reads 1 byte from the stream, even for unversioned properties. The `native_bool` from the usmap is NOT used to skip reading. CUE4Parse: `tagData?.Bool ?? Ar.ReadBoolean()` — if `tagData.Bool` is null (which it is for usmap-sourced properties), reads 1 byte.

### EnumProperty (Unversioned)

Reference: [CUE4Parse FPropertyTagType.cs FEnumPropertyTag](https://github.com/FabianFG/CUE4Parse/blob/master/CUE4Parse/UE4/Assets/Objects/Properties/EnumProperty.cs)

For unversioned EnumProperty, the serialization depends on the **inner type** from the usmap:

| Inner Type | Bytes Read | Notes |
|-----------|-----------|-------|
| ByteProperty | 1 (u8) | Most common |
| UInt16Property | 2 (u16) | |
| UInt32Property | 4 (u32) | |
| UInt64Property | 8 (u64) | |
| Int8Property | 1 (i8) | |
| Int16Property | 2 (i16) | |
| IntProperty | 4 (i32) | |
| Int64Property | 8 (i64) | |
| Other (NameProperty etc.) | 8 (FName) | Enum class values |

**Critical**: When EnumProperty is inside an ArrayProperty/MapProperty/SetProperty, the direct `get_property(name, ancestry)` lookup finds the container, not the inner EnumProperty. The code must extract the `UsmapEnumPropertyData` from the container's `inner_type`/`value_type`.

### StructProperty (Unversioned)

For unversioned StructProperty:
1. `include_header = false` → no struct_type read from stream
2. struct_type resolved from usmap via `get_property(name, ancestry)`
3. If struct_type is known, use `ancestry.with_parent(struct_type)` for usmap lookup
4. `custom_serialization` check for known types (RichCurveKey, GameplayTagContainer, etc.)
5. If not custom: read `UnversionedHeader` + properties in loop

### DataTable Export

Reference: [UAssetAPI DataTableExport.cs](https://github.com/atenfyr/UAssetAPI/blob/master/UAssetAPI/ExportTypes/DataTableExport.cs)

```
NormalExport::from_base()     // reads DataTable properties (RowStruct, etc.)
i32  num_entries              // ONE i32, not two (no "skip" field)
for each entry:
  FName  row_name
  StructProperty::custom_header(struct_type=RowStruct, ancestry=DataTable)
```

The `ancestry` for row structs is `Ancestry::new(base.get_class_type_for_ancestry(asset))` — just `["DataTable"]`. The `custom_header` internally extends ancestry to `["DataTable", "VehiclePartRow"]` for property lookup.

### Key Gotchas

1. **first_num is u16, not u8**: CUE4Parse uses `ushort`. Schemas with >255 total inherited properties overflow u8.
2. **Zero mask size**: CUE4Parse uses `ceil(num_bits/8)` for small masks (≤16 bits) and `ceil(num_bits/32)*4` for large masks (>16 bits). NOT always `ceil(num_bits/8)`.
3. **ObjectGuid is i32, not bool**: UE5.4+ NormalExport writes ObjectGuid presence as `i32` (4 bytes), not `bool` (1 byte).
4. **No "skip" field in DataTable**: CUE4Parse and UAssetAPI read only ONE `i32` for num_entries after NormalExport.
5. **EnumProperty inner type matters**: UInt16Property inner reads 2 bytes, not FName (8 bytes). Always check inner type.
6. **Unversioned properties have no GUID**: `effective_include_header = false` — `optional_guid!` reads nothing.
7. **Schema chain dead end**: If `super_type` is empty and property index exceeds `prop_count`, return `Ok(None)`.
8. **ArrayProperty struct item length**: For unversioned properties, `length = 1` is hardcoded. But `ArrayProperty::new_no_header` calculates `size_est_1 = length / num_entries`, which becomes 0 for arrays with >1 entry. StructProperty with `length = 0` returns empty without reading. Fix: use `length = 1` for struct items when `has_unversioned_properties()`.
9. **MapProperty keys_to_remove**: The Vec must be created OUTSIDE the loop, not inside. Otherwise each iteration overwrites with a single-element Vec.

## Debugging

### Usmap debugging

```bash
node usmap_debug/parse_usmap.js path/to/file.usmap
```

### Asset debugging

The `usmap_test/` crate can be used to test asset parsing:

```bash
cargo build -p usmap_test
./target/debug/usmap_test path/to/file.uasset path/to/file.usmap [path/to/file.uexp]
```

UAssetGUI can be used as a .NET CLI tool for cross-referencing:

```bash
export DOTNET_ROOT="/nix/store/16341mr27xkzix0a048jwj9lrzq5bk1c-dotnet-sdk-8.0.419"
export PATH="$DOTNET_ROOT/bin:$PATH"
cd /tmp/uasset_dumper && dotnet run -- path/to/file.uasset path/to/file.usmap
```

### Key reference files

- **CUE4Parse** (C#): Authoritative reference for all parsing logic
  - `CUE4Parse/UE4/Assets/Objects/Unversioned/` — unversioned header, iterator, fragments
  - `CUE4Parse/UE4/Assets/Exports/UObject.cs` — property deserialization
  - `CUE4Parse/UE4/Assets/Objects/Properties/` — all property types
  - `CUE4Parse/UE4/Assets/Objects/FScriptStruct.cs` — struct type dispatch
  - `CUE4Parse/UE4/Assets/Exports/Engine/UDataTable.cs` — DataTable reading
- **UAssetGUI** (C#): Alternative reference
  - `UAssetAPI/ExportTypes/NormalExport.cs` — NormalExport with UE5.4+ logic
  - `UAssetAPI/ExportTypes/DataTableExport.cs` — DataTable export
  - `UAssetAPI/PropertyTypes/Structs/StructPropertyData.cs` — StructProperty reading

## Building

```bash
# Check compilation (no OpenSSL needed)
cargo check -p unreal_asset_base -p unreal_asset_properties -p unreal_asset_exports -p unreal_asset

# Full build (requires OpenSSL for reqwest in other crates)
cargo build
```
