# Changelog

## 1.0.0 (2026-09-29)

First portable release, generalized from the original single-user build.

- Setup command creates each person's pages, config, and scheduled tasks
- Person-specific details moved into a config record; categories are configurable
- Timezone-aware schedules, so daylight saving changes no longer shift run times
- Ridgeline's logic and page template moved from its task prompt into a skill
- Weekly rollup now delivers email through push notification
