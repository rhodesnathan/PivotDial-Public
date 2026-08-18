# Migration: `@rnmapbox/maps` → `react-native-maps` (Apple Maps)

**Status: complete.** Mapbox is fully removed from the app. Satellite imagery is
rendered by `react-native-maps` using `PROVIDER_DEFAULT` (Apple Maps on iOS).

Shipped across PRs #29 (spike), #31 (port), and the cleanup commit that removed
the Mapbox dependency, config plugin, tokens, and the temporary `/maptest`
screen. Paths are relative to repo root unless noted.

> Note on the original goal: this migration was scoped as "move to Google Maps".
> It landed on **Apple Maps** instead — `PROVIDER_DEFAULT` needs no API key and
> no config plugin on iOS, so it was the shorter path to working imagery. Google
> remains available on the same library as a one-line contingency (see below).

---

## 1. Final versions

| Thing | Value |
| --- | --- |
| `react-native-maps` | **`1.29.0`** — pinned exactly, **no caret** |
| React Native | **`0.86.0`** |
| Expo SDK | `~57.0.7` |
| React | `19.2.3` |
| New Architecture | **ON** (Expo SDK 57 default; `newArchEnabled` is not explicitly set anywhere) |
| Provider | `PROVIDER_DEFAULT` → Apple Maps (iOS). Android out of scope |
| Google Maps API key | **none** — not required for `PROVIDER_DEFAULT` |
| `react-native-maps` config plugin | **not added** — it is a no-op without `iosGoogleMapsApiKey` |

**The exact pin is deliberate.** We are on an unverified RN 0.86 pairing (see
§2), and a silent minor bump ahead of the September field trip is unacceptable.
Do not restore a `^` range without re-running the on-device checks in §6.

---

## 2. New Architecture / RN 0.86 compatibility conclusion

Read from the installed package, not from vendor marketing:

- `react-native-maps` **1.29.0 declares New Architecture (Fabric) support with a
  floor of RN >= 0.81.1** (README Fabric compatibility table: `1.26.1+` →
  `>= 0.81.1`; `1.26.0 and below` → `>= 0.76`).
- Its `package.json` ships `codegenConfig` (`RNMapsSpecs`) with iOS Fabric
  component providers, and `peerDependencies` allow `react-native >= 0.76.0`.
- **RN 0.86 clears that floor but is not explicitly enumerated**, and the
  library's own `devDependencies` pin `react-native ^0.83.10` — i.e. their CI
  targets 0.83.10, not 0.86.

**Conclusion: the 1.29.0 + RN 0.86 pairing is verified empirically on device by
us, not by the vendor.** It sits inside the declared support range but outside
the range the vendor actually tests. That is the entire reason the version is
pinned exactly and the reason the `/maptest` spike existed before the port.

If RN or `react-native-maps` is upgraded, re-verify on a physical device; a
green typecheck and passing unit tests do **not** cover Fabric view rendering.

---

## 3. Heading approach chosen, and why

**Chosen: sample the camera heading when a gesture settles, via
`onRegionChangeComplete` + `getCamera()`.**

`apps/mobile/src/components/FieldMap.tsx`:

```ts
const syncHeading = useCallback(async () => {
  const cam = await mapRef.current?.getCamera();
  if (cam) setHeading(cam.heading ?? 0);
}, []);
// ...
<MapView onRegionChangeComplete={syncHeading} … />
```

**Why.** Mapbox exposed a continuous `onCameraChanged` event carrying
`state.properties.heading` synchronously, which the old code throttled with a
0.5° threshold. `react-native-maps` has no equivalent synchronous heading
payload — `getCamera()` is a **Promise**. Sampling once per settled gesture
avoids an async `getCamera()` call per frame.

**Heading here means the map camera's bearing, not the device compass.** There
is no magnetometer/`watchHeadingAsync` usage anywhere; that is a hard constraint
(HC-1), and CI has a static check for it.

**Known trade-off, accepted:** the compass needle updates on gesture-end rather
than continuously, so it does not counter-rotate live mid-twist. Switching to
`onRegionChange` (continuous) would restore live tracking at the cost of an
async `getCamera()` per frame. It drives only the reset-to-north button's
visibility (`heading > 1 && heading < 359`) and the needle rotation, so the
settled-sample behaviour is sufficient.

---

## 4. Apple Maps has no zoom level — only altitude

**`animateCamera({ zoom })` does not work on Apple Maps. Do not reintroduce it.**

The `Camera` type carries `altitude` (Apple) and `zoom` (Google) as separate
optional fields. On `PROVIDER_DEFAULT`, `getCamera()` returns `altitude` and
leaves `zoom` undefined.

**Every framing operation goes through `fitToCoordinates(coords, { edgePadding })`,**
which is provider-agnostic and expressed in real-world coordinates rather than a
zoom number:

- initial framing → `fitToCoordinates` on the pivot's outer-circle bounds
- the `zoomLevel` prop is retained on `FieldMapProps` only for prop-contract
  stability; it no longer drives the camera

**There is deliberately no altitude↔zoom conversion math anywhere in the
codebase, and none should be added.** Converting between them is provider- and
latitude-dependent and was the single biggest source of framing bugs during the
spike.

**Observed altitude floor: 303 m.** That is the closest Apple Maps satellite
imagery would zoom in over the target farmland during on-device testing —
sufficient for pivot-scale work. Treat it as the practical imagery ceiling when
reasoning about how tight the map can frame.

---

## 5. Contingency: switching to Google

If Apple Maps imagery fails in the field, switch providers **on the same
library** — no re-migration:

