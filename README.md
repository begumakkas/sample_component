# Code Sample Brief

# AdminUnitSelector

A Svelte 5 + TypeScript component from [InsightOut](https://insightout.civic.garden) that lets a user pick one or more regions (administrative units) within a selected country. It is used across the app, in the survey and map pages.

**My role:** I wrote this component and the `/api/adminunit/` Django endpoint it calls.

## What it does

- Loads the administrative units for the currently selected country from the Django API (`GET /api/adminunit/?country_id=<id>`).
- Renders in two modes, controlled by a `multiple` prop:
  - **Single-select:** a native select dropdown.
  - **Multi-select:** a custom checkbox dropdown with a summary label ("3 regions selected") and click-outside-to-close.
- Reads from and writes to a shared Svelte store (`selectionsStore`), so a selection made on one page is still there when the user navigates to another page.

## How the data flows

1. **Country changes in the store** → the component fetches that country's regions and clears any previous region selection, so a region can never belong to the wrong country.
2. **User picks a region** → an effect writes the selection back to the store (`admin_unit_id` and `admin_unit_name` in single mode, `admin_unit_ids` in multi mode).
3. **On mount** → the component seeds itself from the store, so it shows the right selection when arriving from the survey page or a saved analysis. A single survey selection also seeds the multi-select.
4. **Store is cleared** → local state resets to match.

## Design decisions

- **The store is the source of truth across pages; local state drives the UI.** Effects keep the two in sync in each direction, which keeps the component's rendering simple while letting other components (like the survey generator) react to the selection.
- **Loading and error states are handled in the component.** The dropdown disables while loading, and a failed request shows a message and falls back to an empty list.
- **One component, two modes,** rather than two near-duplicate components, since both share the fetching and store logic.

## Stack

Svelte 5 (runes: `$state`, `$effect`, `$props`), TypeScript, shadcn-svelte `NativeSelect`, Tailwind CSS, Lucide icons. Backend: Django.
