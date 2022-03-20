# Composable Alternatives to Bevy's RunCriteria, States, FixedTimestep

This crate offers alternatives to the Run Criteria, States, and FixedTimestep
scheduling features currently offered by the Bevy game engine.

The ones provided by this crate do not use "looping stages", and can therefore
be combined/composed together elegantly, solving some of the most annoying
usability limitations of the respective APIs in Bevy.

## How does this relate to the Bevy Stageless RFC?

This crate draws *very heavy* inspiration from the ["Stageless
RFC"](https://github.com/bevyengine/rfcs/pull/45) proposal for Bevy.

Big thanks to all the authors that have worked on that RFC and the designs
described there.

I am making this crate, because I believe the APIs currently in Bevy are
sorely in need of a usability improvement.

I figured out a way to implement the ideas from the Stageless RFC in a way
that works within the existing framework of current Bevy, without requiring
the complete scheduling API overhaul that the RFC proposes.

This way we can have something usable *now*, while the remaining Stageless
work is still in progress.

## Dependencies

The "run conditions" functionality is always enabled, and depends only on
`bevy_ecs`.

The "fixed timestep" functionality is optional (`"fixedtimestep"` cargo
feature, enabled by default) and adds a dependency on `bevy_core`
(needed for `Res<Time>`).

The "states" functionality is optional (`"states"` cargo feature, enabled
by default) and adds a dependency on `bevy_utils` (to use Bevy's preferred
`HashMap` implementation).

## Run Conditions

This crate provides an alternative to Bevy Run Criteria, called "Run Conditions".

The different name was chosen to avoid naming conflicts and confusion with
the APIs in Bevy. Bevy Run Criteria are pretty deeply integrated into Bevy's
scheduling model, and this crate does not touch/replace them. They are
technically still there and usable.

### How Run Conditions Work?

You can convert any Bevy system into a "conditional system", by calling
`.into_conditional()`. This allows you to add any number of "conditions" on it,
by repeatedly calling the `.run_if` builder method.

Each condition is just a Bevy system that outputs (returns) a `bool`.

The conditional system will present itself to
Bevy as a single big system (similar to Bevy's [system
chaining](https://bevy-cheatbook.github.io/programming/system-chaining.html)),
combining the system it was created from with all the condition systems
attached.

When it runs, it will run each condition, and abort if any of them returns
`false`. The main system will run only if all the conditions return `true`.

(see `examples/conditions.rs` for a more complete example)

```rust
use bevy::prelude::*;
use iyes_loopless::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_system(
            notify_server
                .into_conditional()
                .run_if(in_multiplayer)
                .run_if(on_mytimer)
        )
        .run();
}

/// Condition checking our timer
fn on_mytimer(mytimer: Res<MyTimer>) -> bool {
    mytimer.timer.just_finished()
}

/// Condition checking if we are connected to multiplayer server
fn in_multiplayer(gamemode: Res<GameMode>, connected: Res<ServerState>) -> bool {
    *gamemode == GameMode::Multiplayer &&
    connected.is_active()
}

/// Some system that should only run on a timer in multiplayer
fn notify_server(/* ... */) {
    // ...
}
```

It is highly recommended that all your condition systems only access data
immutably. Avoid mutable access or locals in condition systems, unless are
really sure about what you are doing. If you add the same condition to many
systems, it *will run with each one*.

There are also some helper methods for easily adding common kinds of Run Conditions:
 - `.run_on_event::<T>()`: run if there are events of a given type
 - `.run_if_resource_exists::<T>()`: run if a resource of a given type exists
 - `.run_unless_resource_exists::<T>()`: run if a resource of a given type does not exist
 - `.run_if_resource_equals(value)`: run if the value of a resource equals the one provided
 - `.run_unless_resource_equals(value)`: run if the value of a resource does not equal the one provided

## Fixed Timestep

This crate offers a fixed timestep implementation that uses the Bevy `Stage`
API. You can add a `FixedTimestepStage` to your `App`, wherever you would
like it to run. Typically, a good place would be before `CoreStage::Update`.

It is a container for multiple child stages. You might want to add multiple
child `SystemStage`s, if you'd like to use `Commands` in your systems and
have them applied between each child stage. Or you can just use one if you
don't care. :)

Every frame, the `FixedTimestepStage` will accumulate the time delta. When
it goes over the set timestep value, it will run all the child stages. It
will repeat the sequence of child stages multiple times if needed, if
more than one timestep has accumulated.

(see `examples/fixedtimestep.rs` for a complete working example)

```rust
use bevy::prelude::*;
use iyes_loopless::prelude::*;

fn main() {
    // prepare our stages for fixed timestep
    // (creating variables to prevent code indentation
    // from drifting too far to the right)

    // can create multiple, to use Commands
    let mut fixed_first = SystemStage::parallel();
    // ... add systems to it ...

    let mut fixed_second = SystemStage::parallel();
    // ... add systems to it ...

    App::new()
        .add_plugins(DefaultPlugins)
        // add the fixed timestep stage:
        .add_stage_before(
            CoreStage::Update,
            "my_fixed_update",
            FixedTimestepStage::new(Duration::from_millis(250))
                .with_stage(fixed_first)
                .with_stage(fixed_second)
        )
        // add normal bevy systems:
        .add_startup_system(setup)
        .add_system(do_thing)
        .run();
}
```

Since this implementation does not use Run Criteria, you are free to use
Run Criteria for other purposes. Or better yet: don't, and use the [Run
Conditions](#run-conditions) from this crate instead! ;)

## States

(see `examples/menu.rs` for a complete example)

This crate offers a states abstraction that works as follows:

You create one (or more) state types, usually enums, just like when using
Bevy States.

However, here we track states using two resource types:
 - `CurrentState(T)`: the current state you are in
 - `NextState(T)`: insert this (using `Commands`) whenever you want to change state

### Registering the state type, how to add enter/exit systems

To "drive" the states, you add a `StateTransitionStage` to your `App`. This
is a Stage that will take care of performing state transitions and managing
`CurrentState<T>`. It being a separate stage allows you to control at what
point in your App state transitions happen.

You can add child stages to it, to run when entering or exiting different
states. This is how you specify your "on enter/exit" systems. They are
optional, you don't need them for every state value.

When the transition stage runs, it will check if a `NextState` resource
exists. If yes, it will remove it. If it contains a different value from
the one in `CurrentState`, a transition will be performed:
 - run the "exit stage" (if any) for the current state
 - change the value of `CurrentState`
 - run the "enter stage" (if any) for the next state

### Triggering a Transition

Please do not manually insert or remove `CurrentState<T>`. It should be managed
entirely by `StateTransitionStage`. It will insert it when it first runs.

If you want to perform a state transition, simply insert a `NextState<T>`.
If you mutate `CurrentState<T>`, you will effectively force a transition
without running the exit/enter systems (you probably don't want to do this).

Multiple state transitions can be performed in a single frame, if you insert
a new instance of `NextState` from within an exit/enter stage.

### Update systems

For the systems that you want to run every frame, we provide
a `.run_in_state(state)` and `.run_not_in_state(state)` [run
conditions](#run-conditions).

You can add systems anywhere, to any stage (incl. behind [fixed
timestep](#fixed-timestep)), and make them conditional on one or more states,
using those helper methods.

```rust
use bevy::prelude::*;
use iyes_loopless::prelude::*;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
enum GameState {
    MainMenu,
    InGame,
}

fn main() {
    // prepare any stages for our transitions:

    let mut enter_menu = SystemStage::parallel();
    enter_menu.add_system(setup_menu);
    // ...

    let mut exit_menu = SystemStage::parallel();
    exit_menu.add_system(despawn_menu);
    // ...

    let mut enter_game = SystemStage::parallel();
    enter_game.add_system(setup_game);
    // ...

    // stage for anything we want to do on a fixed timestep
    let mut fixedupdate = SystemStage::parallel();
    fixedupdate.add_system(
        fixed_thing
            // only do it in-game
            .into_conditional()
            .run_in_state(GameState::InGame)
    );

    App::new()
        .add_plugins(DefaultPlugins)
        // Add the "driver" stage that will perform state transitions
        // After `CoreStage::PreUpdate` is a good place to put it, so that
        // all the states are settled before we run any of our own systems.
        .add_stage_after(
            CoreStage::PreUpdate,
            "TransitionStage",
            StateTransitionStage::new(GameState::MainMenu)
                .with_enter_stage(GameState::MainMenu, enter_menu)
                .with_exit_stage(GameState::MainMenu, exit_menu)
                .with_enter_stage(GameState::InGame, enter_game)
        )
        // If we had more state types, we would add transition stages for them too...

        // Add a FixedTimestep, cuz we can!
        .add_stage_before(
            CoreStage::Update,
            "FixedUpdate",
            FixedTimestepStage::from_stage(Duration::from_millis(125), fixedupdate)
        )

        // Add our various systems
        .add_system(
            menu_stuff
                .into_conditional()
                .run_in_state(GameState::MainMenu)
        )
        .add_system(
            animate
                .into_conditional()
                .run_in_state(GameState::InGame)
        )

        .run();
}
