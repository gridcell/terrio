# tiff 0.11.3, patched: lazy chunk tables

A copy of `tiff` 0.11.3 from crates.io, wired up by `[patch.crates-io]` in the workspace
`Cargo.toml`. Remove this directory and the patch entry when upstream has the change.

## Problem

`Image::from_reader` reads the whole `TileOffsets` and `TileByteCounts` (or `StripOffsets` and
`StripByteCounts`) arrays into `Vec<u64>`s. It runs on `Decoder::new` and on every
`seek_to_image`. A Cloud Optimized GeoTIFF with a million tiles has 16 MB of tables: opening it,
or moving between its overviews, reads and decodes all of it, and a remote file fetches it over
HTTP. GDAL (libtiff's deferred strile loading) reads only the entries of the chunks it decodes.

## Change

`src/decoder/image.rs` and `src/decoder/mod.rs`.

- `Image` gains `chunk_count` and `chunk_entries: Option<(Entry, Entry)>`. For an image with at
  least `LAZY_CHUNK_TABLES_FROM` (65,536) chunks whose entries are `SHORT`, `LONG` or `LONG8`,
  `from_reader` keeps the two entries and leaves `chunk_offsets` and `chunk_bytes` empty. Smaller
  images read their tables whole, as before.
- `Image::chunk_file_range` takes the reader and, for a lazy image, reads the chunk's offset and
  byte count (one small read each) through `Entry::raw_value_at`.
- `Image::expand_chunk` looks the chunk up itself and seeks to it, so a lazy lookup cannot leave
  the reader in the wrong place.
- `Decoder::tile_count` and the readout layout use `chunk_count` instead of the table length.

The entry count is checked against the expected chunk count for both lazy and whole tables, as
the table lengths were before.

## Not changed

The `tiff` crate's public API. Callers that read the tables themselves (`get_tag_u64_vec`) still
get the whole array; terrio's `ChunkTables` reads them by block instead.
