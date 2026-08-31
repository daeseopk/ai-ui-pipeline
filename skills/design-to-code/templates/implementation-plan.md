# Implementation Plan

## Screen
<name from the Design Artifact>

## Purpose
<one or two sentences — Step 2>

## Layout
<pattern from Step 3, or "no StyleGallery match — derived directly from artifact">

## Components

### Reuse
- <component> — <design system (if any) or project source>

### Compose
- <new composite> — built from <existing components>

### New
- <component> — <CREATE_LOCAL or CREATE_SHARED, per gap-analysis.md, with reasoning>

## Data
- <endpoint or data source> — <confirmed with user, or still Unknown>

## State
- <server state → TanStack Query / local → useState / global → Zustand, per convention.md>

## Interaction
- <each confirmed behavior from Step 2, with its confirmed specifics>

## Unknown
- <anything still Unknown/Needs Clarification at plan time — do not silently resolve these during code generation>

## Risks
- <e.g. "UserStatusBadge does not exist yet", "Table API may not fully match Design">

## Implementation Order
1. Layout
2. Data layer
3. <feature-specific steps in build order>
4. Loading / Empty / Error state (only the ones confirmed in Step 2)
5. Responsive
