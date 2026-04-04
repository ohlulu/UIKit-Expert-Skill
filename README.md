# UIKit Expert Skill
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://github.com/ohlulu/UIKit-Expert-Skill/blob/main/LICENSE)

Expert UIKit guidance for any AI coding tool that supports the [Agent Skills open format](https://agentskills.io/home) — view lifecycle, Auto Layout, table/collection views, navigation, animation, and modern UIKit patterns.

## Who this is for
- Teams maintaining or building UIKit-based iOS apps
- Developers reviewing or refactoring UIKit view controllers and views
- Anyone modernizing legacy UIKit code with diffable data sources, compositional layout, and modern APIs

## How to Use This Skill

### Option A: Using skills.sh
```bash
npx skills add https://github.com/ohlulu/UIKit-Expert-Skill --skill uikit-expert-skill
```

### Option B: Using pi package manager
```bash
pi install https://github.com/ohlulu/UIKit-Expert-Skill
```

### Option C: Manual install
1. Clone this repository
2. Install or symlink `uikit-expert-skill/` following your tool's skills installation docs

## What's Inside

Reference files load on demand, so your agent gets deep guidance only for the topic at hand.

| Reference | Coverage |
|-----------|----------|
| file-structure | Property ordering, extensions, layout placement for VC / View / Cell |
| animation | Duration, curve, fade defaults |
| list-composition | Row/item controllers, section controllers, load-more controller, pagination seam, diffable, compositional layout, default shapes |
| composer | Storyboard or nib instantiation, dependency wiring, navigation closures, scene root composition |
| testing-principles | Test levels, async spies, assertion strategy, memory leak tracking |
| testing | UIKit lifecycle simulation, semantic list helpers, reuse/visibility tests, pagination integration tests |

Non-opinionated: focuses on correctness and performance, not architecture or code style.

## Skill Structure
<!-- BEGIN REFERENCE STRUCTURE -->
```text
uikit-expert-skill/
  SKILL.md
  references/
    animation.md
    composer.md
    file-structure.md
    list-composition.md
    testing-principles.md
    testing.md
```
<!-- END REFERENCE STRUCTURE -->

## Contributing

Contributions welcome — fix incorrect guidance, add modern APIs, expand reference coverage. Keep content UIKit-specific and non-opinionated.

## License

MIT License. See [LICENSE](LICENSE) for details.
