# Progressive Skill Cap Run Preset

## Summary

Add a run preset that keeps chunk-by-chunk task checklists while limiting skill obligations through live skill caps. Caps are derived from available, non-backlogged `Primary` tasks in unlocked chunks.

Core rule: a primary method at level `L` can raise that skill cap to `L + 10`. Level-1 starter methods can raise only to level `15`. Newly reachable methods chain within the same calculation.

An optional chunk pacing cap can further limit the anchor-derived cap based on the number of unlocked chunks. The first tuning target is `(5 * unlocked chunk count) + 20` until the expected cap reaches `40`, then `(2 * unlocked chunk count) + 20` after that. Interpret this monotonically: once the expected cap reaches the threshold, it should not drop below that threshold just because the late formula starts lower. These numbers should be configurable because the right pacing will need playtesting. The chunk pacing cap is only a reducer: it can lower an anchor-derived cap, but it must never raise one.

Unlike the current task model, this preset must not be limited to one active task per skill. The goal is to do everything that is reasonable at the current cap, not just the highest-level representative task.

## Implementation Contract

- Add a new run preset, separate from the existing complete-everything behavior.
- Add worker support for `progressiveSkillCaps`, returning cap details per skill: anchor cap, expected cap, effective display cap, anchor task, anchor level, and whether the starter ceiling or chunk pacing applied.
- Compute caps live from unlocked chunks, current rules, completed tasks, and backlog. Do not persist cap values.
- Treat backlogged tasks as fully excluded: they do not raise caps and do not block rolling.
- Apply caps before task assignment, like an effective `maxSkill`, so high-level skill tasks and downstream requirements do not unlock early.
- Add an optional chunk pacing cap layer that is applied after anchor caps are calculated and only lowers the effective cap.
- Add configurable pacing values so the default formula can be tuned without code changes.
- Track/display current player skill levels so players who out-level the expected cap can still see relevant available tasks.
- Remove the one-task-per-skill limitation for this preset. Skill tasks should be generated as a list of every valid, incomplete, non-backlogged task at or below the effective cap.
- Generate separate progression tasks when a chunk raises a cap, such as `Reach Mining level 15`.
- Show skill caps and their anchor tasks in the UI so players can audit why tasks are required.

## Cap Behavior

- Any non-backlogged `Primary` task can be a training anchor.
- Completed primary tasks can still count as anchors because the method remains available in the chunk.
- Chaining is allowed: if copper raises Mining to 15 and iron is available in the same unlocked area, iron can then raise Mining to 25.
- A high-level method only helps once reachable. Mithril at level 55 does not raise a fresh account to 65 unless the current cap can already reach 55.
- Skill action tasks only appear when their required level is at or below the current effective cap.
- When the optional chunk pacing cap is enabled, final expected cap is `min(anchorCap, chunkPacingCap)`.
- Chunk pacing must run after anchor calculation, not during anchor discovery, so cap audit data can still explain the available training method separately from the pacing limiter.
- Chunk pacing does not raise skills. If the pacing formula returns `40` and the anchor-derived cap is `15`, the effective cap remains `15`.
- Current player skill levels can raise what is shown for already out-leveled players, but only for tasks that are otherwise valid in the current chunks. The intended effective display cap is `min(anchorCap, max(expectedCap, currentPlayerLevel))`, where `expectedCap` includes the optional chunk pacing reducer.
- Example: if a chunk has copper and mithril but no other Mining training, show `Mine copper ore` and `Reach Mining level 15`, but do not show `Mine mithril ore`.

## Multi-Task Skill Behavior

- In the progressive preset, active skill tasks are all valid skill action tasks with effective required level `<= progressiveSkillCaps[skill].effectiveDisplayCap`.
- Do not collapse skill tasks to `highestCurrent[skill]` or `highestOverall[skill]` in this preset.
- Keep existing one-task-per-skill behavior unchanged when the progressive preset is off.
- Completed and backlogged tasks are excluded from the active task list.
- If multiple tasks produce the same output or are alternatives for the same requirement, keep the existing de-duplication/priority rules where they prevent redundant duplicate outputs, but do not remove lower-level tasks merely because a higher-level task exists.
- Progression tasks are additive. A skill can show both action tasks and a `Reach Skill level X` task at the same time.
- Example: if Mining cap is 15 and the chunk has copper, tin, clay, and mithril, show `Mine copper ore`, `Mine tin ore`, `Mine clay`, and `Reach Mining level 15`; hide `Mine mithril ore`.

## Technical Approach

- Preserve the existing worker result shape for normal modes.
- For the progressive preset, have the worker return a multi-task skill payload instead of the current one-task-per-skill `tempChallengeArr` shape. A safe shape is:
  - `progressiveSkillTasks[skill] = [{ task, level, type: "action" | "cap", capAnchor? }]`
  - `progressiveSkillCaps[skill] = { anchorCap, expectedCap, effectiveDisplayCap, chunkPacingCap?, currentPlayerLevel?, anchorTask, anchorLevel, starterCeiling, limitedByChunkPacing? }`
