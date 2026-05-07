# Asset Parsing Skill

Parse `.uasset` / `.uexp` files using usmap type mappings (unversioned properties).

## When to use

- Parsing UE4/UE5 asset files with unversioned properties
- Debugging asset export reading failures
- Adding support for new property types or engine versions
- Understanding the unversioned property serialization format

## Quick reference

Unversioned assets use a **schema-driven** format: the usmap defines which properties exist and their types. The asset stream contains only property **values** (no names, types, or sizes). An `UnversionedHeader` (fragments + zero mask) indicates which properties are serialized and which are zero/default.

### Key code locations

| What | Where |
|------|-------|
| Entry point | `unreal_asset/src/asset.rs` → `Asset::new()` |
| Export reading | `unreal_asset/src/asset_data.rs` → `read_export()`, `read_export_no_raw()` |
| NormalExport | `unreal_asset/unreal_asset_exports/src/normal_export.rs` → `from_base()` |
| DataTableExport | `unreal_asset/unreal_asset_exports/src/data_table_export.rs` → `from_base()` |
| Property dispatch | `unreal_asset/unreal_asset_properties/src/lib.rs` → `Property::new()` |
| StructProperty | `unreal_asset/unreal_asset_properties/src/struct_property.rs` → `custom_header()` |
| EnumProperty | `unreal_asset/unreal_asset_properties/src/enum_property.rs` → `new()` |
| UnversionedHeader | `unreal_asset/unreal_asset_base/src/unversioned/header.rs` |
| Engine versions | `unreal_asset/unreal_asset_base/src/engine_version.rs` |

### NormalExport flow (UE5.4+)

```
1. Leading 4 null bytes check (UE5.4+, reads i32, seeks back if non-zero)
2. UnversionedHeader: fragments + zero mask
3. Property loop: Property::new() until None
4. ObjectGuid: i32 presence (NOT bool), optional 16-byte GUID
```

### DataTable flow

```
1. NormalExport (RowStruct, bStripFromClientBuilds, etc.)
2. i32 num_entries (single i32, no "skip" field)
3. For each row:
   a. FName row_name
   b. StructProperty::custom_header(struct_type=RowStruct)
      - Reads UnversionedHeader for the row struct
      - Reads properties based on schema + zero mask
```

### UnversionedHeader format

```
Fragment (u16 LE):
  bits 0-6:  skip_num
  bit 7:     has_zeros
  bit 8:     is_last
  bits 9-15: value_num

Zero mask (after all fragments):
  ≤8 bits:  1 byte
  ≤16 bits: 2 bytes
  >16 bits: ceil(bits/32) * 4 bytes

Semantics: bit=1 → property is zero (skip), bit=0 → non-zero (read)
```

### Property reading rules

| Type | Unversioned bytes | Notes |
|------|------------------|-------|
| BoolProperty | 1 (u8) | Always reads, native_bool from usmap NOT used |
| IntProperty | 4 (i32) | |
| FloatProperty | 4 (f32) | |
| ObjectProperty | 4 (PackageIndex) | |
| NameProperty | 8 (FName) | |
| StrProperty | FString (variable) | |
| TextProperty | flags(4) + history_type(1) + conditional | |
| EnumProperty | 1-8 (depends on inner type) | ByteProperty→1, UInt16→2, UInt32→4, etc. |
| StructProperty | UnversionedHeader + inner properties | |
| ArrayProperty | i32 count + elements | |
| MapProperty | i32 count + key/value pairs | |

### Critical implementation details

1. **`first_num` is u16**: CUE4Parse uses `ushort`. Using u8 overflows for schemas with >255 inherited properties.
2. **`include_header = false`** for unversioned: No property GUID is read. `effective_include_header` must be set to `false` in `Property::new` when `has_unversioned_properties()`.
3. **ObjectGuid is i32**: UE5.4+ NormalExport writes `i32` (4 bytes) for ObjectGuid presence, not `bool` (1 byte). CUE4Parse's `ReadBoolean()` reads 1 byte — this is a format difference.
4. **EnumProperty inner type**: Must handle all numeric types (u8, u16, u32, u64, i8, i16, i32, i64), not just ByteProperty. For container elements (Array/Map/Set), extract the EnumProperty data from the container's inner type.
5. **Schema chain walk**: When `prop_count` is exceeded, walk up `super_type`. Return `Ok(None)` if `super_type` is empty.
6. **Fragment val=0**: Skip fragments with `value_num == 0` (they're padding/no-op).
7. **Zero mask semantics**: `bit=1` → zero (skip), `bit=0` → non-zero (read). Matches CUE4Parse `!ZeroMask[i]`.

### StructProperty custom serialization

Some struct types use custom serialization instead of the standard unversioned property loop:

```
RichCurveKey, GameplayTagContainer, Guid, Color, LinearColor,
Quat, Rotator, Vector, Vector2D, Vector4, Box, FloatRange,
PerPlatformBool, PerPlatformFloat, PerPlatformInt, etc.
```

See `CUSTOM_SERIALIZATION` array in `unreal_asset/unreal_asset_properties/src/lib.rs`.

### Debugging

```bash
# Build test binary
cargo build -p usmap_test

# Parse asset with usmap
./target/debug/usmap_test file.uasset file.usmap [file.uexp]
```

### Common issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| All exports are RawExport | Error in read_export_no_raw, silently caught | Add eprintln in the Err branch of read_export |
| TextHistoryType discriminant error | Stream misaligned — wrong byte count read | Check EnumProperty inner type, BoolProperty reading, ObjectGuid size |
| "No schema for X" | Schema not found in usmap | Check name map parsing (null terminator stripping) |
| "attempt to add with overflow" | first_num u8 overflow | Change first_num to u16 |
| DataTable 0 rows | Wrong number of i32 reads before rows | Only ONE i32 for num_entries (no "skip") |
| EnumProperty reads wrong bytes | Inner type not handled for UInt16/UInt32/etc. | Handle all numeric inner types |

### Reference implementations

- **CUE4Parse** (C#, authoritative):
  - `CUE4Parse/UE4/Assets/Exports/UObject.cs` — DeserializePropertiesUnversioned
  - `CUE4Parse/UE4/Assets/Objects/Unversioned/FUnversionedHeader.cs`
  - `CUE4Parse/UE4/Assets/Objects/Unversioned/FIterator.cs`
  - `CUE4Parse/UE4/Assets/Objects/Properties/` — all property types
  - `CUE4Parse/UE4/Assets/Exports/Engine/UDataTable.cs`
- **UAssetGUI** (C#):
  - `UAssetAPI/ExportTypes/NormalExport.cs` — UE5.4+ leading bytes + ObjectGuid
  - `UAssetAPI/ExportTypes/DataTableExport.cs`
  - `UAssetAPI/PropertyTypes/Structs/StructPropertyData.cs`
