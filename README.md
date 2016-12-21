# PICO-8 Token Optimizations
A few of these are pretty obvious, a few of them are not that obvious. Almost all of them make your code more unreadable, harder to edit, and would generally be considered "bad practice" in a less constrained system.

These are mostly the result of various people brainstorming optimizations on the PICO-8 discord server. Feel free to suggest changes, corrections, or other tricks!

## Negative Literals as Hexadecimals
The negative sign counts as a token, so instead of writing negative numbers as you would normally (e.g. `-10`) you can write them using large hexadecimal numbers (e.g. `0xFFF6`). This results in the same number. As a quick shorthand, `0xFFFF` is `-1`, `0xFFFE` is `-2`, etc.
- Use when: assigning, multiplying, or dividing negative literals.
- Caveats: Using hexadecimal numbers takes up more characters. Doesn't work with addition/subtraction because adding negative numbers can be replaced with a subtraction, and subtraction needs the `-` operator.
- Saves: 1 token per number

## Rely on Default Arguments
Functions can be called without passing every argument. Any specified arguments which aren't passed in are assigned a value of `nil` instead. Because of this, many of the PICO-8 API functions have arguments which can often be omitted without changing program behaviour. Some common examples include:
- `btn`/`btnp`: The second argument is an optional player ID (rarely needed for single-player games).
- `sfx`: The second argument is an optional channel, and the third argument is an optional offset.
- `music`: The second argument is an optional fade-length, and the third argument is an optional channel mask. If you want to play music from the beginning of the tracker, you can even call `music` without any arguments.

You can also take advantage of this behaviour in user-defined functions. e.g. say you have a function `foo` which is often (but not always) called with the argument `5`. If you change the function such that a `nil` argument is replaced with `5`, you can remove the argument in all the places you would pass `5` in. The following 22-token program
```lua
function foo(argument)
 --do a thing
end
foo(5)
foo(5)
foo(5)
foo(5)
foo(5)
foo(5)
```
becomes
```lua
function foo(argument)
 argument=argument or 5
 --do a thing
end
foo()
foo()
foo()
foo()
foo()
foo()
```
a 21-token program.

- Use when: you're passing arguments which don't change the behaviour of a function.
- Caveats: Arguments must be specified in order, so try to order your arguments such that they are specified in decreasing order of likelihood to be passed in. If the function only has one argument, the point below about calling functions with strings can probably save the same number of tokens without any overhead.
- Saves: 1 token per argument omitted, with an overhead of up to 5 tokens if you have to replace the `nil` argument with your own default value

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
can be rewritten with 3 fewer tokens as:
```lua
function print_score(player)
 local score=player.score
 if score == 0 then print(score.." you lost")
 elseif score < 10 then print(score.." you lost, but not that bad")
 elseif score < 20 then print(score.." you won!")
 else print(score.." you won by a lot!") end
end
```
This is particularly useful if the access statement is more complex. `player.score` needs to be repeated 4 times before you break even, but `game.scene.player[0].score.current` only needs to be repeated once.

This method has a bonus of being less CPU-intensive: locals can be read faster than table accesses.

