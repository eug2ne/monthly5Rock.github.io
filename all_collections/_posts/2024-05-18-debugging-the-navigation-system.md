---
layout: post
title: (DevLog) 4. Debugging the Navigation System
date: 2024-05-18
categories: ["game", "game_dev", "dev_log", "demo", "kitd", "godot", "code"]
---

After the last devlog, I spent the time debugging my navigation system. There was always a new bug right after I debug something. Some bugs were easy to fix, but others took more time. Honestly, it was a pretty stressful job.

Eventually I managed to pull out a decent system. This is how the whole thing went.

# Adding Physics to the Player Movement
I got the path from the AStarGrid2D node. The next thing to do was make the player follow the path.

The path is **an array of Vector2i that are the coordinates of the tile map to follow from the current position.** So in the _physics_process() I wrote a code that checks the current coordinate and the next coordinate in the path array.

```ts
var current_tile_coord = tile_map.local_to_map(global_position)
	if current_tile_coord == current_path.front():
		current_path.pop_front()
		
		if current_path.is_empty():
			return
```

If the current coordinate matches the next coordinate, it means the player has already entered the next tile. So the path deletes the next coordinate by using the .pop_front() method. If the path is empty after deleting the next coordinate, the _physics_process() returns to deal with additional adjustments when the path ends.

After getting the next coordinate, the code calculates the vector of the player's movement.

```ts
var next_direction = current_path.front() - current_tile_coord

# flip player direction
if next_direction.x < 0:
	get_node("AnimatedSprite2D").flip_h = true
else:
	get_node("AnimatedSprite2D").flip_h = false

# apply player animation
anim.play("run")
```

Then it flips the player's sprite according to the direction, and applies the player animation.

## Making the Player Jump
The first most obvious problem was that the player **couldn't move diagonally upwards.** It just hanged around the edges and got stuck.

![](../../assets/img/kitd/demo/screenshot-24-05-18-jump-stuck.mp4)

I immediately found out the reason. When the player first moves diagonally, the initial vector is (+-1, -1). However, when it enters the next tile, the vector turns to (+-1, 0) which means gravity is applied and the player falls back to the previous tile. ==So the player repeats entering the next tile and falling back to the previous tile.==

To fix this, **the vertical velocity should be maintained during the (+-1, 0) phase.** To distinguish it from the normal horizontal moves, I created a dictionary that saves the player's state.
```ts
var STATE: Dictionary = {
	"roll": false,
	"move_up": false,
	"move_down": false
}
```

When STATE.move_up is true and the vector is (+-1, 0), the player is in mid-air of the jump. I made the jump velocity to gradually decline when the player's in mid-air, creating a smooth curve in the player's jump.
```ts
if next_direction.y < 0:
	# apply jump velocity
	velocity.y = JUMP_VELOCITY
	STATE.move_up = true # set state to move_up
elif next_direction.y == 0 and STATE.move_up:
	# jump mid-air >> gradually decline jump velocity
	velocity.y -= JUMP_VELOCITY / 4
	
else:
	# apply gravity
	velocity.y = GRAVITY
	STATE.move_up = false # set state to default
```

## Pinpointing the Player's Position
You may have noticed that the player's final position doesn't really match the mouse-click position in the previous video.

This is because the player stops right after it enters the final tile. So it stops when it's halfway through the final tile regardless of the actual position of the mouse-click.

To make the extra mile, I added a code that **calculates the direction toward the mouse-click position when it enters the final tile, and sets the player's velocity accordingly.**

```ts
if current_path.is_empty():
		if not is_on_floor():
			# apply gravity
			velocity.y = GRAVITY
			move_and_slide()
		
		if abs(target_position.x - global_position.x) > 3:
			# adjust player toward target-position
			directionX = (target_position.x - global_position.x) / abs(target_position.x - global_position.x)
			velocity.x = directionX * SPEED
			
			move_and_slide()
		
		else:
			tile_map.update_astar(false, global_position) # reset tile_map navigation + update map icon
			velocity = Vector2(0,0) # reset velocity
			hit_area.disabled = false # reset hit-area
			anim.play("idle")
			
		return
```

## Adding a Timer to the Roll
To prevent the player from getting stuck on a wall during roll, I also added a timer.
```ts
if event.is_action_pressed("ui_mouseclick_right"):
	# set roll speed + direction
	directionX = (get_global_mouse_position().x - global_position.x) / abs(get_global_mouse_position().x - global_position.x)
	target_position.x += directionX * ROLL_DISTANCE
	SPEED = 150 # default speed
	hit_area.disabled = true # disable hit-area during roll
	
	# flip player direction
	if directionX < 0:
		get_node("AnimatedSprite2D").flip_h = true
	else:
		get_node("AnimatedSprite2D").flip_h = false
	
	# start roll
	STATE.roll = true
	anim.play("roll_start")
	timer.start(0.5)
```

