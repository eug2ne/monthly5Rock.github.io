---
layout: post
title: (DevLog) 2. Setting up Project and Creating Basic Player Movement
date: 2024-04-14
categories: ["game", "game_dev", "dev_log", "demo", "kitd", "godot", "code"]
---

Sorry about the late update. I went on a 2 week trip to Japan and didn't have time to post.

After coming up with the basic idea of 'Kindle in the Night', I started working on the demo. So far, I have set up my project in Github and created basic movements for the player.

# Setting up the project in Godot

Since the game needed an efficient way to deal with mouse-click inputs, I thought using C++ to create a module to handle input events would help optimize the game and be useful when porting the game to consoles later.

And I can learn C++ in a fun way along the progress, right? So I setted up GDExtension for C++ binding. (You can check out the official guideline and video for set up below)

> [GDExtension Official Guideline](https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/gdextension_cpp_example.html)
>
> [GDExtension Set Up Tutorial Video](https://www.youtube.com/watch?v=8WSIMTJWCBk&t=2823s)

Turns out C++ is too difficult to start off writing practical code as a beginner, and GDExtension is not a good place to start. After many failed attempts that got me to no where, I changed my plan to **mainly using GDScript** for the demo, while studying C++ on the side.

# Creating basic player movement

And this is the progress I made so far.

First, I created a particle that releases on mouse clicks, and shows the location.

Next, I created the player and a basic tile map using the free asset linked below.
> [SunnyLand Asset Pack](https://ansimuz.itch.io/sunny-land-pixel-game-art)

But I soon realized the proportions of the tiles didn't match the proportions I needed. So I made a simple floor tile with the desired proportions.

I then created the code and animations for the basic player movements. This is the code I wrote.

```ts
extends CharacterBody2D

@onready var anim: AnimationPlayer = get_node("AnimationPlayer")
@onready var hit_area: CollisionShape2D = get_node("PlayerHitArea")
var target_position: Vector2
var GRAVITY = ProjectSettings.get_setting("physics/2d/default_gravity")
var SPEED = 150
const POS_BUFFER = 3
const ROLL_DISTANCE = 75
const JUMP_VELOCITY = -400

func _ready():
	# set current position to target_position
	target_position = position
	anim.play("idle")
	
func _input(event):
	# update target_position & set speed on mouse-click event
	if event.is_action_pressed("move"):
		target_position = get_global_mouse_position()
		SPEED = 150 # default speed
		if event.double_click:
			SPEED = 220 # fast speed
	if event.is_action_pressed("roll") and is_on_floor():
		var roll_dir = get_global_mouse_position().x - position.x
		target_position.x += (roll_dir/abs(roll_dir)) * ROLL_DISTANCE
		SPEED = 150 # default speed

func _physics_process(delta):
	# add gravity
	if not is_on_floor():
		velocity.y += GRAVITY * delta
	
	var directionX = target_position.x - position.x
	var directionY = target_position.y - position.y
		
	if abs(directionX) > POS_BUFFER and position.x != target_position.x:
		# flip player direction
		if directionX < 0:
			get_node("AnimatedSprite2D").flip_h = true
		else:
			get_node("AnimatedSprite2D").flip_h = false
		
		# move player to target position
		if Input.is_action_pressed("move"):
			velocity.x = (directionX/abs(directionX)) * SPEED
			anim.play("run")
		elif Input.is_action_pressed("roll"):
			velocity.x = (directionX/abs(directionX)) * SPEED
			anim.play("roll_start")
			# disable hit-area during roll
			hit_area.disabled = true
	else:
		velocity.x = 0 # reset velocity
		hit_area.disabled = false # reset hit-area
		anim.play("idle")
	
	if velocity.y > 0:
		anim.play("fall")
	
	move_and_slide()
```

The **_input() function sets the target position and the speed value when an mouse click event happens.**

If the `left button` is clicked, ***the target position is set to the position of the mouse click*** *and the player moves toward the target position.*

If its `double-clicked`, ***the speed is set faster and the player moves faster.***

The `right button` is the 'roll' button, so when its clicked, ***the target position is set to a roll distance to the left or right,*** *depending on the position of the mouse click. The player then 'rolls' to the target position,* which looks more like a crawl now because the asset didn't have an animation for roll.

And as you can see I disabled the hit-area during roll so the player can be invincible during rolls.

So now it looks like this.

![](../../assets/img/kitd/demo/screenshot-24-04-15.mp4)

Next time, I will tackle on adding ladders to climb up and down. I will also try adding a navigation feature to automatically find the closest path, just like in 'This War of Mine'.

`See you next time!`