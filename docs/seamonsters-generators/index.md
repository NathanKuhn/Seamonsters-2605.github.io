# Generators in the Seamonsters Library

Last year we added a framework to our [Seamonsters Python Library](https://github.com/seamonsters-2605/SeamonstersTemplate) for building robots using Python [Generators](https://wiki.python.org/moin/Generators). The goal was to replace RobotPy's Command framework, which we used in 2017. Generator-based autonomous code looks cleaner and more concise than equivalent Command-based code.

## But what is a generator??

Generators are a way to build iterators, to be used in a "for" loop. Instead of having to build a class which implements the iterator pattern, a generator can be expressed as a single function. Generators use the `yield` command to produce values for iteration. For example:

```python
def my_generator():
    yield 1
    yield 2
    yield 3
    yield 4

for x in my_generator():
    print(x)
# Output:
#   1
#   2
#   3
#   4
```

The `yield` command "pauses" the function and gives control over to the for loop. It saves the location in the function and all of the variable values, and when one loop iteration has completed, it returns back to where it left off.

In this way generators act similar to coroutines&mdash;functions can suspend at their current location and resume later.

## Building a robot using generators

The seamonsters library has a class called `GeneratorBot`. It is modelled after `IterativeRobot`, but instead of having, for example, `teleopInit` and `teleopPeriodic` functions, it has a `teleop` *generator*. This generator completes an iteration 50 times per second, locked to the Driver Station update interval (just like `teleopPeriodic`). This means that the `yield` command is equivalent to saying "pause for 1/50th of a second."

Like Commands, this is a way to avoid using complex state machines to make complex sequences. Generators have some implicit state in that they save their location and variable values when they yield.

## Comparing commands to generators

Commands in RobotPy take the form of classes, which look like this:

```python
class MyCommand(wpilib.command.Command):

    def __init__(self):
        super().__init__()
    
    def initialize(self):
        # immediately before command runs
        pass

    def execute(self):
        # 50 times per second while the command runs
        pass

    def isFinished(self):
        # is the command finished?
        return True

    def interrupted(self):
        # if the command was interrupted before it finished
        pass

    def end(self):
        # after the command completes
        pass
```

In a similar way to building a library of commands for different actions of your robot, you can build a library of generator functions. Generators have similar capabilities to commands but express them in a more concise way. Here is what the command above would look like as a generator:

```python
def my_generator():
    initialize()
    while not isFinished():
        execute()
        yield
    end()
```

Or a more advanced version, which incorporates the `interrupted()` function:

```python
def my_generator():
    initialize()
    try:
        while not isFinished():
            execute()
            yield
    except GeneratorExit: # when the loop breaks or generator is otherwise interrupted
        interrupted()
    finally:
        end()
```

Many of the patterns that the Command framework makes simple and easy, are just as simple and easy with generators. For example, **running commands in sequence.**

With commands:

```python
group = CommandGroup()
group.addSequential(Command1())
group.addSequential(Command2())
```

With generators:

```python
def generator_sequence():
    yield from generator1()
    yield from generator2()
```

> `yield from` is a shortcut that is equivalent to saying:
> 
> ```python
> for _ in generator1():
>     yield
> ```

This is more intuitive than the Command example. Instead of building a group object and adding commands to it at the start of autonomous, you're just calling each of them in order, with a function that seems to stay running throughout the autonomous period. You can even insert extra code in between. This is something that is very simple with generators but would require you to build entire new Command subclasses:

```python
def generator_sequence():
    yield from generator1()
    print("I just finished generator1!!")
    self.output.set(True)
    yield from generator2()
```

The seamonsters library includes several functions to replicate the utility of commands. For example, you can **run generators in parallel:**

```python
def generators_in_parallel():
    yield from sea.parallel(generator1(), generator2())
```

For each iteration of `generators_in_parallel`, one iteration each will be completed of `generator1` and `generator2`. `sea.parallel` will continue until all of the given generators have completed.

Here are some others:

- `sea.wait(count)`: Yield a given number of iterations. You can use this to wait for an amount of time&mdash;`yield from sea.wait(50)` will wait 1 second. Equivalent to `WaitCommand`.
- `sea.timeLimit(generator, count)`: Run a generator with a time limit. After a number of iterations it will be stopped (catch the `GeneratorExit` exception to handle this).
- `sea.watch(generators...)`: Like `sea.parallel`, except all generators will be stopped when the last one that you specified ends. So you can have the end of one action depend on an unrelated event. For example, drive forward until the camera sees a target.

Usually our robot generators don't yield values (they just yield for timing). But yielding values each iteration can be a useful property that commands lack. We had a generator function last year which would yield `True` or `False` if a vision target was visible. We could call it with `sea.untilTrue(generator)`, which would stop it once it yielded True. Or we could call it with `sea.ensureTrue(generator, count)`, which would stop it *only* if it yielded True a certain number of times in a row, to prevent noise.

Generators can also return values after they complete. So the equivalent to a `ConditionalCommand` would look like this:

```python
if yield from conditional_generator():
    yield from generator1()
else:
    yield from generator2()
```

## Conclusion

Generators are neat, and they worked well for us last year! Though there was a learning curve as a different paradigm for programming, they ultimately made it easier for new programmers to see what was happening in autonomous sequences. Here are some examples of how we used generators last year:

- [shooter.py](https://github.com/Seamonsters-2605/CompetitionBot2018/blob/master/shooter.py): This was code for the conveyor. You can see a simple teleop loop, along with some functions for autonomous sequences with some more complicated constructions. `prepGenerator()` is particularly interesting. It has an embedded function called `pulseIntake` which turns on and off the intake motors repeatedly. `sea.watch` is used to make this run until a flag is set, and a `try/finally` block provides an end condition in case the function is interrupted early. All this is expressed relatively concisely, compared to how it might look with commands.

    Since this class implements `GeneratorBot`, it could be deployed to the robot as a standalone file, but for our final `robot.py` we ran this and a few other robot modules together using `sea.parallel`.

    Here you can also see one downside of generators&mdash;they lack the "Subsystem" feature of commands for preventing conflicts. We had to use a flag called `teleopLock` which would be set whenever an autonomous function started, to temporarily pause the teleop loop from running.
- [auto_strategies.py](https://github.com/Seamonsters-2605/CompetitionBot2018/blob/master/auto_strategies.py). We built a library of generators for all of the actions of our robot, then in this file built a function for each autonomous sequence. Mostly these are sequential combinations, but you can also see usage of `sea.timeLimit`, `sea.ensureTrue`, `sea.watch`, etc.


Finally, here is the code for our implementation of generators for robots:

- [bot.py](https://github.com/Seamonsters-2605/SeamonstersTemplate/blob/master/seamonsters/bot.py) for the GeneratorBot
- [generators.py](https://github.com/Seamonsters-2605/SeamonstersTemplate/blob/master/seamonsters/generators.py) for general utilities