- Use when: you're accessing a table multiple times within a scope block
- Caveats: The table might not even be necessary in the first place; make sure to check if it's just a mental model.
- Saves: n-1 tokens per table access, with an overhead of n+2 tokens to create the local variable (and an additional n+2 if you're modifying it and assigning it to the table at the end), where n is the number of tokens needed to access the variable

## Initialize Table Properties in Declaration
Tables are commonly initialized using `t={}`, and then followed by table property declarations (e.g. `t.x=0`). This is probably a result of a mental model which goes "Create my table, now set my table's X position to 0". You just wanted a table with an X position of 0 though, and didn't really need to separate it into two steps.
e.g. the following table + property declaration:
```lua
t = {}
t.one = 1
t.two = 2
t.three = 3
```
can be rewritten with 3 fewer tokens as:
```lua
t = {
  one = 1,
  two = 2,
  three = 3
}
```

- Use when: you're declaring tables and their properties at the same time
- Caveats: Similar to the comma assignment point above, you won't be able to reference values which are being assigned simulataneously. e.g. `t = {}; t.a = 1; t.b = t.a` will result in `t.b == 1`, but `t = {a = 1, b = a}` will result in `t.b == nil`.
- Saves: 1 token per property declared

## Prefer Property Access to Array Indexing
Accessing tables through array indexing (e.g. `table["key"]`) is a 3-token statement, whereas accessing a property (e.g. `table.key`) is a 2-token statement.

- Use when: accessing table properties with string literals
- Caveats: `table[key]` allows for a variable `key`, whereas property access is static.
- Saves: 1 token per access

## Vector Dimensions as Properties Instead of Array Elements
A common, but more complex example of the above point is the way you might handle PICO-8 vectors: you could create a table in the format `point={x,y}`, and then access the x and y values using `point[1]` and `point[2]`, or you could create a table in the format `point={x=x,y=y}` and access them using `point.x` and `point.y`. The former costs more tokens to access (3 vs. 2), but the latter costs more tokens to create (9 vs 5). You can reduce the creation cost by including a constructor function, e.g. `function vec(x,y) return {x=x,y=y} end`. This function has an overhead, but reduces the token cost of creating vectors to `vec(x,y)`, a 4-token statement. Table access is typically more common than table creation, so even without the constructor function the latter format will usually save tokens. 

- Use when: you need vectors
- Caveats: If you need to loop through the properties for whatever reason, your loop structure may have to change (e.g. you might need to replace `for i in all(point)` with `for k,v in pairs(point)`.
- Saves: 1 token per access. Without constructor function, costs 4 tokens per creation. With constructor function, saves additional 1 token per creation, with an overhead of 13 tokens.

*Note that vectors is simply a common example of this optimization; it can be applied to other objects as well*

## Parsing Data from String Literals
If you're working on a larger project, you might run into a situation where you've hit the token limit because you're trying to store large amounts of data. This could be enemy coordinates, lines of NPC dialogue, level configurations, item properties, etc.; whatever it is, chances are it's being stored in a big table somewhere in your project, taking up a bunch of tokens. Instead of storing data in tables, you can often save tokens by storing it in strings and parsing it into tables at runtime.

One of the added benefits of this method is that, depending on your data, you might actually be able to reduce your character count by moving the string data into unused portions of the cartridge memory (e.g. empty map space, empty sfx, etc.) and extract it at runtime.

The implementation of this can, of course, vary quite a bit from project to project. The following is a simple example of how one might convert a table of single-digit numbers into a string parsed at runtime: 
```lua
data={0,1,2,3,4,5,6,7,8,9,4,6,7,2,8,8,2,8,9,1,6,5,3,3,6,8,9,3,3,6,7,9,0,3,1,2,2,3,5,7,8,9,0}

for i in all(data) do
 print(i)
end
```
```lua
data_string="0,1,2,3,4,5,6,7,8,9,4,6,7,2,8,8,2,8,9,1,6,5,3,3,6,8,9,3,3,6,7,9,0,3,1,2,2,3,5,7,8,9,0"
data={}
while #data_string > 0 do
 local d=sub(data_string,1,1)
 if d!="," then
  add(data,d)
 end
 data_string=sub(data_string,2)
end

for i in all(data) do
 print(i)
end
```
Both programs produce the same result, but the first is 56 tokens and the second is only 44. Notably, each additional element in the first program's `data` table requires an additional token, but the second program stays the at same token count for an arbitrary length `data_string`.

*Note that this example is just for demonstration purposes, and not meant to show an optimal method for either storing or parsing data.*

- Use when: you're dealing with large amounts of similar data stored in tables
- Caveats: Remember that PICO-8's other limits (character, compressed, RAM) can be just as restrictive as the token limit; this technique is useful but it doesn't give you infinite storage. Also, strings aren't limited in length, but accessing data in a string with more characters than PICO-8 numbers can represent may require a bit of extra work (e.g. the `#` operator will overflow).
- Saves: depends; usually a whole lot of tokens, but with a fairly large overhead