Since the roll doesn't require a path, I used the STATE dictionary to control the action.

## Smooth Falling
The next thing I noticed was that the player would "extremely" slow down when they move down from an edge.

![](../../assets/img/kitd/demo/screenshot-24-05-28-slow-fall.mp4)

The reason was similar to the jump problem but with the horizontal velocity; the initial vector is (+-1, 1), then when the player enters the next tile, the vector turns to (0, 1). **With the horizontal velocity turning to 0, player falls off the edge "extremely" slowly.**

Thus a similar solution was applied to the horizontal velocity.

```ts
# adjust velocity.x
if next_direction.y > 0 and next_direction.x != 0:
	velocity.x = next_direction.x * SPEED
	STATE.move_down = true
elif next_direction.y > 0 and STATE.move_down:
	# down fall >> gradually decline velocity.x
	if velocity.x == 0:
		# velocity.x reach 0
		pass
	elif velocity.x > 0:
		velocity.x -= SPEED / 8
	else:
		velocity.x += SPEED / 8


else:
		velocity.x = next_direction.x * SPEED
		STATE.move_up = false # set state to default
```

When STATE.move_down is true and the vector is (0, 1), the player is falling off. For the fall trajecotory to be straight, *the direction should be maintained and not be reversed mid-air.* So using the If/Elif statement, I **made the absolute value of the horizontal velocity to decrease but stays 0 when it becomes zero.**

## Detecting the Edge
This one was slightly unnecessary, but I wanted it personally so I made it.

To prevent the player from falling off edges, and also to later add cool slide animations, I made the player to detect edges.

When the player enters the last tile of the path, it calls the is_on_edge() function.

```ts
func is_on_edge():
	# detect player on edge
	var player_coord = tile_map.local_to_map(global_position)
	var tile_coord = Vector2i(player_coord.x, player_coord.y+1)
	var is_edge = false
	if tile_map.get_cell_tile_data(0, tile_coord):
		is_edge = tile_map.get_cell_tile_data(0, tile_coord).get_custom_data("edge")
	var tile_pos = tile_map.map_to_local(tile_coord)
	
	if is_edge:
		var e_direction = tile_map.get_cell_tile_data(0, tile_coord).get_custom_data("e_direction")
		if e_direction == 1:
			# player on edge (left)
			return tile_pos.x - global_position.x > 5
		else:
			# player on edge (right)
			return tile_pos.x - global_position.x < -5
	
	return false
```

The function checks
> 1. If the last tile is an edge tile + the edge direction
> 2. If the player is moving off the center of the tile for a certain pixels

If the is_on_edge() returns true and the player is still moving toward the mouse-click position, **the horizontal velocity is slowed and reversed.** The target position which was originally the mouse-click position is also updated to fit inside the final tile.

```ts
if current_path.is_empty():		
	if STATE.roll or abs(target_position.x - global_position.x) > 3:
		# (player roll) xor (adjust player toward target-position)
		directionX = (target_position.x - global_position.x) / abs(target_position.x - global_position.x)
		
		if is_on_edge():
			velocity.x = -1 * directionX * SPEED / 2
			# update target_position
			var player_coord = tile_map.local_to_map(global_position)
			var tile_pos = tile_map.map_to_local(player_coord)
			target_position.x = tile_pos.x + directionX * 4
			
			# TODO: add slide animation
		else:
			velocity.x = directionX * SPEED
		
		move_and_slide()
```

# Advanced Navigation
## Getting More Accurate Target Positions
This is the previous code I used to convert the mouse-click position to the position of the closest navigation tile. Do you see the problem?
```ts
# convert target_pos to closest navigation-tile position

var tile_navigatable = get_cell_tile_data(1, target_coord)

while !tile_navigatable:
	if icon_click:
		if current_coord.y > target_coord.y:
			# player move up
			target_coord.y -= 1
		else:
			# player move down
			target_coord.y += 1
	
	else:
		target_coord.y += 1
	
	tile_navigatable = get_cell_tile_data(1, target_coord)
```

The problem is it only searches the tile *directly below the clicked tile.* Most of the time it's fine, but in some cases, the result could drastically change. Like when you think you clicked on a tile that is hanging on an edge and the player moves to the tile a whole level below.

