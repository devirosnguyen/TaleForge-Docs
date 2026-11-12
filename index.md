---
title: TaleForge
---

# TaleForge

A data-driven dialogue and quest system for Unreal Engine 5, in C++, usable from Blueprint.

> Version 1.0. See [Extending TaleForge](Extending.md) to write your own conditions, actions and widgets.

## Quick start

**Try it first.** In the Content Browser, turn on **Settings -> Show Plugin Content**, open
`TaleForge Content/QuickStart/L_TaleForge_QuickStart` and press **Play**. Walk with WASD (or the left stick) and the mouse, talk with
**E** (gamepad X) when the prompt shows, open the journal with **J** (Tab, gamepad Menu). Mira gives you an errand for Bram, the
guard across the square; come back to her for the reward. Everything there is Blueprint and data, made only from engine content:

| Asset | |
|---|---|
| `BP_TaleForge_QuickStartGameMode` | Uses the character and controller below. Set it as your map's game mode, or copy it |
| `BP_TaleForge_QuickStartPlayerController` | Holds the **Story** component, creates the Story HUD, finds who is in front (**Talk Range**) and shows the "E Talk to ..." prompt; **Theme** sets the look |
| `BP_TaleForge_QuickStartCharacter` | A third-person character made of engine shapes (keys bound in code: no input assets to set up) |
| `BP_TaleForge_QuickStartNpc` | Someone to talk with: a **Dialogue Participant** (Dialogue, Speaker, Participant Id) and a **Talk Camera**, the view a Focus Camera action moves to. Its body takes the speaker's name color |
| `DLG_QuickStart_Mira`, `DLG_QuickStart_Guard`, `Q_QuickStart_Errand` | The conversations and the quest: open them in the editors to see how they are made |
| `SPK_*`, `TH_TaleForge_QuickStart` | Speakers with portraits, and a theme with a coin icon |

To use TaleForge in your own game:

