# Requirement Extraction

Turn a validated Design Artifact into an explicit statement of what needs to be built. Don't just list UI elements — separate these five categories, and be honest about which ones a static Claude Design frame can and can't answer.

## Screen Purpose

One or two sentences: what is this screen for. Usually inferable from the screen name + the components it composes (a screen with a search bar + table + pagination is a list/browse screen).

## UI Requirements

The component list from the Design Artifact's `components` field, restated in plain language. This is the one category that's fully derivable from the artifact.

## Behavior Requirements

How interactions should work — debounce timing, what triggers a fetch, what a button click does. **A static `.dc.html` cannot show this.** Draft a best guess from structural cues (a `SearchBar` component implies "typing triggers something"), then explicitly confirm specifics with the user before Step 7's Implementation Plan locks them in. Never invent business behavior the design doesn't imply.

## Data Requirements

What data the screen needs, and where a `mock_candidate` from the artifact likely stands in for a real endpoint. Endpoint shape is almost always `Unknown` unless the user provides it — mark it as such rather than guessing a plausible-looking REST path.

## State Requirements

Check for each of: Loading, Empty, Error, Success, Disabled, Selected, Hover, Responsive. The artifact shows exactly one of these (whatever was captured). Default every other state to `Needs Clarification` — do not assume "the rest look like X."

## Output shape

```text
Requirement: User Search

UI:
- SearchInput

Behavior:
- Search on keyword change   [confirmed with user]
- Debounce 300ms             [confirmed with user]

Data:
- GET /users?keyword={keyword}   [confirmed with user — was Unknown in the artifact]

State:
- Loading    [confirmed with user]
- Empty      [confirmed with user]
- Error      [Needs Clarification — not yet asked]
```

Every Behavior/Data/State line either carries `[confirmed with user]` or is still marked `Needs Clarification`/`Unknown` — never a bare assertion with no provenance. This feeds directly into Step 7's Implementation Plan `Unknown`/`Risks` sections.