- Update active-task rendering to branch on the progressive payload. Render all entries in `progressiveSkillTasks` under Skill Tasks, using the existing checkbox/backlog/detail UI for action tasks.
- Add a lightweight synthetic detail path for cap tasks, or render cap tasks without `showDetails` if the existing details modal cannot represent synthetic tasks cleanly.
- Update `activeTasks` serialization for this preset so one skill can contain multiple active tasks. Keep compatibility by only using the multi-task shape when the progressive preset is enabled.
- Any code that assumes `.skill-challenge.${skill}-challenge` is a single row must branch for progressive mode or target a task-specific class/id.

## Implementation Steps

Implement this preset in small checkpoints so behavior can be verified after each change.

1. Add the preset flag and UI entry only.
   - Add a new boolean rule such as `Progressive Skill Caps` to the default `rules` object.
   - Add a readable `ruleNames` description and place the rule in the appropriate `ruleStructure` category.
   - Add a new `Progressive Skill Cap` preset in `rulePresets` and `rulePresetFlavor`.
   - Keep this first step behavior-neutral except for allowing the preset/rule to be selected and persisted.
   - Verify: applying the new preset saves/reloads without errors, and existing presets still apply their previous rules.

2. Add an independently testable all-skill-tasks display option.
   - Add a boolean rule such as `Show All Skill Tasks` or `All Skill Tasks`, separate from `Progressive Skill Caps`.
   - When the option is enabled, keep the current task eligibility rules but render every valid, incomplete, non-backlogged skill task instead of only the highest-level representative for each skill.
   - Do not apply progressive caps in this step.
   - Keep existing one-task-per-skill behavior when the option is disabled.
   - Store active skill tasks using the same multi-entry `activeTasks[skill]` shape that progressive mode will later reuse.
   - Update `setupCurrentChallengesFromSaved` so saved maps with multiple skill tasks can render all saved rows.
   - Verify: with the option off, task output is unchanged; with it on, a skill with several currently valid tasks shows all of them, and check/backlog/complete/reload still works.

3. Add worker-side cap calculation helpers without changing active tasks yet.
   - Add a helper for effective skill task level that matches existing boost handling.
   - Add a helper for backlog/completed checks so cap logic and active task logic use the same exclusions.
   - Add `calculateProgressiveSkillCaps(valids)` that scans valid, non-backlogged, `Primary`, non-`Secondary` skill tasks.
   - Start each skill at level `1`, allow level-1 anchors to raise to `15`, allow other reachable anchors to raise to `anchorLevel + 10`, and repeat until no cap changes.
   - Completed primary tasks should be allowed as anchors; backlogged primary tasks should be ignored.
   - Return `progressiveSkillCaps[skill] = { cap, anchorTask, anchorLevel, starterCeiling }`.
   - Verify: log or inspect worker output for a known Mining chunk and confirm copper/tin/clay produce cap `15`, while mithril alone does not raise the cap.

4. Apply caps as an effective `maxSkill` before downstream task assignment.
   - When `rules["Progressive Skill Caps"]` is enabled, use the calculated caps wherever current logic checks `maxSkill` for skill tasks and subskill requirements.
   - Do not mutate or persist `maxSkill`; use a local effective cap map.
   - Preserve existing behavior when the rule is off.
   - Verify: high-level skill tasks and task dependencies above cap no longer become active early, while normal modes remain unchanged.

5. Add the optional chunk pacing cap setting.
   - Add a new optional rule such as `Chunk Skill Pace Limit` that only affects current calculations when `Progressive Skill Caps` is enabled.
   - Add persisted configurable pacing values, initially:
     - base: `20`
     - early per chunk: `5`
     - threshold: `40`
     - late per chunk: `2`
   - Interpret the formula monotonically, for example `earlyCap = base + earlyPerChunk * chunks`; once `earlyCap >= threshold`, use `max(threshold, base + latePerChunk * chunks)`.
   - Add minimal rules/settings UI so the option can be enabled and the per-chunk values can be edited.
   - Do not change task generation in this checkpoint.
   - Verify: enabling/disabling the new option and changing its values persists through save/reload/import/export without changing current tasks yet.

6. Apply chunk pacing as a reducer after anchor cap calculation.
   - Add `calculateChunkPacingSkillCap(unlockedChunkCount, settings)`.
   - Calculate anchor caps exactly as Step 4 does today.
   - If `Chunk Skill Pace Limit` is enabled, calculate `chunkPacingCap` and set `expectedCap = min(anchorCap, chunkPacingCap)`.
   - Keep the anchor cap and anchor task in the cap audit data even when the final expected cap is reduced by pacing.
   - Do not let chunk pacing raise a skill cap under any circumstances.
   - Verify: a skill with an anchor-derived cap in the 70s or 80s is reduced by the pacing cap after 3-4 chunks, while a skill with anchor cap `15` remains `15`.