1. Install it from Fab (the launcher installs it into the engine's `Engine/Plugins` folder), then enable **TaleForge** in
   **Edit -> Plugins** and restart the editor. A copy in `<YourProject>/Plugins/TaleForge` works too.
2. Add a **Story** component (`UTFStoryComponent`) to your player controller (or pawn, or player state). It holds the player's
   variables, quests and current conversation.
3. Make a **Speaker** (`TFSpeakerAsset`) per character, **Dialogues** (`TFDialogueAsset`) and **Quests** (`TFQuestAsset`):
   **Content Browser -> Miscellaneous -> Data Asset**.
4. Add a **Dialogue Participant** component to each NPC, set its Dialogue, Speaker and Participant Id.
5. When the player interacts: `Story -> Start Dialogue With (NPC)`. Bind **On Line** and **On Choices** to show text and
   options, call **Advance** to go on from a line and **Choose (index)** to pick an option. **On Dialogue Ended** fires at the end.
6. Give quests from a dialogue (**Start Quest** action) or with **Accept Quest**. Tell quests what happens in your game with
   **Report Quest Event** (a gameplay tag, an optional target, a count). Bind **On Quest Changed** for the tracker and toasts.

## Dialogue

| | |
|---|---|
| Dialogue | Nodes linked by id. A conversation begins at a **Start** node; several Start nodes can be named (Greeting, Shop) |
| Line | A participant role (NPC, Player) says a text, with an optional mood, audio and auto-advance duration |
| Choice | Options for the player. Each option has conditions (hidden, or shown disabled), actions, and can be once-only |
| Branch | Goes to Next when all its conditions pass, else to False Next |
| Event | Runs actions and goes on (set variables, fire events between lines) |
| Random | Picks one output by weight, from the conversation's seeded stream |
| Jump | Continues at a named entry of this or another dialogue |
| End | Ends the conversation (a missing Next ends it too) |
| Participant / Speaker | A dialogue lists roles and their default speakers. The Story component's Player Speaker and the NPC's Participant Speaker override them |

Every action on a node runs when the node is entered; an option's actions run when it is chosen.

### The dialogue editor

Double-click a dialogue to open it in its graph editor (new ones: Add -> Gameplay -> TaleForge -> Dialogue, which starts with a Start
node linked to a first line):

- **Graph:** right-click (or drag a wire into empty space) to add a node, drag wires between pins to link them (a Choice has one pin
  per option, a Random one per output, a Branch True / False). Copy, cut, paste, duplicate and delete work on the selection, with
  undo. **Auto Arrange** lays the nodes out from the Start nodes; a dialogue written without positions (JSON) is arranged when opened.
  The search box finds the next node whose text, name or speaker matches.
- **Node / Dialogue tabs:** the selected node's fields (only those of its type), and the dialogue's own (participants).
- **Issues:** the validator's findings, updated on every change; they also show as bubbles over the nodes. Double-click to jump.
- **Debug:** while you Play In Editor, the node a conversation is on is marked "Running now", the conversation's Local, Actor and
  Global variables are listed, and the Story component is shown below, so you can change the player's variables while playing.

Quests, speakers and themes open in the standard details editor.

## Quests

| | |
|---|---|
| Quest | Title, description, category, prerequisites (conditions), stages, rewards (actions run on completion), fail actions. Repeatable, retryable after failing, tracked on accept |
| Rewards | **Give Reward**: experience, currency, items or anything else, each with your game's id, an amount, a display name and a theme icon. The journal lists them before the quest is done, a notice shows each one when given. Your game gives them in a **Reward Receiver** (below); **Add To Variables** also keeps a count in Global variables (`{Gold}`), with no game code |
| Stage | Journal text, objectives, and where to go when it is done: the first **Link** whose conditions pass, else **Next Stage**; none ends the quest (completed, or failed when **Ends In Failure**). **All** or **Any** required objectives finish it. Enter and complete actions |
| Objective | Progresses when you report an event whose tag is its **Event Tag** or a child of it (`Enemy.Wolf.Alpha` counts for `Enemy.Wolf`), for its **Target** if set, up to **Required Count**; its conditions must also pass. Without an event tag it completes when its conditions pass; with neither, only **Complete Objective** does |
| Roles | **Required** (needed; failing it fails the quest), **Optional** (never blocks), **Fail Trigger** (completing it fails the quest: "the merchant dies") |
| Branch To | Completing this objective ends the stage at once and goes to that stage (kill or persuade the bandit) |
| Time limit | Seconds per objective; running out fails it |
| Talk event | Starting a conversation with a participant reports `TaleForge.Event.Talk` with its Participant Id as target, before the conversation runs |

**Giving rewards to the player.** Implement the **TF Reward Receiver** interface (event **Receive Reward**: the reward and the
Story component) on the player state, the pawn, the player controller or one of their components: TaleForge calls the first one it
finds (the actor that owns the Story component first), on the server. Add the item to your inventory, the experience to your stats.
In a dialogue, a Give Reward action on a node or an option gives at once ("Here, take this").

Objective text can show progress: `Wolves {Count}/{Required}` (**Get Objective Text**). Report negative counts when an item is
dropped; counts never go below 0 or above the required count. Quest conditions are checked again whenever variables change
through the Story component, after each conversation step, and when you call **Refresh Quests** (do it when other game state that
your conditions read has changed).

## Variables, conditions, actions

| | |
|---|---|
| Variables | Bool, int, float or name values plus gameplay tags, in three scopes: **Local** (this conversation), **Actor** (the NPC's Participant component), **Global** (the player's Story component) |
| Conditions | Compare Variable, Has Tags, Chance, Group (All / Any, nestable), Quest Status (optionally at a stage), Objective Completed; each can be inverted. Write your own in Blueprint or C++ |
| Actions | Set Variable (set, add, toggle), Modify Tags, Send Event (received by **On Dialogue Event**), Give Reward, Start Quest, Report Quest Event, Complete Objective, Set Quest Stage, End Quest (fail or abandon), Focus Camera, Play Sequence. Write your own |

**Camera and cutscenes.** **Focus Camera** moves the player's view, with a blend, to the partner (through its first active Camera
component if it has one, else its location and rotation), to the first actor with a **Camera Tag** (place a camera actor for the shot
and tag it), or back to the player. The view goes back when the conversation ends or is stopped. **Play Sequence** plays a Level
Sequence at a play rate. Put them on a Start node for the whole conversation, or on any node or option for one shot. In multiplayer
they run on the server and the player's client follows.

