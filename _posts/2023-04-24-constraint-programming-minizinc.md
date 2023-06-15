---
layout: post
title:  "Constraint programming in Minizinc"
description: Solving the river crossing (and some other) problems in Minizinc
img:
date: 2023-04-25  +1430
---

For any language, the first round of learning it, is when we try out the examples in the docs or some book. But, we truly learn the language, when you have some problem in mind, that you want to solve, and try to code it up, ddg'ing left and right and reading the docs ten times over.

This blog is about Minizinc, a modelling language for linear and constraint programming that I have been playing around with.
First, it is absolutely necessary to read the minizinc docs page to understand the basic syntax, since this is not a traditinal programming language. There are a ton of examples such as task scheduling, stable marriage problem etc. 

Then, I tried some simple problems such as finding the maximum of an array and finally tried modelling the famous river crossing problem in Minizinc.

## Max of array
Program I want to write is very very simple - calculate the max of a set of numbers.

We need to use `1..n` notation to declare an array of size n. Decision variables are declared as `var`. 

### Attempt #1
```minizinc
int: n;
array[1..n] of int: numbers;

var int: max;

% maximization
constraint forall (i in numbers) (
    max >= i
);
   
solve maximize max;
```

The program obviously did not work in the first attempt. Here, I created a maximization constraint, so the output was `max = 2147483646` since I never enforced it to be a number in the array.

### Attempt #2
I need to add a constraint that the max is within the array. I could use some for atleast one constraint, which I was not able to find. So I ended up using indices instead of numbers in the array.

```minizinc
int: n;
array[1..n] of int: numbers;

var int: max_i;

% belongs to set
constraint max_i >= 1 /\ max_i <= n;

% maximization
constraint forall (i in 1..n) (
    numbers[max_i] >= numbers[i]
);
   
solve maximize max_i;
```

## 2 largest numbers
This was slightly more complicated. But using the `->` (from the official Minizinc stable marriage problem example) it is made easier.
```minizinc
int: n;
array[1..n] of int: numbers;

var int: max_i;
var int: max_j;

% belongs to set
constraint max_i >= 1 /\ max_i <= n;
constraint max_j >= 1 /\ max_j <= n;

% maximization
constraint forall (i in 1..n) (
    numbers[max_i] >= numbers[i]
);

constraint forall (j in 1..n) (
    numbers[max_j] < numbers[j] -> j = max_i
);

solve satisfy;
```

