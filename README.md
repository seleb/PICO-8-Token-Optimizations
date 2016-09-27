# PICO-8 Token Optimizations
A few of these are pretty obvious, a few of them are not that obvious. Almost all of them make your code more unreadable, harder to edit, and would generally be considered "bad practice" in a less constrained system.

Feel free to suggest changes, corrections, or other tricks!

## Negative Literals as Hexadecimals
The negative sign counts as a token, so instead of writing negative numbers as you would normally (i.e. `-10`) you can write them using hexadecimal numbers which are larger than PICO-8 allows (i.e. `0xFFFFF6`). This causes an overflow resulting in the same number. As a quick shorthand, `0xFFFFFF` is `-1`, `0xFFFFFE` is `-2`, etc.
- Use when: assigning, multiplying, or dividing negative literals.
- Caveats: Using hexadecimal numbers takes up more characters. Doesn't work with addition/subtraction because adding negative numbers can be replaced with a subtraction, and subtraction needs the `-` operator.
- Saves: 1 token per number

## Calling Functions with Strings
Instead of calling functions with brackets (i.e. `FUNC()`) you can call them using strings (i.e. `FUNC""`). On its own, this doesn't save any tokens, but the string used to call the function will be passed in as the first argument at no extra token cost. This means that `FUNC("STRING")`, a 3-token statement, is the same as `FUNC"STRING"`, a 2-token statement.

In most cases, this format will also work for a single number contained in a string, e.g. `BTN(0)` can safely be replaced with `BTN"0"`. It's important to note that the argument is still passed in as a string, so some user-defined functions may not work as expected in this format, e.g.
```lua
function FOO(NUMBER)
 return NUMBER==0 or type(NUMBER)=="NUMBER"
end
function BAR(NUMBER)
 NUMBER+=0
 return NUMBER==0 and type(NUMBER)=="NUMBER"
end
print(FOO"0") -- false
print(BAR"0") -- true
```
- Use when: calling a function with a single literal argument.
- Caveats: Numbers are passed in as string and may need to be converted. String-to-number conversion is fine for most cases, but currently has a rounding error with negative numbers (see http://www.lexaloffle.com/bbs/?tid=27597).
- Saves: 1 token per function call

## Assignment with Commas
Multiple variables can be declared at the same time in the format `var1,var2 = value1,value2`. These declarations can be chained indefinitely, removing the need for an `=` per variable assigned.
- Use when: assigning multiple variables at the same time.
- Caveats: If you reference one of the variables being assigned within the comma-separated list, it will evaluate to its value **before** assignment, regardless of order. e.g. the statements `x,y=1,2; x,y=3,x` will result in `x` having a value of `3`, and `y` having the value of `1`. You can mix-and-match different variable types here, but you can't mix-and-match `local` and global variables.
- Saves: 1 token per variable

## Replace Constant Variables with Literals
When you're coding, it's almost always better to store constants as variables; e.g. if your game has gravity, you might write `g=9.8` and reference `g` instead of writing `9.8` everywhere gravity needs to be applied. It's helpful while you're coding, but doesn't actually contribute to the final program, so once you've decided on a constant, you can replace all those variable references with the literal.
- Use when: you're prepared for commitment.
- Caveats: It's really annoying to change a constant manually after taking out the variable, so try to leave this one to the end if you can.
- Saves: 3 tokens per variable

## Actually Do Your Math
Similar to storing constants in variables, it's often easier to represent constant values in code using multiple literals, e.g. `1/3` and `1/21` are probably more user-friendly than `0.333...` and `0.0476...`. By replacing these with values calculated outside of the editor, you can save some tokens.

It's also helpful to remember to use algebra to simplify statements, e.g. `2*(variable/10)` might make sense when you first write it, but it's the same as `variable/5`.
- Use when: you're relying on the editor to do the math for you.
- Caveats: Again, it's harder to edit once you've made the change so leave it to the end if you can.
- Saves: depends

## Replace Table Elements with Separate Variables
It's typical to use tables to store the properties of an object which "belong" to that table; e.g. you might have a table `player` which has a table `player.position` which has the elements `player.position.x` and `player.position.y`. In some cases, this is just an organizational habit and you could just have `player_position_x` and `player_position_y` as completely unrelated variables and avoid the tokens needed for table access.
- Use when: you're treating a table as a mental model instead of a table.
- Caveats: This typically isn't useful if you're using OOP stuff; e.g. if your player and your enemies are all tables that have `position` and are handled in one loop, you probably won't save on tokens by taking one of them out of the system.
- Saves: 1 token per table access

## Replace Multiple Table Accesses with Local Variables
If you're repeatedly accessing a single table value, you can store the value in a local variable and reference that instead.
e.g. the following function:
```lua
function print_score(player)
 if player.score == 0 then print(player.score.." you lost")
 elseif player.score < 10 then print(player.score.." you lost, but not that bad")
 elseif player.score < 20 then print(player.score.." you won!")
 else print(player.score.." you won by a lot!") end
end
```
can be rewritten with 3 less tokens as:
```lua
function print_score(player)
 local score=player.score
 if score == 0 then print(score.." you lost")
 elseif score < 10 then print(score.." you lost, but not that bad")
 elseif score < 20 then print(score.." you won!")
 else print(score.." you won by a lot!") end
end
```
This is particularly useful if the access statement is more complex. `player.score` needs to be repeated 4 times before you break even, but `game.scene.player.score.current` only needs to be repeated once.

- Use when: you're accessing a table multiple times within a scope block
- Caveats: The table might not even be necessary in the first place; make sure to check if it's just a mental model.
- Saves: at least 1 token per table access, with an overhead of 4 tokens to create the local variable (and an additional 4 if you're modifying it and assigning it to the table at the end)