![](../../assets/img/kitd/demo/screenshot-24-05-18-navigation-tile-search.mp4)

I **expanded the searching scope to include the bottom right and left tiles, and loop the search until one of the three tiles are a navigation tile.**

```ts
func get_target_coord(current_coord, click_coord):
	# convert target_coord to naivgation-tile coord
	if get_cell_tile_data(0, click_coord).get_collision_polygons_count(0) == 1:
		# click on background tile
		click_coord.y -= 1
		return click_coord
	
	
	# check icon click
	var icon = check_icon_click()
	var search_tiles = [
		click_coord,
		Vector2i(click_coord.x-1, click_coord.y),
		Vector2i(click_coord.x+1, click_coord.y)
	]
	# search closest navigation-tile
	var tile_navigatable = search_tiles.any(func(coord): return get_cell_tile_data(1, coord))
	while !tile_navigatable:
		if icon:
			if current_coord.y > click_coord.y:
				# player move up
				click_coord.y -= 1
			else:
				# player move down
				click_coord.y += 1
		else:
			click_coord.y += 1
		
		search_tiles = [
			click_coord,
			Vector2i(click_coord.x-1, click_coord.y),
			Vector2i(click_coord.x+1, click_coord.y)
		] # update search-tiles
		tile_navigatable = search_tiles.any(func(coord): return get_cell_tile_data(1, coord))
		
	search_tiles = search_tiles.filter(func(coord): return get_cell_tile_data(1, coord))
	click_coord = search_tiles[0]
	
	if icon:
		if current_coord.y > click_coord.y:
			# player move up
			click_coord += icon.destination_up
		else:
			# player move down
			click_coord += icon.destination_down
	
	return click_coord
```

The function then returns the first coordinate of the detected navigation tiles. The coordinate of the tile right below the mouse-click position is the first item of the search_tiles array, so *the search is set to prioritize it first.*

## Getting More Accurate Paths
Lastly, I updated the path finding function to check every possibility.
```ts
func _get_current_path(current_pos, click_pos) -> Array[Vector2i]:
	var current_coord = local_to_map(current_pos)
	var click_coord = local_to_map(click_pos)
	
	# check icon click
	var icon = check_icon_click()
	# convert click_pos to closest navigation-tile position
	var target_coord = get_target_coord(current_coord, click_coord)
	
	if target_coord.y != current_coord.y:
		# update navigation map
		update_astar(true, current_coord)
	
	
	var nav_pos = map_to_local(target_coord)
	var path = astar.get_id_path(
			local_to_map(current_pos),
			local_to_map(nav_pos)
		).slice(1)
	if path.is_empty():
		# update navigation map
		update_astar(true, current_coord)
		path = astar.get_id_path(
			local_to_map(current_pos),
			local_to_map(nav_pos)
		).slice(1)
		
	return path
```

The update_astar() funciton updates the astar grid to either include the navigatable tiles or exclude them. 
```ts
func update_astar(navigatable: bool, player_pos: Vector2):
	var tilemap_size = get_used_rect().end - get_used_rect().position
	for x in tilemap_size.x:
		for y in tilemap_size.y:
			var coord = Vector2i(x,y)
			if navigatable:
				# enable path-finding on navigatable cells
				if get_cell_tile_data(2, coord):
					astar.set_point_solid(coord, false)
			
			else:
				# disable path-finding on navigatable cells
				if get_cell_tile_data(2, coord) and not get_cell_tile_data(1, coord):
					# disable navigatable cells exclusively
					astar.set_point_solid(coord)
				
				# switch icon image according to player_pos
				get_tree().call_group("Icons", "switch_image", player_pos)
```
If the initial path comes out empty, the update_astar() function includes the navigatable tiles in the astar grid, and a new path is calculated.

And this is the final product.

![](../../assets/img/kitd/demo/screenshot-24-05-18-final.mp4)

---
There are some times the player's movement seem "off" for a reason I don't know. It's probably because I need to conduct a proper QA, but that would take a long time. For now, the system is good enough for a demo and I have a lot of other features to make.

I also noticed that the videos in my post don't show up in Chrome. It seems like a common problem with Github pages in Chrome and Firefox in general. Unfortunately, I haven't found a fix yet. I will try my best to avoid using videos in the post for the meantime, but sometimes using only images and code snippets alone has a limit.

The videos work great in Safari and Orion, so if you have trouble viewing videos, I recommend changing browsers.

The next thing to do is to `add items and new interactions`, which also means *new animations.* So yes, the next devlog would be about creating a new spritesheet for the player and animating it.

`See you next time!`