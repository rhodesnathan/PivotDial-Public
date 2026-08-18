# PivotDial (Public)

This is a public companion repository to PivotDial's main (private) codebase.
It exists to share select engineering write-ups and notes from the project
publicly, without exposing the full private source tree.

## About PivotDial

PivotDial is a mobile app that shows centre-pivot irrigation rotation angles
on a satellite map. Drop a pin on the pivot's centre, enter the pivot's
length, and read the rotation angle at any point in the field — either where
you're standing (GPS) or wherever you drag a pin — with no extra hardware, no
subscription, and no signal required out in the field.

The displayed angle is always derived from GPS or map-drawn positions (a
compass bearing from the pivot centre, 0° = north, clockwise), never from the
phone's magnetometer/compass sensor. All angle math runs locally on-device;
there's no server round-trip and no account system.

## Contents

- [`MIGRATION_NOTES.md`](./MIGRATION_NOTES.md) — write-up of the migration
  from `@rnmapbox/maps` to `react-native-maps` (Apple Maps) in the PivotDial
  mobile app: version pinning rationale, New Architecture/RN compatibility
  findings, heading/camera approach, Apple Maps quirks found on-device, and
  the Mapbox → react-native-maps API mapping.

More write-ups may be added here over time.
