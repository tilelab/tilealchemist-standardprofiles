# TileAlchemist Standard Profiles

The `land` and `cropped-waterways` global PMTiles layers: two
[TileAlchemist](https://github.com/tilelab/tilealchemist) profiles,
plus the workflow that builds and publishes them every month.

TileAlchemist itself is the pipeline — fetching, sharding, merging,
publishing — and ships no profiles of its own. This repository is one
consumer of it, and doubles as a worked example of the cross-repo path
described in TileAlchemist's
[`docs/PROFILES.md`](https://github.com/tilelab/tilealchemist/blob/main/docs/PROFILES.md).

Both are built from [Protomaps'](https://protomaps.com) daily planet
basemap builds.

| Profile | Produces | Example style |
| --- | --- | --- |
| `land` | Every tile's land polygon(s), derived by inverting the source's surface water. | [`examples/land.json`](examples/land.json) |
| `cropped-waterways` | The source's waterway lines, cropped to the portions that don't overlap real water. | [`examples/cropped-waterways.json`](examples/cropped-waterways.json) |

Neither profile names a layer or an attribute. Each asks for a *feature
set* — `SURFACE_WATER`, `WATERWAYS` — and whichever schema the source
declares answers it (see `TileSchema` in TileAlchemist's
[`docs/PROFILES.md`](https://github.com/tilelab/tilealchemist/blob/main/docs/PROFILES.md)),
so the same two files produce the same two layers from an OpenMapTiles-schema
provider as from Protomaps. Only the `source:` line in the workflow changes.

## Why these two exist

Protomaps' basemap does ship an `earth` layer, but it is the
coastline-derived landmass:
[Natural Earth's `land` theme at low zooms, OSMCoastline's preprocessed land
polygons from z6 up](https://docs.protomaps.com/basemaps/layers). Inland
water is not taken out of it — lakes, reservoirs and wide rivers all sit
inside `earth`, and the style paints `water` back over them afterwards.
That's fine for the default styles, which just want a land-colored ground
to draw water on. But a custom style that wants to treat land as its own
styleable/maskable layer (a distinct fill, a texture, a land-only overlay)
needs the water actually punched out of the polygon rather than covered up
by a later layer. [MapTiler's `land` layer](https://docs.maptiler.com/schema/land/)
is the closest existing reference: plain, attribute-less land polygons
meant as a base layer for custom styling. The **land** profile derives one
by inverting the surface water the source does publish, so what comes out
is land with every lake and river already removed.

(The same holds, for a different reason, of OpenMapTiles-schema providers
such as OpenFreeMap: they ship a `water` layer and no land layer at all.)

Once land replaces water as a style's base layer, the source's waterway
lines (rivers, streams) start visually clashing with it: a stroked line
drawn straight through the water polygon it represents now runs across
solid land-colored fill instead. The **cropped-waterways** profile removes
exactly the overlapping portions, so waterway lines only appear where
they're actually on land. See it combined with `land` in
[`examples/cropped-waterways.json`](examples/cropped-waterways.json).

## Using the prebuilt layers

Finished PMTiles files are published on every run; you don't need to run
the pipeline yourself just to use them. Preview a profile's output directly
in Maputnik:
[`land`](https://maplibre.org/maputnik/?style=https://raw.githubusercontent.com/tilelab/tilealchemist-standardprofiles/main/examples/land.json),
[`cropped-waterways`](https://maplibre.org/maputnik/?style=https://raw.githubusercontent.com/tilelab/tilealchemist-standardprofiles/main/examples/cropped-waterways.json).

Each layer carries its attribution in its own metadata, so a MapLibre style
pointing at it through the
[PMTiles protocol](https://github.com/protomaps/PMTiles) picks it up
automatically, no separate TileJSON or manual attribution string needed.
Today that reads:

> TileAlchemist · extracted from [© OpenStreetMap](https://www.openstreetmap.org/copyright)

Only the words before the credit are written down in this repository. The
rest is whatever the archive that was actually walked declares for itself:
TileAlchemist reads the `attribution` key out of the source PMTiles' own
metadata during `prepare-shards` and substitutes it for the `{source}`
placeholder in the workflow's `attribution` template (for Protomaps, a link
to the OpenStreetMap copyright page; OpenFreeMap declares a longer string
naming itself and OpenMapTiles). Each of those states its own licensing, so
the template here says only what this project did to the data and leaves the
licence to the credit it carries in. The provider credited in a built layer
is therefore always the provider whose bytes went into it, and a run that
can't state what its output credits fails rather than publishing an
unattributed layer — see TileAlchemist's
[`docs/ARCHITECTURE.md`](https://github.com/tilelab/tilealchemist/blob/main/docs/ARCHITECTURE.md)
"Source attribution".

Nothing about that has to be repeated in your style; pointing a source at
the file is the whole of it:

```js
import { Protocol } from "pmtiles";

const protocol = new Protocol();
maplibregl.addProtocol("pmtiles", protocol.tile);

const map = new maplibregl.Map({
  style: {
    version: 8,
    sources: {
      land: {
        type: "vector",
        url: "pmtiles://https://tilealchemist.foxandfeature.com/land.pmtiles",
      },
    },
    layers: [
      { id: "land", type: "fill", source: "land", "source-layer": "land" },
    ],
  },
  // ...
});
```

Or grab a file directly from its GitHub release, tagged `<profile>-latest`
(e.g. [`land-latest`](https://github.com/tilelab/tilealchemist-standardprofiles/releases/tag/land-latest),
[`cropped-waterways-latest`](https://github.com/tilelab/tilealchemist-standardprofiles/releases/tag/cropped-waterways-latest)):
large builds ship as multiple parts, the release description has a
one-line `gh release download` command that reassembles them.

## `land`

For every tile in the `min_zoom`..`max_zoom` pyramid, `LandProfile.transform()`
(`land.py`), called by the inherited
`Profile.transform_tile()`, computes
`water.subtract_water(tile, tile.buffered_square)`, which is:

1. This tile's real surface water, via the `SURFACE_WATER` feature set
   (`tile.features(SURFACE_WATER)`, see `FeatureSet` in
   [TileAlchemist's `docs/PROFILES.md`](https://github.com/tilelab/tilealchemist/blob/main/docs/PROFILES.md)). For `PROTOMAPS` this reads the
   polygons out of the `water` layer — which holds polygons, lines and label
   points together — and drops tunnel water (a water polygon running through
   a tunnel isn't open water at the surface). `Tile.features()` hands each
   remaining polygon back as shapely.
2. Unioned via `tilealchemist/water.py:surface_water_union()`, memoized on
   the `Tile`.
3. Inverting the tile: `land = tile_square - union(remaining water polygons)`,
   entirely in that tile's own local coordinates. `tile_square` is
   `tile.buffered_square`: buffered past the tile edge on every side by
   whatever edge buffer the schema declares its producer wrote — 8px for
   Protomaps (128 units at the standard 4096 extent), what its basemap sets
   on water polygons — so the buffered land polygon is fully determined by
   data already fetched, no neighboring tiles needed. Without this, a
   renderer stroking the coastline as a thick line sees the polygon end
   abruptly at the tile boundary, producing a visible kink where two tiles
   meet instead of a continuous line.

Because each tile is inverted independently against its own square, there's
no cross-tile geometry work and no re-simplification beyond whatever detail
the source archive already has at that zoom.

Gap tiles (no archive entry at all for that tile_id) get an identical bare
buffered square: full land is the well-defined "nothing to subtract" case.

## `cropped-waterways`

`CroppedWaterwaysProfile.transform()`
(`cropped_waterways.py`), called by the inherited
`Profile.transform_tile()`, reads this tile's waterway line features via
`tile.features(WATERWAYS)` (already shapely, see `Tile` in
[TileAlchemist's `docs/PROFILES.md`](https://github.com/tilelab/tilealchemist/blob/main/docs/PROFILES.md)). Protomaps has no waterway layer
as such — the `PROTOMAPS` schema answers `WATERWAYS` with the line
geometries inside the same `water` layer the polygons came from, and the
output layer's fields (`kind`, `name`, `layer`, …) are that schema's own,
via `output_fields()`. Each line is then cut with the
same `water.subtract_water(tile, ...)` the land profile uses, keeping only
the portion that doesn't overlap the water. Both profiles therefore share
one water union per tile - memoized on the `Tile` itself, so the second
profile to ask gets the first one's result rather than recomputing it.
Reusing the exact same union (not recomputing it independently) matters: if
the two profiles' water polygons disagreed even by a sub-pixel amount, a
cropped waterway line and the land polygon it's meant to hug could show a
visible seam once rendered together.

Gap tiles are skipped entirely for this profile: a gap means the source
archive had no water and no waterway data at all for that tile_id, so
there's no faithful "cropped waterway" content to invent, unlike land's
well-defined "whole square minus nothing" case.

## Running it

Both profiles are built together by
`.github/workflows/build-land-and-waterways.yml`, triggered manually
(`workflow_dispatch`, inputs `min_zoom`/`max_zoom`) or on a monthly
schedule. It calls TileAlchemist's `_pipeline.yml` cross-repo, once, with
both profiles (`profile`/`output_basename` accept a comma-separated list),
so one `prepare-shards` walk and one fetch per worker cover both layers and
Protomaps only sees each tile's bytes fetched once per run. The pipeline
never checks this repository out; a first job uploads `land.py` and
`cropped_waterways.py` as the `profiles` artifact, and that is the whole
handover. Publishing —
GitHub Releases via TileAlchemist's `_publish-release.yml`, plus a
Backblaze B2 mirror as a local job — happens here, in the repository that
owns the credentials.

The one source-specific thing in that call is `source: protomaps`. No build
URL appears anywhere: `ProtomapsSource` re-resolves the newest dated build
from
[`build-metadata.protomaps.dev/builds.json`](https://build-metadata.protomaps.dev/builds.json)
at the start of every run, which is not optional here — Protomaps keeps
only the last seven days of daily builds, so a URL pinned in this workflow
would 404 within the week. Protomaps
[discourages hotlinking those builds](https://docs.protomaps.com/basemaps/downloads)
from a deployed map, whose URLs can move; one archive walk per monthly run
is the download case that page describes, not the serving it warns about —
what gets deployed is the `.pmtiles` published from here.

There is nothing to install or configure to trigger a run.

## License / attribution

The code in this repository (profiles, workflow, styles) is licensed under
the [MIT License](LICENSE).

The `.pmtiles` files themselves are a different matter: their data is
derived from OpenStreetMap via Protomaps' daily planet basemap builds
(© OpenStreetMap contributors, ODbL), and that license carries through
however it's reshaped or repackaged downstream. Protomaps publishes those
builds as ODbL Produced Works requiring visible `© OpenStreetMap`
attribution, linked to the OpenStreetMap copyright page that sets out those
terms — which is exactly what the `{source}` half of the attribution
template carries into every layer built here, and why that half is left to
say it rather than restating the licence alongside it. The lowest zooms
additionally rest on public-domain Natural Earth data, which asks for no
attribution of its own; neither profile reads the `landcover` layer, the one
part of a Protomaps build carrying data under a license that would. See
"Using the prebuilt layers" above for the resulting attribution string and
how each `.pmtiles` file carries it automatically.
