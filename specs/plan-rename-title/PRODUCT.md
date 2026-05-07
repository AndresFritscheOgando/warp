# Plan Title Rename

## Summary

Users can manually rename a plan's title directly from the header bar above the rendered plan content. The custom title replaces the auto-generated one everywhere the plan appears: the header, the plan picker menu, and search results.

## Figma

Figma: none provided

## Behavior

### Affordance and triggering edit mode

1. The plan title in the header bar is rendered as an interactive, clickable element. Clicking on the title text once enters edit mode. No double-click is required.

2. When the user hovers over the title area, a pencil/edit icon appears adjacent to the title text. Clicking the pencil icon also enters edit mode. The icon is not visible while not hovered and not while in edit mode.

3. While a plan is currently streaming (the agent has not yet finished writing it), the title is not clickable and the pencil icon is not shown. Renaming is only available once streaming is complete.

4. The plan title is not editable when viewing an earlier version of the plan (version history mode). The affordance and pencil icon are hidden in that state.

### Edit mode

5. When edit mode is entered, the static title text is replaced in-place with a single-line text input. The input is pre-populated with the current title and the text is selected so the user can immediately type a replacement.

6. The text input has focus immediately upon entering edit mode.

7. The text input renders in the same visual position as the title label — same baseline, same width constraints — so the header layout does not shift when edit mode activates.

8. While the input is focused, the pencil icon is not shown.

### Committing and cancelling

9. Pressing **Enter** commits the new title. The input is replaced by the updated static title label. If the input text is non-empty after trimming whitespace, that trimmed string becomes the new title.

10. Pressing **Escape** cancels the rename. The input is replaced by the original title, unchanged.

11. If the user clicks outside the title input (the input loses focus without Enter or Escape), the rename is committed with the current input text, same as Enter.

12. If the committed title is empty (blank or whitespace-only), the title reverts to the previous non-empty title rather than being saved as empty. The fallback label ("Planning document") is never written back as the user's saved title — it only appears when no title has been set at all.

13. After committing, the header immediately reflects the new title without requiring a reload or navigation.

### Persistence and propagation

14. A committed title change is persisted so it survives app restarts. The new title is used in all future renders of the plan: the header bar, the plan picker dropdown/menu, and any search or listing surface that shows plan names.

15. The title persists independently of the plan content. Subsequent agent updates to the plan content do not overwrite a user-set title.

16. If the plan is synced to Warp Drive, the cloud copy reflects the new title after the rename is committed.

### Keyboard and accessibility

17. The edit-mode text input is accessible via keyboard: Tab focus can reach the edit affordance (pencil icon) when hovered, and pressing Space or Enter on the focused icon enters edit mode.

18. Pressing Tab while the title input is focused commits the rename (same as clicking outside) and moves focus to the next focusable element in the header.

### Edge cases

19. If a plan has never had an agent-assigned title (the field is empty), the header displays the fallback label "Planning document". The user can still click the title area to enter edit mode and supply their own title.

20. Very long titles are clipped in the header display (same behavior as the current auto-generated title), but the full title is stored and shown in the plan picker and search results.

21. If the user enters a title that is longer than can be displayed, the input does not truncate the text while the user is typing; clipping only occurs when the label is re-rendered after commit.

22. Concurrent agent update: if the agent sends a title update while the user has the rename input open, the agent update does not overwrite the input contents or dismiss edit mode. When the user commits or cancels, the result from their explicit action wins.