## Text

Lines and options are `FText`, so they are gathered for localization. When shown, `{Speaker}` becomes the speaker's name, a role
name (`{Player}`) that role's speaker name, and a variable name (`{Gold}`) its value (Local, then Actor, then Global). Plural forms
work with int variables: `{Count} {Count}|plural(one=wolf,other=wolves)`. Unknown names stay as written.

## UI

One call shows everything: **Create Story HUD** (Player Controller, Story) in your player controller's Begin Play. It holds:

| Widget | |
|---|---|
| Dialogue (`UTFDialogueWidget`) | Name, portrait (by mood), the line typed out, options (locked ones shown disabled, chosen ones dimmed), continue mark, AUTO / SKIP mark, backlog |
| Quest tracker (`UTFQuestTrackerWidget`) | The tracked quest's title and current objectives, with progress and timers; hidden during conversations |
| Quest notices (`UTFQuestToastWidget`) | New quest, objective complete / failed, quest completed / failed, each reward given (progress too, if enabled); **Show Toast** for your own |
| Journal (`UTFJournalWidget`) | Active / completed / failed tabs, the story so far (earlier stages dimmed), objectives, rewards; track a quest. Bind **Toggle Journal** to your journal key |

**Layouts** (Project Settings -> Plugins -> TaleForge UI -> Layout, or **Set Dialogue Layout** at runtime): **Bottom Box** (RPG),
**Visual Novel** (a large portrait standing on a wide box, options in the middle), **Speech Bubble** (above whoever speaks: the
partner, or the player's pawn for the player's lines).

**Controls** (all keys in the settings): Space / Enter / gamepad A continue (first press finishes the typing), arrows / D-pad / stick
move between options (locked ones are skipped), 1-9 choose directly, mouse click continues or chooses, L / gamepad Y / right click /
wheel up opens the backlog, A / gamepad X toggles auto-advance, hold Ctrl / right trigger to skip until the next choice. In the
journal: up / down select, left / right (or shoulders) change tab, T / gamepad X track, Esc / B / J close. While a conversation or the
journal is open the Story HUD gives it input focus and shows the cursor, and gives input back after (**Manage Input Mode**).

**Text:** lines, options and objectives use rich text. `<Accent>`, `<Emphasis>`, `<Whisper>`, `<Shout>`, `<Item>`, `<Danger>` are
built in; add your own styles in the theme. `<img id="Coin"/>` shows a picture from the theme's **Icons**. A line's **Duration**
advances it by itself; its **Audio** plays as a voice (auto-advance waits for it; with **Advance When Voice Ends** in the settings a
voiced line goes on when its voice ends, auto-advance or not). The dialogue widget's **On Voice Started** gives the line (speaker,
mood) and the playing audio component: the place to start lip sync or facial animation. **Set Text Scale** on the HUD makes every
text larger (accessibility).

### Themes

A **TFTheme** data asset holds the whole look: fonts (body, names, options, titles, small), colors (text, dim text, names or the
speakers' own colors, accent, success, failure, panels, outlines), corner radius, outline width, paddings, sizes (box, bubble,
portraits), typing speed, auto-advance timing, toast time, fade time, sounds (typing, advance, select, choose, notice) and the
named text styles and icons. Panels and options draw rounded boxes, or your own textures (Panel Brush / Option Brush, 9-sliced). Pick
the theme in Project Settings -> Plugins -> TaleForge UI, or call **Set Theme** on the HUD or any widget at runtime.

### Your own widgets

