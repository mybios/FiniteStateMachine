# Finite-State-Machine

This project implements a Finite-State-Machine (FSM) designed to be used in games.

Furthermore it implements even a Stack-Based-FSM (SBFSM). So you may tell it to 'continue with the last state before the active one'.

You describe your FSM using a nice and well documented DSL (Domain Specific Language).

## Description

This replaces the code we usually had for keyboard-input (run-left-right-duck-jump), clicked buttons on the GUI (idle-over-down-refreshing), tower-states (idle-aiming-firing-reloading) or for the connection procedure when setting up peer2peer connections in our games (more complex; example further down).

The idea is to generate a single FSM for every 'layer' of input that your engine allows.
In our example it's a multi-button GUI. Some of the buttons will stay pressed and the GUI will enter a 'selection-grid-mode' until the left button is pressed again. Some of them just do immediate actions and become immediately released afterwards. Some of both of those have a refresh-time and stay disabled for that period.
In this example the state of the GUI ('selection-grid-mode' or not) would be such a machine.

We place those machines in a single class where they could 'talk with each other' by reading their respective states. That way it is possible to construct and react on compound states.

A nice example for such a machine is the setup of a multiplayer game.
It would be like the following:

##### Server:

* Send the 'load level' signal to other players
* Load level
* Display 'waiting for other players' message-box
* Wait for all other players to finish loading
* Send the level-data to other players and wait for acknowledgement from each of them
* Remove the 'waiting for other players' message-box
* Send 'start' signal to other players
* Start the game


### Test-drive

Time to take it for a test-drive.

