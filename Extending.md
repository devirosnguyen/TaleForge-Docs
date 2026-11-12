---
title: Extending TaleForge
---

# Extending TaleForge

[Back to the overview](index.md)

## Your own conditions

**Blueprint:** create a Blueprint class whose parent is **TFCondition**, override **Is Met**, and return true or false. It then
appears in every condition list (Branch nodes, options). The **Context** input gives you:

- **Get Instigator** / **Get Partner**: the player side and the actor talked with
- **Get Context Variable** / **Set Context Variable** / **Context Has Tag**: the variables of any scope
- **Context Random Fraction**: a number from the conversation's seeded stream (keeps replays repeatable)

World functions (Get Player Character, line traces...) work inside Is Met. Override **Describe** to give the condition a
readable one-line summary in editors.

**C++:**

```cpp
UCLASS(meta = (DisplayName = "Is Night"))
class UMyCondition_IsNight : public UTFCondition
{
	GENERATED_BODY()
protected:
	virtual bool IsMet_Implementation(const FTFStoryContext& Context) const override
	{
		const UWorld* World = GetWorld();
		return World && MyGame::IsNight(World);
	}
};
```

The **Invert** checkbox on every condition flips its result; you don't need a "not" version.

## Your own actions

Same pattern with **TFAction** and **Run** (`Run_Implementation` in C++). Actions run when their node is entered or their option
is chosen. For things that belong to game code (open a shop, play a camera shot), a **Send Event** action with a gameplay tag and a
payload name is often enough: bind **On Dialogue Event** on the Story component. To give items, experience or money, use **Give
Reward** and a Reward Receiver (next section).

## Rewards into your game

TaleForge has no inventory, stats or wallet of its own: **Give Reward** describes what to give (kind, your id, amount, display name,
icon) and your game gives it. Implement `ITFRewardReceiver` on whatever owns that state for the player:

```cpp
UCLASS()
class AMyPlayerState : public APlayerState, public ITFRewardReceiver
{
	GENERATED_BODY()
	virtual void ReceiveReward_Implementation(const FTFReward& Reward, UTFStoryComponent* Story) override
	{
		switch (Reward.Kind)
		{
		case ETFRewardKind::Experience: AddExperience(Reward.Amount); break;
		case ETFRewardKind::Currency: AddMoney(Reward.Id, Reward.Amount); break;
		case ETFRewardKind::Item: Inventory->AddItem(Reward.Id, Reward.Amount); break;
		default: break;
		}
	}
};
```

It runs on the server, after the quest changes of the same step (a receiver may start or change quests). The first receiver found
gets each reward: the actor that owns the Story component, then the player state, the pawn and the player controller, each before
its components. `UTFQuestAsset::GetRewardList()` returns the rewards a quest gives on completion (for your own UI), and
`UTFQuestWidgetBase::GetRewardMarkup()` the rich text the journal and the notices show.

## Driving a conversation from C++

`FTFDialogueRunner` holds every rule and needs no world: give it a dialogue and an `FTFStoryContext` (pointers to your
variable stores and Seen set, which must outlive the conversation), then call `Advance` and `Choose`. `UTFStoryComponent`
is a thin wrapper around it (and around `FTFQuestModel` for quests) that adds Blueprint events.

## Feeding quests from your game

Quests never look into your game; your game reports what happens. Pick a gameplay tag tree for it, for example:

| Game code | Call |
|---|---|
| An enemy dies | `Report Quest Event (Enemy.Wolf.Alpha, Target = None, Count = 1)` |
| The player picks up 3 herbs / drops 1 | `Report Quest Event (Item.Herb, None, 3)` / `(Item.Herb, None, -1)` |
| The player enters a trigger volume | `Report Quest Event (Place.Reached, Target = Camp, 1)` |

An objective with Event Tag `Enemy.Wolf` counts every child tag; set its **Target** to count only one NPC, item or place. Objectives
that depend on your own state (inventory, level) are better written as conditions: a custom condition reading your inventory, plus a
call to **Refresh Quests** when the inventory changes.

## Quests from C++

`FTFQuestModel` holds every quest rule and works on an `FTFQuestLog` through an `FTFStoryContext` (its `Quests` pointer, and
`OnQuestChange` to hear about changes). `UTFStoryComponent::MakeContext()` gives a context over the component's own state.

## Cameras, cutscenes, voice and lip sync

- **Shots:** place a Camera Actor where the shot should be, give it an actor tag (`Shot_Well`), and add a **Focus Camera** action
  (Target: Tagged Actor, Camera Tag: `Shot_Well`) to the node that should cut to it. For "look at whoever talks" use Target: Partner and
  give your NPC a Camera component framing its face (the quick start NPC's **Talk Camera** shows how). From code:
  `UTFStoryComponent::SetDialogueCamera(Actor, BlendTime)`; null goes back to the player. The view before the conversation comes
  back when it ends.
- **Cutscenes:** **Play Sequence** plays a Level Sequence where the conversation runs. To wait for it, put the cutscene on a line
  with a **Duration** as long as the sequence, or send an event (**Send Event**) and continue from your own code.
- **Voice:** set a line's **Audio**. The dialogue widget plays it; **Advance When Voice Ends** (UI settings) moves on when it ends.
- **Lip sync / faces:** bind the dialogue widget's **On Voice Started** (Line, Voice). `Line.Speaker` / `Line.Mood` say who speaks
  and how; `Voice` is the playing audio component, which you can feed to your lip-sync or facial animation system. In C++:

```cpp
Hud->GetDialogue()->OnVoiceStarted.AddDynamic(this, &AMyPlayerController::HandleVoice);

void AMyPlayerController::HandleVoice(const FTFLineView& Line, UAudioComponent* Voice)
{
	if (AMyNpc* Npc = Cast<AMyNpc>(Story->GetPartner()))
	{
		Npc->Face->StartTalking(Voice, Line.Mood);
	}
}
```

## Your own widgets in C++

Every TaleForge widget derives from `UTFWidgetBase`: override `BuildDefaultTree()` to build the widget tree when no Blueprint
provides one, and `ApplyTheme()` to style it from `GetTheme()` (called again when the theme or the text scale changes). Widgets
never own rules: they read the Story component and call its functions (`Advance`, `Choose`, `SetTrackedQuest`), so a widget of yours
works in multiplayer as soon as the built-in ones do. For a Blueprint layout instead, see **Your own widgets** in the
[overview](index.md).

## Editor scripting

`UTFEditorLibrary` (Python, Editor Utility Blueprints):

```python
import unreal
quest = unreal.TFEditorLibrary.import_json_asset("D:/Story/Q_Errand.json", "/Game/Story")   # creates or updates, does not save
for issue in unreal.TFEditorLibrary.validate_asset(quest):
    unreal.log_warning(issue)
unreal.EditorAssetLibrary.save_loaded_asset(quest)
```

The quick start content was generated with these same calls. For whole folders, use the `TFData` commandlet described in the
overview.
