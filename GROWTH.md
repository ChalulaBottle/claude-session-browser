# Growth: Claude Session Browser

Purpose: a Windows desktop app that lists every Claude Code session on the machine and lets you jump back into one with a double click, plus a small desktop buddy that acts out what Claude is doing.
State: growing
Lane: free
Rules: Windows only desktop app (pywebview and WebView2); this repo is a fork of an upstream project, keep changes scoped to this fork; a cloud session cannot launch the desktop app, so new work must be checkable with plain python tests and mocks, not a live run; never touch session data under the user's Claude folder beyond what the existing session reading code already does.

## Next improvements, in order

1. Build the REST wrapper the project's own task list already designs in detail: list, detail, resume and tag update routes over the existing session reading class, bound to loopback only with a shared secret header on every route. Why: the task list says every capability these routes need already exists as a plain method, so this is a transport job, not new logic, and it flags the missing auth as unsafe to leave open. Size: M. Done when: the new routes exist, a request without the shared secret header is rejected, and unit tests cover all four routes against a mocked version of the class, with no need to launch the desktop app.
2. Add a first unit test suite for the two pieces of pure logic that have no tests today: the sprite packer's palette and frame rules, and the updater's version comparison logic. Why: both already encode real rules, such as a nine color limit and a transparency threshold, but a cloud session cannot test anything that needs the desktop app, so these are the parts it can actually prove. Size: S. Done when: a tests folder exists with passing cases for both modules, runnable with plain python and no GUI.
3. Write the short design note the task list asks for before any code: how an outside caller can set the desktop buddy's displayed state directly, instead of only inferring it from the newest transcript, including a timeout so a stale state cannot stick if the caller goes away. Why: the task list calls this the one piece that does not already exist as a function to wrap and says to design it before writing it. Size: S. Done when: the task list gains an answer for the override key, the default timeout, and what happens once it expires.
4. Build the adapter that turns a folder of pixel art frames into both the desktop sprite format and a device firmware header, reusing the packer's existing palette and frame rules instead of a second art pipeline. Why: the task list is explicit that a second visual skin must not need two separate art sources feeding two separate tools. Size: L. Done when: the adapter converts a sample frame folder into both output formats and a round trip test shows the pixel data matches between them.
5. (operator step) Decide whether the already published screenshot that shows real session titles should stay in the repository as is, or be swapped for a redacted capture. Why: the task list itself flags that image as not safe to publish without checking first, and it is already on the public branch. Size: S. Done when: the operator says keep or replace, and if replace, a redacted screenshot takes its place.

## Grown recently

- 2026-10-22: GROWTH.md seeded for the growth rotation.
