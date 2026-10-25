# The design system governs the UI, and prototypes give only layout

Date: 2026-10-24

## Status

accepted

Ruled by the designer over the device reviews of the redesign stack,
zingo-mobile pull requests #1727 to #1750, on 2026-10-23 and 2026-10-24.

## Context

The redesign of Home, Send, Receive and Swap was built from an HTML
prototype. The stack took the prototype's pixel values as they were: type
at 10, 10.5, 11, 14, 21, 24, 28 and 32; radii at 2 to 42; icons at 14, 17,
18 and 22; spacing at 3, 5, 6, 10, 14 and 30. Its phase 0 (#1733) also
changed the theme to match the prototype: `radiusLg` for cards,
`borderSubtle` and the sheet radius.

The same parts came out different from screen to screen. Swap's title bar
used 18 pt type and 24 pt icons where History's used 16 and 20. Four
layouts of label and value rows coexisted. `ContactInitial` drew contact
avatars beside an unused `Avatar`, and `NoticeCard` drew notices beside
`Banner`. A second green, `fgPriceUp`, went from one use to 38. The design
review took several rounds of comments, each naming values one by one.

The theme in `app/theme` and the components in `ui/primitives` and
`ui/widgets` already held the system. Nothing written told an author, or an
agent, that the system outranks a prototype. The ESLint colour lock kept hex
values out of the screens, and every colour in the stack was a token. Type
sizes, radii and icon sizes had no such guard.

## Decision

`app/theme`, the shared components and `DESIGN.md` at the repository root
govern every UI change.

1. A prototype gives the layout, the content and the flow. Each of its
   values maps to the nearest token, text variant or component.
2. A token keeps its value unless the designer changes it.
3. A value or a variant the system lacks goes into `app/theme`, or into the
   shared component as a variant, before a screen uses it. The designer
   approves it, it ships with a Storybook story, and `DESIGN.md` changes in
   the same commit.
4. A rule that a component can enforce lives in that component. The contrast
   rule lives in `SurfaceContext` and `useInsetFill`, and the notice stack's
   single background lives in `Banner`.
5. Departures from the system are listed in `DESIGN.md` under Exceptions,
   each one approved by the designer. Today there are two: the tab bar and
   the two cards of Swap.

`DESIGN.md` follows the DESIGN.md format of google-labs-code/design.md: a
YAML front matter with the tokens, then the rules in prose. The front matter
mirrors `app/theme`. `AGENTS.md` sends every UI change to `DESIGN.md`.

## Considered options

The prototype as the source of truth was rejected. Its values carry no
roles, it is redrawn for each feature, and it already produced the drift
this record answers.

A Figma library as the source of truth was rejected. Agents and reviewers
work from the repository, and a source outside it drifts from the code
without a check.

Keeping the rules in `CONTEXT.md` was rejected. `CONTEXT.md` is the project's
vocabulary; the design rules need tokens, roles and examples, and a format
that design tools read.

A `DESIGN.md` maintained by hand was rejected as the end state. It is the
starting point, until a script generates its front matter.

## Consequences

`DESIGN.md` lands at the root of zingo-mobile. Until
`scripts/design-md` exists, a change to `app/theme` updates the front matter
in the same commit. Once it exists, the script generates the front matter
and CI fails when the two disagree. The DESIGN.md linter runs in CI.

The ESLint rule that locks colours extends to type, radii and icon sizes in
`screens/` and `ui/`: a hand-set `fontSize`, `lineHeight`, `letterSpacing`,
`borderRadius` or icon `size` fails, except inside `app/theme` and in the
tab bar's exception.

Screens the redesign has not reached keep hand-set values and the legacy
text components (`BoldText`, `RegText`, `FadeText`) until each is
redesigned. `DESIGN.md` lists them under Known gaps.

A design review compares a pull request against `DESIGN.md` and names the
rule it breaks, instead of listing values.