## River crossing problem in Minizinc
The river crossing problem is a famous riddle where a farmer has a wolf (don't know what farmers do with wolves), a goat and a cabbage and needs to transport them across a river. There is a single boat with space for exactly one item and only the farmer can row the boat. If the wolf is left alone with the goat or the goat is left alone with the cabbage, the former will eat the latter. With all these constraints, we need to find the minimum number of moves for the farmer to safely transport all his goods to the other side.

Given my recent Minizinc mood, I decided to try my hand at modelling and solving this problem using Minizinc. Basically we are going to specify all the constraints of this problem in Minizinc and are going to watch it give us the set of moves that the farmer has to make.

There is already a [blog post](https://sasnauskas.eu/solving-river-crossing-puzzles-with-minizinc/) on exactly this, which I did a quick glance over. I didn't really understand why step0 and step1 variables were defined and what the set type was.
Anyway since the goal is for me to play around with Minizinc, nothing beats trying to solve the problem from scratch.

I start by defining the 4 entities and the 2 locations that they can take:
```minizinc
enum Entity = {Farmer, Wolf, Goat, Cabbage};
enum Location = {LeftBank, RightBank};
```

I think it is possible to just let Location be a boolean type of true or false for the left and right banks respectively, but the program will probably look unnecesarily complicated and hard to understand.

Next, we define the actual decision variables, the values that we want to solve for and that should be filled in by the constraint solver.
For that, we need to decide how we want to model the problem. I am going to have an array of Entity X time steps and at each time step, we are going to have a variable for the location that each entity is going to be at.
```minizinc
array[Entity, 0..10] of var Location: positions; % For each entity, for each timestep, assign it a location
var 0..10: end;
```

Next, we need to add the constraints on these positions. These include the basic rules of movement - that the position can only change if you take a boat with the farmer, so it can't change randomly or illogically and the problem specific rules such as space for only one other entity in the boat and the different animal's food related rules.

We start with the most trivial ones representing the starting position and the ending positions.
```minizinc
% Starting position
constraint forall(e in Entity) (
  positions[e, 0] = LeftBank
);

% Ending position
constraint forall(e in Entity, t in 0..10 where t >= end) (
    positions[e, t] = RightBank
);
```

Next, we model the more complicated constraints regarding movement. I started with some giant hodgepodge of comparisons and conditions which unsurprisingly did not work. Then I decided to break it into the smallest, simple building blocks. This made each constraint more readable and it definitely helps a lot if we only focus on fixing one aspect of the problem at a time.

The way I worked was pick an aspect of the problem, describe it using English statements and converting them into constraints. One problem with this approach is we might end up with some redundant constraints this way. I also ended up with an extra unnecessary constraint, as I describe in the end of the post.

First, if you are not in the same bank as the farmer, your location should not change in the next move.
```minizinc
% Constraint 1 - Cannot move if not on farmer's bank
constraint 
forall(e in Entity, t in 0..end-1) (
    positions[e, t] != positions[Farmer, t] -> positions[e,t+1] = positions[e,t]
);
```

Second, you can only move with the farmer. They way this is modelled is that if you position has changed (old not equal to new), then your old location should have been the same as the farmer's old location and your new location must also be the same as the farmer's new location. 
```minizinc
% Constraint 2 - Can only move with farmer
constraint 
forall(e in Entity, t in 0..end-1) (
    positions[e, t+1] != positions[e, t] -> positions[e,t] == positions[Farmer, t] /\ positions[e,t+1] == positions[Farmer,t+1]
);
```

Finally, the farmer can transport a maximum of one entity in the boat.
```minizinc
% Only one entity can move apart from Farmer
constraint 
forall(t in 0..end-1) (
  sum(e in Entity where e != Farmer)( positions[e, t+1] != positions[e, t] ) <= 1
);
```

Lastly, we add the eating constraints.
```minizinc
% Goat and cabbage constraint
constraint
forall(t in 0..end-1) (
  positions[Farmer, t] != positions[Goat, t] -> positions[Goat, t] != positions[Cabbage, t]
);

% Goat and wolf constraint
constraint
forall(t in 0..end-1) (
  positions[Farmer, t] != positions[Wolf, t] -> positions[Wolf, t] != positions[Goat, t]
);
```

Then we end it with the objective function to minimize the number of steps.

```minizinc
solve minimize end;
```

Running this gives us the output steps:
```minizinc
positions = 
[|                 0:         1:         2:         3:         4:         5:         6:         7:         8:         9:        10: 
 |  Farmer: LeftBank, RightBank,  LeftBank, RightBank,  LeftBank, RightBank,  LeftBank, RightBank, RightBank, RightBank, RightBank
 |    Wolf: LeftBank,  LeftBank,  LeftBank, RightBank, RightBank, RightBank, RightBank, RightBank, RightBank, RightBank, RightBank
 |    Goat: LeftBank, RightBank, RightBank, RightBank,  LeftBank,  LeftBank,  LeftBank, RightBank, RightBank, RightBank, RightBank
 | Cabbage: LeftBank,  LeftBank,  LeftBank,  LeftBank,  LeftBank, RightBank, RightBank, RightBank, RightBank, RightBank, RightBank
 |];
end = 7;
----------
==========
```

I checked the blog post again after I was done. Now I was able to understand it better and it seems that I modelled it the same way as them. The blog uses a `predicate` method of specifying constraints that I did not know.
Moreover, it made me think about why I seemed to have more constraints and tried to see if something was redundant.
Found out that constraint 1 was not required and was captured in constraint 2 itself.

Also the blog has a nicer output, I currently just prefer to dump the intermediate variables in all their gory detail instead of putting in the effort to learn how to pretty print the output.


The final code is:

```minizinc
enum Entity = {Farmer, Wolf, Goat, Cabbage};
enum Location = {LeftBank, RightBank};

array[Entity, 0..10] of var Location: positions; % For each entity, for each timestep, assign it a location
var 0..10: end;

% =================================
% ========= Constraints ===========
% =================================

% Starting position
constraint forall(e in Entity) (
  positions[e, 0] = LeftBank
);

% Ending position
constraint forall(e in Entity, t in 0..10 where t >= end) (
    positions[e, t] = RightBank
);


% Cannot move if not on farmer's bank
%constraint 
%forall(e in Entity, t in 0..end-1) (
%    positions[e, t] != positions[Farmer, t] -> positions[e,t+1] = positions[e,t]
%);

% Can only move if on farmer's bank
constraint 
forall(e in Entity, t in 0..end-1) (
    positions[e, t+1] != positions[e, t] -> positions[e,t] == positions[Farmer, t] /\ positions[e,t+1] == positions[Farmer,t+1]
);

% Only one entity can move apart from Farmer
constraint 
forall(t in 0..end-1) (
  sum(e in Entity where e != Farmer)( positions[e, t+1] != positions[e, t] ) <= 1
);

% Goat and cabbage constraint
constraint
forall(t in 0..end-1) (
  positions[Farmer, t] != positions[Goat, t] -> positions[Goat, t] != positions[Cabbage, t]
);

% Goat and wolf constraint
constraint
forall(t in 0..end-1) (
  positions[Farmer, t] != positions[Wolf, t] -> positions[Wolf, t] != positions[Goat, t]
);
 
solve minimize end;
```

Even though I modelled the problem independantly, I ended up with the same model as the blog. This made me curious, is there another way of modelling this problem or not?
So, I decided to come up with something, even if it were super inefficient, just to get a different modelling of the problem.

## Another (although inefficient) version of river crossing 
This time, I start with defining the entities a little bit differently:
```minizinc
enum Entity = {Goat, Tiger, Cabbage, None};
```

Instead of the farmer, I have an entity called `None`.

I am going to model the decision making through an array (whose size is the number of timesteps, kind of similar to before), but this time, instead of deciding the location of each entity, I am going to decide which entity will move in this timestep.
```minizinc
array[1..10] of var Entity: movement;
var 1..10: end;
```

Modelling it this way has an advantage and a disadvantage. The advantage is that some of the constraints are implicit with this kind of model. Specifically, the constraint that either none or exactly one entity can travel with the farmer and the constraint that you can only move if you are on the same side of the farmer.
The disadvantage is that now I don't know explicitly know the location of each of the entities, and need to calculcate if to track it. The problem is that most of the other constraints depend on the location of the entity, so I need to define that as an easy to use function.

```minizinc
function var int: bank(int: time, array[int] of var Entity: movement, var Entity: e) =
  sum(t in 1..time) (movement[t] == e) mod 2;
  
predicate is_left_bank(int: time, array[int] of var Entity: movement, var Entity: e) =
  bank(time, movement, e) == 0;
  
predicate is_right_bank(int: time, array[int] of var Entity: movement, var Entity: e)=
  bank(time, movement, e) == 1;
```

We define the function `bank` which tracks the number of times each entity has moved (taken the boat). If it takes an odd number of moves, it remains on the left bank and if it takes an even number of moves, it ends up on the right bank.
Thus, we can get the location of each entity at each timestamp. However, it is quite inefficient, since it needs to traverse the complete array for every entity, every time it is called.
But since we have thrown efficiency to the wind, let's continue down this path.

Next, we add the constraints:

First is the ending constraint, the problem is solved if all the entities are on the right bank. This was quite simple in the previous model, but now we need to calculate the location to know which bank each entity is on. Note that we could not use the predicate I had defined earlier. This is because of a function signature mismatch - here the time t is of type var int and the entity is not a var. However, as we will see later, the other constraints need a different signature and since those are required more often, I have used that signature.

```minizinc
% Ending point - all except None must be on right bank
constraint
  forall(e in Entity where e != None) (
    sum(t in 1..end) (movement[t] == e) mod 2 == 1
  );
```

The next constraint is something that we brought upon ourselves with this kind of model. In an odd turn/move, you can only move if you are on the left bank. Similarly, in an even turn, you can only move if you are on the right bank. This only works if we imagine the farmer to be continuously moving left and right, back and forth. While the original problem does not state it this way, it does not make sense for the farmer to linger at a bank for more than one turn. If it does, in some time based version of the problem, then this model will no longer hold.
```minizinc
% Can only move from the side you are on
constraint
  forall(t in 2..10 where t <= end) (
    if movement[t] != None then
      if t mod 2 == 1 then is_left_bank(t-1, movement, movement[t])
      else is_right_bank(t-1, movement, movement[t])
      endif
    else true
    endif
  );
```

Then we add the eating constraints:
```minizinc
% Goat and cabbage cannot stay together
constraint
  forall(time in 2..10 where time <= end-1) (
    sum(t in 1..time-1) (movement[t] == Goat) mod 2 = sum(t in 1..time-1) (movement[t] == Cabbage) mod 2 -> movement[time] in ['Goat', 'Cabbage']
);

% Goat and tiger cannot stay together
constraint
  forall(time in 2..10 where time <= end-1) (
    sum(t in 1..time-1) (movement[t] == Goat) mod 2 = sum(t in 1..time-1) (movement[t] == Tiger) mod 2 -> movement[time] in ['Goat', 'Tiger']
);
```

Finally, the objective is:
```minizinc
solve minimize end;
```

When I run this, I actually do get the correct answer almost immediately (within a second). However, the program takes around 18-19 seconds to end, which I assume is for it to check if a more optimal solution exists. So, clearly the code is not the best in terms of efficiency. I will need to see if there are some changes that can be done for making this better, but for now I will end with this inefficient code itself.

The full code is below:
```minizinc
enum Entity = {Goat, Tiger, Cabbage, None};

array[1..10] of var Entity: movement;
var 1..10: end;

function var int: bank(int: time, array[int] of var Entity: movement, var Entity: e) =
  sum(t in 1..time) (movement[t] == e) mod 2;
  
predicate is_left_bank(int: time, array[int] of var Entity: movement, var Entity: e) =
  bank(time, movement, e) == 0;
  
predicate is_right_bank(int: time, array[int] of var Entity: movement, var Entity: e)=
  bank(time, movement, e) == 1;

% Ending point - all except None must be on right bank
constraint
  forall(e in Entity where e != None) (
    sum(t in 1..end) (movement[t] == e) mod 2 == 1
  );

% Can only move from the side you are on
constraint
  forall(t in 2..10 where t <= end) (
    if movement[t] != None then
      if t mod 2 == 1 then is_left_bank(t-1, movement, movement[t])
      else is_right_bank(t-1, movement, movement[t])
      endif
    else true
    endif
  ); 

% Goat and cabbage cannot stay together
constraint
  forall(time in 2..10 where time <= end-1) (
    sum(t in 1..time-1) (movement[t] == Goat) mod 2 = sum(t in 1..time-1) (movement[t] == Cabbage) mod 2 -> movement[time] in ['Goat', 'Cabbage']
);

% Goat and tiger cannot stay together
constraint
  forall(time in 2..10 where time <= end-1) (
    sum(t in 1..time-1) (movement[t] == Goat) mod 2 = sum(t in 1..time-1) (movement[t] == Tiger) mod 2 -> movement[time] in ['Goat', 'Tiger']
);

solve minimize end;
```

## Other misc. minizinc stuff
The `minizinc` command does 3 things - compiles the program, runs it with a solver and returns the output in the format you need.

If we run using the `-c` option, we get the flatzinc file.
