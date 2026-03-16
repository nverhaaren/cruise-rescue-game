# Save Format

Pichu Rescue Cruise persists game state in two equivalent ways: automatic
browser storage (localStorage) and user-controlled export/import files.
Both use the same JSON structure.

---

## localStorage

**Key:** `pichu-rescue-cruise-save`

The game writes to this key automatically after every meaningful state change
(a Pichu is rescued, a care activity is performed, an upgrade is purchased, or
a cruise is docked). On page load the key is read and the save is restored
silently. If the key is absent (first visit or cleared storage) the game starts
fresh.

---

## Export / Import files

The **💾 Export** button downloads a `.json` file named
`pichu-rescue-save-v<version>.json`. The **📂 Import** button reads any such
file back in, validates the version field, and restores the state.

---

## JSON structure

```json
{
  "version": 1,
  "savedAt": "2026-03-16T12:34:56.789Z",
  "stars": 12,
  "cruiseNum": 3,
  "totalRescued": 7,
  "upgrades": {
    "clinic": true,
    "gym": true
  },
  "pichus": [
    {
      "id": "a1b2c3",
      "name": "Sparky",
      "hp": 72,
      "happy": 55,
      "strength": 40,
      "emoji": "⚡"
    }
  ]
}
```

### Top-level fields

| Field          | Type   | Description                                              |
|----------------|--------|----------------------------------------------------------|
| `version`      | number | Format version. Currently `1`. Used to reject incompatible saves. |
| `savedAt`      | string | ISO 8601 timestamp of when the save was written.         |
| `stars`        | number | Current ⭐ star balance.                                 |
| `cruiseNum`    | number | The cruise number that will be sailed next.              |
| `totalRescued` | number | Cumulative Pichus rescued across all cruises.            |
| `upgrades`     | object | Map of upgrade id → `true` for every owned upgrade. Absent keys are treated as not owned. See [Upgrade IDs](#upgrade-ids). |
| `pichus`       | array  | Pichus currently aboard the ship (empty between cruises). See [Pichu object](#pichu-object). |

### Pichu object

| Field      | Type   | Range  | Description                        |
|------------|--------|--------|------------------------------------|
| `id`       | string | —      | Random identifier, unique per run. |
| `name`     | string | —      | Display name chosen from the name pool. |
| `hp`       | number | 0–100  | Health points.                     |
| `happy`    | number | 0–100  | Happiness level.                   |
| `strength` | number | 0–100  | Strength level.                    |
| `emoji`    | string | —      | Always `"⚡"` in v1.              |

### Upgrade IDs

| ID        | Ship feature unlocked         |
|-----------|-------------------------------|
| `clinic`  | Ship Clinic (double heal)     |
| `theater` | Mini Theater                  |
| `gym`     | Gym Room + `gym` activity     |
| `cafe`    | Pichu Café + `feed` activity  |
| `pool`    | Splash Pool + `swim` activity |
| `spa`     | Cozy Spa + `spa` activity     |

---

## Versioning

The `version` field allows the game to detect incompatible saves. If the
version in a file does not match the current `SAVE_VERSION` constant in the
game code, the import is rejected with an error toast and no state is changed.

When a future version introduces breaking changes to the schema, `SAVE_VERSION`
should be incremented and a migration path documented here.