The motivation for this project came from a nice article I found [here](http://gameprogrammingpatterns.com/state.html) which comes with some examples. We tried to solve the proposed problems with our new project.

*By the way: [This](http://gameprogrammingpatterns.com/) seems to be a great book, so try to support the author in any way possible for you.*

He's making a point using a FSM that looks like this:

* ducking --(release down)--> standing
* standing --(press down)--> ducking
* standing --(press B)--> jumping
* jumping --(press down)--> diving

So the file ```GameProgrammingPatterns1.cs``` in the test-folder contains that machine.

## Usage

This is a short paragraph that is about an example what configuring a state machine actually looks like.

```c#
private enum VState { DUCKING, STANDING, JUMPING, DESCENDING, DIVING };
private enum VTrigger { DOWN_RELEASED, DOWN_PRESSED, UP_PRESSED, SPACE_PRESSED };

private enum HState { STANDING, RUNNING_LEFT, RUNNING_RIGHT, WALKING_LEFT,
                     WALKING_RIGHT, WALKING_DELAY_LEFT, WALKING_DELAY_RIGHT};
private enum HTrigger { LEFT_PRESSED, LEFT_RELEASED, RIGHT_PRESSED, RIGHT_RELEASED };

private Fsm<State, Trigger> verticalMachine;
private Fsm<State, Trigger> horizontalMachine;

private Keys[] lastKeysPressed;
private Hero hero;

public void main() {
  horizontalMachine = Fsm.Builder<HState, HTrigger, GameTime>(STANDING)
    .State(STANDING)
      .TransisionTo(WALKING_LEFT).On(LEFT_PRESSED)
      .TransisionTo(WALKING_RIGHT).On(RIGHT_PRESSED)
      .OnEnter(e => {
        ConsoleOut();
        hero.HAnimation = HAnimation.STANDING;
        hero.delayTimer.StopAndReset();
      })
    .State(WALKING_LEFT)
      .TransisionTo(WALKING_DELAY_LEFT).On(LEFT_RELEASED)
      .OnEnter(e => {
        ConsoleOut();
        hero.HAnimation = HAnimation.WALK_LEFT;
        hero.delayTimer.StopAndReset();
      })
    .State(WALKING_RIGHT)
      .TransisionTo(WALKING_DELAY_RIGHT).On(RIGHT_RELEASED)
      .OnEnter(e => {
        ConsoleOut();
        hero.HAnimation = HAnimation.WALK_RIGHT;
        hero.delayTimer.StopAndReset();
      })
    .State(WALKING_DELAY_LEFT)
      .TransisionTo(WALKING_RIGHT).On(RIGHT_PRESSED)
      .TransisionTo(RUNNING_LEFT).On(LEFT_PRESSED)
      .OnEnter(e => {
        hero.delayTimer.Start();
      })
      .Update(a => {
        hero.delayTimer.Update(a.GameTime);
        if(hero.delayTimer) {
          horizontalMachine.TransitionTo(STANDING);
        }
      })
    .State(WALKING_DELAY_RIGHT)
      .TransisionTo(WALKING_LEFT).On(LEFT_PRESSED)
      .TransisionTo(RUNNING_RIGHT).On(RIGHT_PRESSED)
      .OnEnter(e => {
        hero.delayTimer.Start();
      })
      .Update(a => {
        hero.delayTimer.Update(a.GameTime);
        if(hero.delayTimer) {
          horizontalMachine.TransitionTo(STANDING);
        }
      })
    .State(RUNNING_LEFT)
      .TransisionTo(STANDING).On(LEFT_RELEASED)
      .OnEnter(e => {
        ConsoleOut();
        hero.HAnimation = HAnimation.RUNNING_LEFT;
        hero.delayTimer.StopAndReset();
      })
    .State(RUNNING_RIGHT)
      .TransisionTo(STANDING).On(RIGHT_RELEASED)
      .OnEnter(e => {
        ConsoleOut();
        hero.HAnimation = HAnimation.RUNNING_RIGHT;
        hero.delayTimer.StopAndReset();
      })
    .GlobalTransitionTo(STANDING).On(SPACE_PRESSED)
    .Build();
  
  verticalMachine = Fsm.Builder<VState, VTrigger, GameTime>(STANDING)
    .State(STANDING)
      .TransisionTo(DUCKING).On(DOWN_PRESSED)
      .TransisionTo(JUMPING).On(UP_PRESSED)
      .OnEnter(e => {
        ConsoleOut();
        hero.VAnimation = VAnimation.IDLE;
      })
      .OnExit(Console.Out.WriteLine($"From [{e.From}] with [{e.Input}] to [{e.To}]"))
    .State(DUCKING)
      .TransisionTo(STANDING).On(DOWN_RELEASED)
      .OnEnter(e => {
        ConsoleOut();
        hero.VAnimation = VAnimation.DUCKING;
      })
      .OnExit(ConsoleOut)
    .State(JUMPING)
      .TransisionTo(DIVING).On(DOWN_PRESSED)
      .OnEnter(e => {
        ConsoleOut();
        hero.VAnimation = VAnimation.JUMPING;
      })
      .OnExit(ConsoleOut)
      .Update(a => {
        hero.height += a.GameTime.ElapsedGameTime.TotalSeconds * 100F;
        if(hero.height >= 200F)
          verticalMachine.TransitionTo(DESCENDING);
      })
    .State(DESCENDING)
      .TransisionTo(DIVING).On(DOWN_PRESSED)
      .OnEnter(e => {
        ConsoleOut();
        hero.VAnimation = VAnimation.DESCENDING;
      })
      .OnExit(ConsoleOut)
      .Update(a => {
        hero.height -= a.GameTime.ElapsedGameTime.TotalSeconds * 100F;
        if(hero.height <= 0F) {
          hero.height = 0F;
          verticalMachine.TransitionTo(STANDING);
        }
      })
    .State(DIVING)
      .TransisionTo(DESCENDING).On(DOWN_RELEASED)
      .OnEnter(e => {
        ConsoleOut();
        hero.VAnimation = VAnimation.DIVING;
      })
      .OnExit(ConsoleOut)
      .Update(a => {
        hero.height -= a.GameTime.ElapsedGameTime.TotalSeconds * 150F;
        if(hero.height <= 0F) {
          hero.height = 0F;
          verticalMachine.TransitionTo(STANDING);
        }
      })
    .GlobalTransitionTo(STANDING).On(SPACE_PRESSED)
    .Build();
}

protected override void Update(GameTime gameTime) {
  if (GamePad.GetState(PlayerIndex.One).Buttons.Back == ButtonState.Pressed ||
      Keyboard.GetState().IsKeyDown(Keys.Escape))
    Exit();
  
  var s = Keyboard.GetState();
  if (s.IsKeyDown(Keys.Up))
    verticalMachine.Trigger(UP_PRESSED);
  if (s.IsKeyDown(Keys.Down))
    verticalMachine.Trigger(DOWN_PRESSED);
  if (!s.IsKeyDown(Keys.Down) && lastKeysPressed.Contains(Keys.Down))
    verticalMachine.Trigger(DOWN_RELEASED);
  
  if (s.IsKeyDown(Keys.Left))
    horizontalMachine.Trigger(LEFT_PRESSED);
  if (s.IsKeyDown(Keys.Right))
    horizontalMachine.Trigger(RIGHT_PRESSED);
  if (!s.IsKeyDown(Keys.Right) && lastKeysPressed.Contains(Keys.Right))
    horizontalMachine.Trigger(RIGHT_RELEASED);
  if (!s.IsKeyDown(Keys.Left) && lastKeysPressed.Contains(Keys.Left))
    horizontalMachine.Trigger(LEFT_RELEASED);
  
  lastKeysPressed = s.GetPressedKeys();
  
  // Update the machines themselves.
  verticalMachine.Update(gameTime);
  horizontalMachine.Update(gameTime);
}

private void ConsoleOut(TransitioningValueArgs<string> e) {
  Console.Out.WriteLine($"From [{e.From}] with [{e.Input}] to [{e.To}]");
}
```



Another example with a spell-button that has a refresh-time:

```c#
private enum State { IDLE, OVER, PRESSED, REFRESHING };
private enum Trigger { MOUSE_CLICKED, MOUSE_RELEASED, MOUSE_OVER, MOUSE_LEAVE };

private Dictionary<Button, Fsm<State, Trigger, GameTime>> buttonMachines = new
  Dictionary<Button, Fsm<State, Trigger, GameTime>>();

private void CreateMachineFor(Button button)
  buttonMachines.Add(button, Fsm.Builder<State, Trigger, GameTime>(IDLE)
    .State(IDLE)
      .TransisionTo(OVER).On(MOUSE_OVER)
      .OnEnter(e => {
        button.State = ButtonState.IDLE;
      })
    .State(OVER)
      .TransisionTo(IDLE).On(MOUSE_LEAVE)
      .TransisionTo(PRESSED).On(MOUSE_CLICKED)
      .OnEnter(e => {
        button.State = ButtonState.OVER;
      })
    .State(PRESSED)
      .TransisionTo(IDLE).On(MOUSE_LEAVE).If(button.Kind == Kind.FLIPBACK)
      .TransisionTo(REFRESHING).On(MOUSE_RELEASED)
      .OnEnter(e => {
        button.State = ButtonState.DOWN;
      })
    .State(REFRESHING)
      .OnEnter(e => {
        hero.doSpell(button.DoAssociatedSpell());
        button.RefreshTimer.Start();
        button.State = ButtonState.REFRESHING;
      })
      .Update(a => {
        if(button.RefreshTimer.Value <= 0F) {
          button.RefreshTimer.StopAndReset();
          machine.TransitionTo(IDLE);
        }
      })
    .Build();
}

public void main() {
  Button b1 = new Button("name1", "someText", ...);
  Button b2 = new Button("name2", "someOtherText", ...);
  
  CreateMachineFor(b1);
  CreateMachineFor(b2);
  ...
}
```

## Inspired by:

* [Fluent-State-Machine](https://github.com/Real-Serious-Games/Fluent-State-Machine) - by RoryDungan (MIT License)
* [Nate](https://github.com/mmonteleone/nate) - by mmonteleone (MIT License)