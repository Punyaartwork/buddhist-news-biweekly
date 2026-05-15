# Skill Routing Map — buddhist-news-biweekly

When a candidate is promoted to a Primary Pick, the skill MUST suggest exactly one downstream skill from the Dek Phut Skill Registry. This file is the canonical mapping.

The downstream skill is what the Wiki Agent will invoke to draft the first content version.

---

## Mapping table

| Category | Primary skill | Secondary skill (if applicable) |
|---|---|---|
| Heritage & Archaeology | `dekphut-discovery` | `dekphut-stupa` (only if specifically a stupa / เจดีย์) |
| Notable Figures | `dekphut-kaji` (พระเกจิ) | `dekphut-people` (lay Buddhist figures, scholars) |
| Scholarship | `dekphut-discovery` | `dekphut-write` (wider-interest news angle) |
| Cultural & Festivals | `dekphut-event` | — |
| Cross-cultural Buddhism | `dekphut-discovery` | `dekphut-write` (news angle) |
| Politics & Society | `dekphut-write` | — (neutral tone; flag sensitivity) |
| Controversies | `dekphut-write` | — (only reached when override is ON) |

---

## Selection rule

1. Choose the **Primary skill** for the category by default.
2. Promote to **Secondary** only when the story is clearly a better fit.
3. Never select a skill that is not in the Dek Phut Skill Registry. If the category doesn't fit any registry skill, the candidate must be skipped, not force-routed.

---

## Registry skills (kebab-case, exact names)

- `dekphut-kaji`
- `dekphut-people`
- `dekphut-stupa`
- `dekphut-event`
- `dekphut-policy`
- `dekphut-discovery`
- `dekphut-write`
- `write-dhammapada` *(reserved for Tipitaka / อรรถกถา stories — rare from news scan)*

When in doubt → `dekphut-write` is the safe default for general news/articles.

---

## Versioning

When the Strategy Agent updates the Skill Registry, update this file in the same PR. The skill must not silently route to a renamed/deleted skill.