1. `provider={PROVIDER_GOOGLE}` instead of `PROVIDER_DEFAULT` in
   `FieldMap.tsx` (one line).
2. Supply the **Google Maps iOS API key, already created in Google Cloud project
   `pivotdial`**, via the `react-native-maps` config plugin
   (`iosGoogleMapsApiKey`). This requires a **native rebuild**, not an OTA.

Caveats to remember if this is used:
- Google Maps Platform terms **prohibit offline tile caching** in third-party
  apps. Switching to Google re-imposes the constraint that originally drove the
  Mapbox choice — relevant to the deferred offline release (see
  `docs/calibration-spec.md` §8).
- `anchor` works on Google but `centerOffset` is the Apple path; both are already
  set from one `PIN_GEOMETRY` table, so pins are correct on either provider.
- Google populates `camera.zoom`; Apple populates `camera.altitude`. Framing
  still goes through `fitToCoordinates`, so no code change is needed.

---

## 6. Apple-Maps-specific issues found on device

Both were caught only by running on a physical device — neither is visible to
typecheck or unit tests.

1. **Pin tip anchoring.** `anchor` is **Google Maps only on iOS**; Apple Maps
   ignores it and centres the marker view on the coordinate. The pivot arm
   therefore lined up with each teardrop's *head* instead of its point. Apple's
   equivalent is **`centerOffset`** (in points). A single `PIN_GEOMETRY` table
   now holds each shape's rendered size and tip fraction and derives both props
   from it: `drop` → `-13.33`, `numbered` → `-18.5`, `dot` → `0`.
2. **Attribution overlap.** A `legalLabelInsets` override printed the "Legal"
   link on top of the  Maps logo. It was unnecessary — every screen puts its
   bottom sheet in a sibling view *below* the map, and the map's floating buttons
   sit on the right, so the bottom-left corner is never covered. The override was
   removed; MapKit lays the attribution out itself. **Do not re-add
   `legalLabelInsets`** — MapKit positions the logo and Legal link as a pair and
   expects to own that corner, and obscuring the link is an App Store review risk.

---

## 7. What was removed in cleanup

| Removed | Where |
| --- | --- |
| `@rnmapbox/maps` dependency | `apps/mobile/package.json` (+ lockfile) |
| `'@rnmapbox/maps'` config plugin | `apps/mobile/app.config.js` `plugins` |
| `EXPO_PUBLIC_MAPBOX_ACCESS_TOKEN`, `MAPBOX_DOWNLOADS_TOKEN` | `apps/mobile/.env` (git-ignored) and `.env.example` |
| `src/mapbox.ts` (+ its `_layout.tsx` side-effect import) | removed during the port (PR #31) |
| `app/maptest.tsx` and its route | temporary spike screen |
| Settings → "Developer" card linking to `/maptest` | `app/settings/[id].tsx` |

**Still to be deleted manually — EAS environment variables** (they exist in all
three environments and are not removable from this repo):

| Variable | Environments |
| --- | --- |
| `EXPO_PUBLIC_MAPBOX_ACCESS_TOKEN` | `development`, `preview`, `production` |
| `MAPBOX_DOWNLOADS_TOKEN` (secret) | `development`, `preview`, `production` |

Both are now unreferenced by any code, config, or build step. Delete via the EAS
dashboard or `eas env:delete`. Consider also revoking the tokens in the Mapbox
account, since the public token was inlined into previously published JS bundles.

---

## 8. Structural facts that survived the migration

- **`packages/geometry` was not modified**, and neither was
  `apps/mobile/src/logic/circleOverlay.ts`. The port only consumes values they
  already produce. `fromENU(centre, {e,n})` and `bearing(centre, pos)`
  (north-zero, clockwise) are map-library-agnostic.
- **One adapter** bridges the two coordinate conventions: the geometry package
  emits GeoJSON-order `[lon, lat]`; `react-native-maps` wants
  `{ latitude, longitude }`. `toCoord()` in `FieldMap.tsx` is the single
  conversion point.
- **The `FieldMapProps` contract is unchanged** — `centre`, `radiusM`, `endGunM`,
  `travelArc`, `pins`, `armAngleDeg`, `fitCoords`, `onMapPress`, `userLocation`.
  No caller changed: `app/setup.tsx`, `app/readout/[id].tsx`, and
  `app/settings/[id].tsx` were untouched by the port.

### API mapping used

| Mapbox | react-native-maps |
| --- | --- |
| `styleURL` (satellite style URL) | `mapType="satellite"` |
| `ShapeSource` + `LineLayer` | `<Polyline>` (ring, end-gun ring, north tick, arm) |
| `lineDasharray` (line-width units) | `lineDashPattern` (points) — rescaled |
| `ShapeSource` + `FillLayer` | `<Polygon fillColor>` (rotation wedge) |
| `PointAnnotation` | `<Marker>`; `onSelected` → `onPress`; drag coord from `event.nativeEvent.coordinate` |
| `compassEnabled={false}` | `showsCompass={false}` (our reset-to-north button replaces Apple's compass) |
| `<Camera defaultSettings>` | `initialRegion` |
| `Camera.fitBounds(ne, sw, padding)` | `mapRef.fitToCoordinates(coords, { edgePadding })` |
| `Camera.setCamera({ heading: 0 })` | `mapRef.animateCamera({ heading: 0, pitch: 0 })` |

Custom-view markers set `tracksViewChanges` **false** after their content
settles (the `PivotMarker` wrapper). Leaving it true re-snapshots every frame —
a real performance sink on the New Architecture.