7. Add current player skill level inputs and cap display semantics.
   - Add UI for current player skill levels, ideally near the skill/cap audit UI or a compact settings panel.
   - Persist player skill levels separately from rules so they can be edited without changing preset definitions.
   - Feed current player skill levels into current task calculation.
   - Use player level as a display/eligibility floor for already out-leveled players: `effectiveDisplayCap = min(anchorCap, max(expectedCap, currentPlayerLevel))`.
   - Preserve the lower `expectedCap` separately so the UI can still show what the preset expects for the chunk count.
   - Verify: if expected cap is `40`, anchor cap is `80`, and the player has level `60`, valid tasks up to `60` show; if the player has level `30`, tasks above `40` stay hidden.

8. Build the progressive multi-task payload.
   - Add `calculateProgressiveSkillTasks(valids, progressiveSkillCaps)` for the new preset.
   - Include every valid, incomplete, non-backlogged skill action task whose effective level is at or below that skill's effective display cap.
   - Reuse the all-skill-tasks rendering/storage path from step 2.
   - Keep existing alternative/deduplication rules where they prevent duplicate outputs, but do not remove lower-level tasks only because a higher-level task exists.
   - Add one synthetic cap task per skill when the cap is above the current completed/passive baseline, using `type: "cap"` and `capAnchor`.
   - Return `progressiveSkillTasks[skill] = [{ task, level, type, capAnchor? }]`.
   - Verify: Mining with copper/tin/clay shows all three action tasks plus `Reach Mining level 15`.

9. Return the new worker fields only for the progressive preset.
   - Add `progressiveSkillCaps` and `progressiveSkillTasks` to the worker response when the rule is enabled.
   - Keep `tempChallengeArrSaved` unchanged or compatible for non-progressive modes.
   - Update `workerOnMessage` to store the new fields in globals.
   - Verify: normal modes still receive and render the old one-task-per-skill payload.

10. Render progressive skill tasks in the active task UI.
   - Branch in `setupCurrentChallenges` when `rules["Progressive Skill Caps"]` and `progressiveSkillTasks` are present.
   - Render action tasks through the all-skill-tasks path from step 2, with existing checkbox, backlog, details, and context-menu behavior.
   - Render cap tasks as synthetic rows, either without `showDetails` or with a small synthetic detail path.
   - Use task-specific class/id fragments so multiple tasks for one skill do not collide with `.skill-challenge.${skill}-challenge`.
   - Verify: checking, backlogging, and opening details works for action tasks, and cap rows do not throw errors.

11. Add the cap audit UI.
   - Add a compact display under Skill Tasks or in the skill/task sidebar showing each progressive cap, anchor task, anchor level, starter-ceiling status, chunk pacing cap, expected cap, current player level, and effective display cap.
   - Only show this UI when the progressive preset is enabled.
   - Make it clear when the chunk pacing cap is reducing the anchor cap, and when player level is allowing tasks above the expected cap.
   - Verify: changing chunks, completing tasks, and backlogging anchors updates the displayed cap source after recalculation.

12. Regression test the rollout.
   - Test with the progressive preset off and compare a few known chunks against current behavior.
   - Test the Mining cases from this document: copper/tin/clay, copper plus mithril, and copper plus iron.
   - Test chunk pacing with a 3-4 chunk route that would otherwise produce a very high cap.
   - Test player levels below, equal to, and above the expected cap.
   - Test a skill with many low-level valid tasks to confirm all eligible tasks render.
   - Test backlogging a cap anchor and confirm it stops raising the cap and stops blocking rolling.
   - Test completed primary anchors and confirm they still raise caps.
   - Test save/reload/import/export paths for both progressive and normal presets.

## Test Plan

- Existing task generation is unchanged when the preset is off.
- Mining with only copper/tin/clay shows the level-1 mining task plus `Reach Mining level 15`.
- Mining with copper/tin/clay shows all available level-1 Mining action tasks, not just one of them.
- Mining with copper plus mithril does not show `Mine mithril ore` until the Mining cap reaches 55.
- Mining with copper plus iron can chain to a higher cap if iron becomes reachable.
- A skill with several valid tasks below cap renders each unfinished, non-backlogged task.
- Backlogging an odd anchor removes its cap effect and its roll requirement.
- UI shows cap and anchor data consistently after changing chunks, rules, completed tasks, and backlog.
- With chunk pacing enabled, a high anchor-derived cap is reduced by the pacing formula and does not force 70s/80s skills after only a few chunks.
- With chunk pacing enabled, the pacing formula never raises a low anchor-derived cap.
- With current player skill levels set above the expected cap, valid tasks up to the player level can appear, while the UI still shows the lower expected cap.

## Defaults

- First version uses `+10` headroom and `15` starter ceiling.
- Chunk pacing starts with `20` base, `5` levels per chunk up to expected level `40`, then the same base and `2` levels per chunk after that, clamped so the expected cap does not fall back below `40`. These are tuning defaults, not hard-coded design constants.
- Weird primary anchors are handled through backlog rather than curated method quality metadata.