Every widget builds its own tree in C++ when it has none. To change one, make a Widget Blueprint whose parent is the class (e.g.
TFDialogueWidget), lay it out freely, and name the parts you want driven like ours (`SpeakerName`, `LineText`, `Portrait`,
`PortraitBox`, `OptionList`, `ContinueIndicator`, `ModeIndicator`, `Panel`, `BacklogPanel`, `BacklogList`; tracker: `Title`,
`Objectives`; toasts: `List`; journal: `Tabs`, `QuestList`, `Details`, `Hints`; option: `Frame`, `Number`, `Label`). Unnamed parts are
simply not driven; events like **On Line Shown**, **On Options Shown** and **On Refreshed** let the Blueprint do the rest. Then pick
your class in the settings (Story HUD, Dialogue, Option, Tracker, Toast, Journal).

## Structured data: JSON and CSV

Right-click dialogues or quests in the Content Browser -> **TaleForge**:

| | |
|---|---|
| Export to JSON... | One `<Asset>.json` per asset. Edit it in any text editor or generate it from your own tools |
| Update from JSON... | Replaces the asset's content with the file (undoable). Ids are kept, so saves and once-only options keep working |
| Export Texts (CSV)... | Every line, option, stage and objective text, one row each (`Asset, Key, Kind, Speaker, Text`), for writers and translators in a spreadsheet (Google Sheets: File -> Import) |
| Import Texts (CSV)... | Applies the Text column back. Only changed texts are touched |
| Validate | See **Validation** below |

Drag a TaleForge `.json` file into the Content Browser to create a new asset from it. Other JSON files are left to the engine's
importers. Ready-made examples are in `Docs/Examples` (a dialogue and a quest; import the quest first, the dialogue refers to it
at `/Game/TaleForgeExamples/Q_LostCoin`).

The JSON format, by example (see `Docs/Examples` for every field):

```json
{
  "format": "TaleForge.Dialogue", "version": 1,
  "participants": [ { "role": "NPC", "speaker": "/Game/Characters/DA_Mira.DA_Mira" } ],
  "nodes": [
    { "id": "start", "type": "Start", "next": "greet" },
    { "id": "greet", "type": "Line", "speaker": "NPC", "mood": "Happy", "text": "Hello {Player}!", "next": "ask" },
    { "id": "ask", "type": "Choice", "options": [
        { "id": "rich", "text": "I'm rich", "next": "end",
          "conditions": [ { "class": "CompareVariable", "variable": "Gold", "op": "GreaterOrEqual", "value": 100 } ] },
        { "text": "Bye", "actions": [ { "class": "StartQuest", "quest": "/Game/Quests/Q_LostCoin.Q_LostCoin" } ] } ] },
    { "id": "end", "type": "End" }
  ]
}
```

- **Ids:** `"id"` and links (`next`, `falseNext`, `target`, `branchTo`) take a readable name or a guid. A name keeps the id it has
  in the asset; a new name gets an id derived from the asset's id and the name, so importing the same file twice gives the same ids.
- **Conditions and actions:** `"class"` is a built-in short name (`CompareVariable`, `HasTags`, `Chance`, `Group`, `QuestStatus`,
  `ObjectiveCompleted`, `SetVariable`, `ModifyTags`, `SendEvent`, `StartQuest`, `ReportQuestEvent`, `CompleteObjective`,
  `SetQuestStage`, `EndQuest`, `FocusCamera`, `PlaySequence`) or the path of your own class (`/Game/Logic/BP_IsNight.BP_IsNight_C`); the other fields are its
  properties. Story values are written `{"int": 25}`, `{"float": 2.5}`, `{"bool": true}`, `{"name": "Wolf"}`, or by hand as a plain
  `25`, `2.5`, `true`, `"Wolf"`.
- **Texts and localization:** a text whose string did not change keeps its key and its translations; a new or changed string gets
  the key `<id>_Text` in the `TaleForge` namespace, so the engine's localization gather picks every text up with a stable key.
- **Errors:** a file with any error (unknown link, class, type, gameplay tag...) changes nothing; each error says where it is.

For CI or batch work, with the editor closed:

```
UnrealEditor-Cmd.exe MyGame.uproject -run=TFData -validate [-path=/Game/Story]   (exit code 1 on errors)
UnrealEditor-Cmd.exe MyGame.uproject -run=TFData -export=D:/Story/Json [-path=...]
UnrealEditor-Cmd.exe MyGame.uproject -run=TFData -import=D:/Story/Json [-path=...]   (updates by asset name, creates the rest, saves)
```

The same conversions work at runtime (`FTFDataExchange`), e.g. to load story content downloaded or modded after release.

## Validation

**Validate** (menu), Data Validation (on save, if enabled in Editor Preferences) and `-run=TFData -validate` report:

| Errors | Warnings |
|---|---|
| No Start node; two Start nodes with the same entry name; a link to a node or stage that does not exist; a Jump to a missing entry; a Choice without options; duplicate ids or names; an endless loop of nodes that never stop (or of quest stages without objectives); a quest without stages; a Required Count below 1; a link without a target; a reward without an Id or with an Amount below 1 | Nodes or stages never reached; a loop that can only end through a condition; a Branch without conditions; a Random without weights; a speaker role not in Participants; empty texts; texts that cannot be translated (culture invariant); empty entries in condition / action lists; a fail trigger with a Branch To; a Give Reward without rewards |

## Multiplayer

The server owns every story; the Story component replicates to its owning player only.

- **Put the Story component on the player controller** (or the player state, or the pawn). Clients call **Start Dialogue With**,
  **Advance**, **Choose**, **Stop Dialogue** and **Set Tracked Quest** as usual: they are sent to the server. The conversation (line,
  options, partner), the quest log and the Global variables come back to that player, with the same events as on the server, so the
  same UI works in single player, on a listen server and on clients.
- **Change stories on the server:** Accept Quest, Report Quest Event, Set Variable and the other quest and variable calls are server
  calls; on a client they are refused with a warning (a client cannot give itself gold). Report game events where they happen on the
  server (kills, pickups, triggers).
- **Who may talk:** a client's request to talk is checked by **Can Start Dialogue With** (default: the actor has a Dialogue Participant
  with a dialogue, within **Max Talk Distance** of the player's pawn when set). Override it for your own rules.
- **Rewards:** the Reward Receiver is called on the server only; **On Reward Given** fires on the server and on the owning client
  (the notices use it).
- **Party quests:** players whose Story components have the same **Party Id** share quests marked **Shared**: when one accepts it,
  all get it, and what any of them reports (events, completed objectives) counts for all.
- **Camera:** Focus Camera runs on the server and moves that player's view on their client; the end of the conversation puts it back.
- Works with the generic replication system and with Iris.

## Saving

**Save To Slot** / **Load From Slot** on the Story component do everything with a ready save game. To put the story in your own
save game, store the struct from **Make Save Data** and give it back to **Load Save Data**. A save holds the Global variables, what
was seen, every quest (by asset path and id, stages and objectives by id) and the Actor variables of the world's participants (by
Participant Id: set one on spawned NPCs; placed ones default to their actor name).

Saves survive content changes: a quest that no longer exists is dropped, a quest whose saved stage was removed restarts at its first
stage, objectives are matched by id (new ones start at 0). Each fix is logged. **On Story Loaded** fires after a load.

## Safety

A dialogue that loops without a line or choice, links to a missing node, or jumps to a missing entry ends with a warning in the log
(`LogTaleForge`) instead of hanging; so do quest stages that link in a loop, and a link to a missing stage keeps the quest where it is.
If the NPC is destroyed during a conversation, the next step stops it. Rewards and other actions may start or change quests while a
quest is updating; quest changes are broadcast after the update, in order.

## Modules

| Module | Type | Contents |
|---|---|---|
| `TaleForge` | Runtime | Dialogue and quest data, runtime rules, components, saving |
| `TaleForgeUI` | Runtime | Widgets and themes |
| `TaleForgeEditor` | Editor | Asset editors, import/export, validation |

Platforms: Win64. Engine versions: 5.6, 5.7, 5.8.

## Support

devirosnguyen@gmail.com
