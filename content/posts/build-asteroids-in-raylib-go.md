+++
date = '2025-10-09T05:33:54+01:00'
draft = false
title = "Build an Asteroids Game with Raylib-go"
description = "Build a full-featured Asteroids game with ship controls, shooting mechanics, collision detection, and asteroid physics using Go and raylib-go"
featuredImage = '/images/asteroids/featured.png'
categories = ["Game Development", "Programming"]
tags = ["go", "raylib", "game-development", "asteroids", "collision-detection", "physics", "graphics-programming", "tutorial"]
keywords = ["asteroids game", "raylib go tutorial", "game development", "collision detection", "go programming", "2d game development", "space shooter game"]
+++

In this tutorial, you’ll learn how to build an Asteroids game using Raylib-go, a lightweight library for game development.

By the end, we will have a complete game with player movement controlled by the keyboard, shooting, collisions, and win/lose states and all in Go.

## Demo

![Demo](/images/asteroids/demo.gif)

## Setting up your Go project

First, we need to set up our project and pull Raylib-go.

```go
mkdir go-asteroids
go mod init asteroids
go get -v -u github.com/gen2brain/raylib-go/raylib
```


### Setting up the Game

We start by creating basic game loop functions to initialise the resources, update the state, draw it to the screen and deinitialise the resources.

```go
const (
	screenWidth  = 800
	screenHeight = 400
)

func init() {
	//Builtin go function which runs before main()

	// Setup the raylib window
	rl.InitWindow(screenWidth, screenHeight, "Asteroids")
	rl.SetTargetFPS(60)
}

func draw() {
	rl.BeginDrawing()

	// Set the background to black
	rl.ClearBackground(rl.Black)

	// Draw the score to the screen
	rl.DrawText("Score 0", 10, 10, 20, rl.Gray)

	rl.EndDrawing()

}

func update() {
	//TODO:  Update the state

}

func deinit() {
	rl.CloseWindow()
}

func main() {
	// When the main function ends, call the deinit() function
	defer deinit()

	// Continue the loop until the window is closed or ESC is pressed
	for !rl.WindowShouldClose() {
		draw()
		update()
	}
}
```

First, we set up a `main()` function, which starts by deferring the `deinit()` function. This will ensure that the `deinit()` function runs at the end of the main() function.We use an [init()](https://go.dev/doc/effective_go#init) function, which runs **before** the main() function. These two functions allow us to set up resources in memory and tear them down when we no longer need them.

In the `init()` function, we set up the Raylib window and set the FPS.

We create a `draw()` function to use the new RayLib window to draw to the screen. We start the drawing with `BeginDrawing` and finish it with `EndDrawing`. We define our draw methods between those two calls. We set up an `update()` function, which is called each frame. We will fill that out later.

We simply clear the background with black and then draw a score in the top left corner.

In the `main()` function, we are called `WindowShouldClose`, which is a function that allows us to run our game loop until the Raylib window is closed or the ESC key is pressed.


Now, we can run the code:

```bash
go run main.go
```

What we should see is a black window with the score in the top left hand corner.


![Inital status with score](/images/asteroids/inital-score.png)

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/610fad31ad7f3a87135b5d37443552284452bfbb/go-asteroids/main.go)

## Drawing a background

It is a great start, though, we want to make this start look like our game. To start, we can add the space background.

We create a new `resources/` directory in our project and download the [space_background.png](https://github.com/timlittle/blog-code/blob/main/go-asteroids/resources/space_background.png) into it.

We load the image from a file and display it as our background:

```go
var (
	texBackground rl.Texture2D
)
```

We start by declaring some global state, with the background text. Next, we need to load the image:

```go
func init() {
	//Builtin go function which runs before main()

	// Setup the raylib window
	rl.InitWindow(screenWidth, screenHeight, "Asteroids")
	rl.SetTargetFPS(60)

	// Load textures
	texBackground = rl.LoadTexture("resources/space_background.png")
}
```

We load the image from disk using the `LoadTextures` call and assign it to the `texBackground` variable.

We need to make sure we are unloading it after the game, so we can do that in the `deinit()` function.

```go
func deinit() {
	rl.CloseWindow()

	// Unload textures when the game closes
	rl.UnloadTexture(texBackground)
}
```

To do that, we run the `UnloadTexture` call.

Now, we can draw the background to the screen:

```go
func draw() {
	rl.BeginDrawing()

	// Set the background to a nebula
	bgSource := rl.Rectangle{X: 0, Y: 0, Width: float32(texBackground.Width), Height: float32(texBackground.Height)}
	bgDest := rl.Rectangle{X: 0, Y: 0, Width: screenWidth, Height: screenHeight}
	rl.DrawTexturePro(texBackground, bgSource, bgDest, rl.Vector2{X: 0, Y: 0}, 0, rl.White)

	// Draw the score to the screen
...
```

We have a bit of an issue, the image is 1536x1024px, when our window size is only 800x400px. To scale the image to our window, we need to use the `DrawTexturePro` call.
This call takes the texture we want to draw, a rectangle within that texture and a destination texture on our window to draw it to. It also takes an Origin point, a Rotation and a Color, however, we are not using them.

For the `bgSource`, which is used to select the part of the image to draw, we create a rectangle to select the whole image. For the `bgDest`, we select the whole screen in Raylib. We then call the `DrawTexturePro` to draw the image to the screen.

Let's check to see that our background is being shown:

```bash
go run main.go
```

Which results in:

![Window with background](/images/asteroids/with-background.png)


Beautiful, now it looks ready to start building our game on top of.

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/6a97ee89336f2e9a646635814406af85abcb697e/go-asteroids/main.go)


## Create and draw the player

Next, we can set up the player for the game. This will be a ship that flies around and shoots at the asteroids.

For this, I am going to use [Kenney - Simple Space](https://kenney.nl/assets/simple-space) assets. There are some amazingly talented game artists around, and many of them provide assets for free.

We can download the assets and put them into our `/resources` directory.

Now, we load them into our game. We start by adding another texture to our game state:

```diff
var (
+      texTiles      rl.Texture2D
       texBackground rl.Texture2D
)
```

Then we need to load it in our `init()` function:

```diff
        // Load textures
+       texTiles = rl.LoadTexture("resources/tilesheet.png")
        texBackground = rl.LoadTexture("resources/space_background.png")
```

Then unload it in our `deinit()` function:

```diff
        // Unload textures when the game closes
        rl.UnloadTexture(texBackground)
+       rl.UnloadTexture(texTiles)
 }
```

Now that we have the texture loaded into memory, we can start to use it. However, as you may have spotted, the texture is a spritemap, made up of an 8x6 grid of sprites. We want to use one of them for our ship.

Luckily, we already saw a function that would allow us to select a section of the texture and draw it to the screen. We can use the `DrawTexturePro` to take a smaller section of the original image and then draw that for us.

Let's start by defining our source selection rectangle. We will also create a `tileSize` to track how large we want the image.

In the `consts`, we add the `tileSize`:

```diff
const (
	screenWidth  = 800
	screenHeight = 400
+   tileSize     = 64
)
```

Now, we define our source selection rectangle for the ship:
```diff
var (
	texTiles      rl.Texture2D
	texBackground rl.Texture2D
+	spriteRec     rl.Rectangle
)
```

Then, in the `init()` function, we can define the sprite source:
```diff
	texTiles = rl.LoadTexture("resources/tilesheet.png")
	texBackground = rl.LoadTexture("resources/space_background.png")

+	spriteRec = rl.Rectangle{X: tileSize * 0, Y: tileSize * 2, Width: tileSize, Height: tileSize}
}
```

We select the ship on the 3rd row (Index 2) on the 1st position (Index 0). 

As the tiles on the map are `tileSize` wide and high. We use that variable to calculate where in the tilemap we want to select. You can see from the image below how we have picked the ship we want.

![Tilesheet indexed](/images/asteroids/tilemap_index.png)

Now we have our selection, we need to draw it on the screen. To make these easier, we create a `Player` struct to hold all the player related behaviours.

```go
type Player struct {
	position     rl.Vector2
	speed        rl.Vector2
	size         rl.Vector2
	acceleration float32
	rotation     float32
	isBoosting   bool // We will use this in the next step
}
```

We create the `Player` structure with Vectors for the `position`, `speed` and `size`. We have some floats for `acceleration` and `rotation`, which we use to move the sprite. Finally, we have a boolean for boosting.

Now we can draw the sprite to the screen:

```
func (p *Player) Draw() {
	destTexture := rl.Rectangle{X: p.position.X, Y: p.position.Y, Width: p.size.X, Height: p.size.Y}
	rl.DrawTexturePro(
		texTiles,
		spriteRec,
		destTexture,
		rl.Vector2{X: p.size.X / 2, Y: p.size.Y / 2},
		p.rotation,
		rl.White,
	)
}
```

We define a `destTexture`, similar to the way we did the background; however, instead of covering the whole screen, we want the sprite to appear in the centre of the screen.

We need to create the new player when we start the game. For this, we create a variable in the state and a little helper function to do all the initialisation:
```diff
var (
	texTiles      rl.Texture2D
	texBackground rl.Texture2D
	spriteRec     rl.Rectangle
+	player        Player
)
```

New `initGame()` function:
```go
func initGame() {
	player = Player{
		position:     rl.Vector2{X: 400, Y: 200},
		speed:        rl.Vector2{X: 0.0, Y: 0.0},
		size:         rl.Vector2{X: tileSize, Y: tileSize},
		rotation:     0.0,
		acceleration: 0.0,
		isBoosting:   false,
	}
}
```

We need to add the new initialisation of the game to the `init()` function:

```diff
...
	texBackground = rl.LoadTexture("resources/space_background.png")

	spriteRec = rl.Rectangle{X: tileSize * 0, Y: tileSize * 2, Width: tileSize, Height: tileSize}

+	initGame()
}

```

Now we have a `player` object, we need to add it to the `draw()` in the main game loop:

```diff
...
	rl.DrawTexturePro(texBackground, bgSource, bgDest, rl.Vector2{X: 0, Y: 0}, 0, rl.White)

+	//Draw the player
+	player.Draw()

	// Draw the score to the screen
	rl.DrawText("Score 0", 10, 10, 20, rl.Gray)
...
```

Looks like that is all we need to draw the player, let's give it a shot and see if it works:

```
go run .
```

We should see our background image, and our new sprite drawn into the window in the middle of the screen:

![Player drawn](/images/asteroids/player_drawn.png)

Great!

We are successfully using the tilemap and drawing a sprite to the screen.

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/801ebfc08e5eeb7468be3f3ee9e2d680277f6b29/go-asteroids/main.go)


## Implementing player movement

Next, we start to give the game some life, with some movement. 

We want to move the player using the keyboard. So we will need to set up some code to manage the keyboard inputs and manipulate our sprite accordingly.


Let's start by creating some constants for the speed of the player and the rotation speed. This allows us to tweak it later.

```diff
const (
	screenWidth   = 800
	screenHeight  = 400
	tileSize      = 64
+	rotationSpeed = 2.0
+	playerSpeed   = 6.0
)
```

Now we have our variables, we can create an `Update()` method on our **Player** struct to move the player when the keyboard is used:

```go
func (p *Player) Update() {

	// Rotate the player with the arrow keys
	if rl.IsKeyDown(rl.KeyLeft) {
		player.rotation -= rotationSpeed
	}
	if rl.IsKeyDown(rl.KeyRight) {
		player.rotation += rotationSpeed
	}

	// Accelerate the player with up
	if rl.IsKeyDown(rl.KeyUp) {
		if player.acceleration < 0.9 {
			player.acceleration += 0.1
		}
	}
	// Decellerate the player with down
	if rl.IsKeyDown(rl.KeyDown) {
		if player.acceleration > 0 {
			player.acceleration -= 0.05
		}
		if player.acceleration < 0 {
			player.acceleration = 0

		}
	}

	// Get the direction the sprite is pointing
	direction := getDirectionVector(player.rotation)

	// Start to move to the direction
	player.speed = rl.Vector2Scale(direction, playerSpeed)

	// Accelerate in that direction
	player.position.X += player.speed.X * player.acceleration
	player.position.Y -= player.speed.Y * player.acceleration

	// To void losing our ship, we wrap around the screen
	wrapPosition(&p.position, tileSize)
}
```

We start by capturing the `KeyLeft` and `KeyRight` events, then rotate the ship based on the direction. We capture `KeyUp` and `KeyDown` for acceleration.

To make it a little more realistic, the ship has no friction (Just like space!), so we need to actively decelerate to slow down.

We use the `getDirectionVector` to translate our **rotation** into a vector, and we use that vector to change the direction of the ship with the `speed` attribute. We then add the `acceleration` and the `speed` to the position to move the ship in that direction.

Finally, we use the `wrapPosition` function to wrap the position so our ship is not lost in space as soon as we leave the screen.

We need to create our `getDirectionVector()` function:

```go
func getDirectionVector(rotation float32) rl.Vector2 {
	// Convert the rotation to radians
	radians := float64(rotation) * rl.Deg2rad

	// Return the vector of the direction we are pointing at
	return rl.Vector2{
		X: float32(math.Sin(radians)),
		Y: float32(math.Cos(radians)),
	}
}
```

And our `wrapPostion()` function:

```go
func wrapPosition(pos *rl.Vector2, objectSize float32) {
	// If we go off the left side of the screen
	if pos.X > screenWidth+objectSize {
		pos.X = -objectSize
	}
	// If we go off the right side of the screen
	if pos.X < -objectSize {
		pos.X = screenWidth + objectSize
	}
	// If we go off the bottom of the screen
	if pos.Y > screenHeight+objectSize {
		pos.Y = -objectSize
	}
	// If we go off the top of the screen
	if pos.Y < -objectSize {
		pos.Y = screenHeight + objectSize
	}
}
```

Finally, we can replace our `Update()` method with the main game loop to move the ship each frame:
```go
func update() {
	player.Update()
}
```

Let's run `go run main.go` to see if we can move the ship.


![Ship movement](/images/asteroids/ship_movement.gif)

Amazing!

Our player is moving based on the keypresses from the arrow keys.

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/ca4deb4d8ad394ce3f3fded0e232b75fe0e5c48b/go-asteroids/main.go)

### Adding a booster

To give the game a little moe realism, let's add a boost to the little ship, so it looks like that is how the ship is moving.

We already have the `isBoosting` attribute on the **Player** struct. Let's use that. 

We will select another sprite from the tilemap and draw it only when we are boosting.

```diff
var (
	texTiles      rl.Texture2D
	texBackground rl.Texture2D
	spriteRec     rl.Rectangle
+	boostRec      rl.Rectangle
	player        Player
)
```

We create a new selection rectangle from the tilemap in the global game state. 

```diff
        texBackground = rl.LoadTexture("resources/space_background.png")

+       // Sprites for the ship and it boost
        spriteRec = rl.Rectangle{X: tileSize * 0, Y: tileSize * 2, Width: tileSize, Height: tileSize}
+       boostRec = rl.Rectangle{X: tileSize * 7, Y: tileSize * 5, Width: tileSize, Height: tileSize}

        initGame()
```

We then select the tile we want to use. We want to use the yellow thrust pattern, which makes it look like the ship is activating its engine.

```diff
 func (p *Player) Draw() {
        destTexture := rl.Rectangle{X: p.position.X, Y: p.position.Y, Width: p.size.X, Height: p.size.Y}
+       if p.isBoosting {
+               rl.DrawTexturePro(
+                       texTiles,
+                       boostRec,
+                       destTexture,
+                       rl.Vector2{X: p.size.X / 2, Y: p.size.Y/2 - 40},
+                       p.rotation,
+                       rl.White,
+               )
+       }
        rl.DrawTexturePro(
```

We update the `Draw()` method on the **Player** struct to check if we are boosting; if we are, draw the boost sprite under the ship sprite.

```diff
        if rl.IsKeyDown(rl.KeyRight) {
                player.rotation += rotationSpeed
        }
+       // Default to not boosting
+       player.isBoosting = false

        // Accelerate the player with up
        if rl.IsKeyDown(rl.KeyUp) {
            if player.acceleration < 0.9 {
                player.acceleration += 0.1
            }
+            player.isBoosting = true
        }
```

Finally, we add a toggle on the acceleration action so `isBoosting` is true when accelerating and defaults to false when we are not.

Let's run the game with `go run main.go`

![Boosting ship](/images/asteroids/boosting.gif)

It now looks like our ship is using its engines to speed up!

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/f82631bf9e62a6e7722ae21ae7afcc39bcde1d1f/go-asteroids/main.go)

## Defining our Asteroids

Now that we have the player boosting and moving, we need to add the asteroids to the scene, so the player has something to dodge and shoot.

We start by following the same sprite selection pattern as before. We create a rectangle to select and we initialise it.

We also need a place to store the asteroids:

```diff
var (
	texTiles      rl.Texture2D
	texBackground rl.Texture2D
	spriteRec     rl.Rectangle
	boostRec      rl.Rectangle
+	asteroidRec   rl.Rectangle
+	asteroids     []Asteroid
	player        Player
)
```

We create an `asteroidRec` to select the sprite and an `asteroids` slice to store them in.

```diff
const (
	screenWidth      = 800
	screenHeight     = 400
	tileSize         = 64
	rotationSpeed    = 2.0
	playerSpeed      = 6.0
+	initialAsteroids = 5
)
```

We also add a constant for how many asteroids we want to spawn in to start with.

In the classic Asteroids manner, we are going to start with large asteroids. When we shoot them, they will break apart and spawn smaller asteroids. 

To do this, we need a way to track the size of the asteroid. We can use an enum for this, though. Go does not have an enum construct; we can use the `iota` to create one:
```go
// Enum for storing the size of the asteroid
type AsteroidSize int

const (
	Large AsteroidSize = iota
	Medium
	Small
)
```

Here, we create a new type `AsteroidSize` and provide it with three named options `Large`, `Medium` and `Small`. Under the hood, these are **int**s. However, we can now use them to make our code easier to read.

Now to create the **Asteroid**:

```go
type Asteroid struct {
	position     rl.Vector2
	speed        rl.Vector2
	size         rl.Vector2
	asteroidSize AsteroidSize
}

func (a *Asteroid) Draw() {
	// Draw the asteroid to the screen
	destTexture := rl.Rectangle{X: a.position.X, Y: a.position.Y, Width: a.size.X, Height: a.size.Y}
	rl.DrawTexturePro(
		texTiles,
		asteroidRec,
		destTexture,
		rl.Vector2{X: a.size.X / 2, Y: a.size.Y / 2},
		0.0,
		rl.White,
	)
}

func (a *Asteroid) Update() {
	// Move the asteroid in its direction
	a.position = rl.Vector2Add(a.position, a.speed)

	// Wrap the position, so they are always on screen
	wrapPosition(&a.position, a.size.X)
}
```

We start with the `Asteroid`, which contains the `position`, `size` and `speed` vectors. We have the new `asteroidSize` to determine how big to draw the sprite.

We create the `Draw()` method to draw the sprite onto the screen.

In the `Update()` method, we add the `speed` to the `position` to move the asteroid and ensure that our asteroids wrap around the screen.

### Creating Asteroids

Now that we have our Asteroid defined, we can start to create some when the game initialises. To do this, we create a helper function to generate a Large Asteroid in a random place, with a random speed and direction.

```go
// Asteroid helper functions
func createLargeAsteroid() Asteroid {

	// Generate a random edge of the screen to spawn
	randomEdge := rl.GetRandomValue(0, 3)
	var position rl.Vector2

	// Generate a random position on screen
	randomX := float32(rl.GetRandomValue(0, screenWidth))
	randomY := float32(rl.GetRandomValue(0, screenHeight))

	switch randomEdge {
	case 0:
		position = rl.Vector2{X: randomX, Y: +tileSize}
	case 1:
		position = rl.Vector2{X: screenWidth + tileSize, Y: randomY}
	case 2:
		position = rl.Vector2{X: randomX, Y: screenHeight + tileSize}
	case 3:
		position = rl.Vector2{X: -tileSize, Y: randomY}
	}

	// Generate a random speed and direction for the asteroid
	speed := rl.Vector2{
		X: float32(rl.GetRandomValue(-10, 10)) / 10,
		Y: float32(rl.GetRandomValue(-10, 10)) / 10,
	}

	// Create the large asteroid
	return createAsteroid(Large, position, speed)
}

```

This function uses the `getRandomValue` function to generate a number between `0` and `3`, which represent different sides of the screen. Then it creates a vector with a random speed. Finally, it passes it to another helper function to create the Asteroid.

```go
func createAsteroid(asteroidSize AsteroidSize, position, speed rl.Vector2) Asteroid {

	// Scale the image of the asteroid based on the asteroidSize
	var size rl.Vector2
	switch asteroidSize {
	case Large:
		size = rl.Vector2{X: tileSize * 1.0, Y: tileSize * 1.0}
	case Medium:
		size = rl.Vector2{X: tileSize * 0.7, Y: tileSize * 0.7}
	case Small:
		size = rl.Vector2{X: tileSize * 0.4, Y: tileSize * 0.4}
	}

	// Create the asteroid
	return Asteroid{
		position:     position,
		speed:        speed,
		size:         size,
		asteroidSize: asteroidSize,
	}
}
```

This function takes in a size and its vectors and creates an Asteroid of that size at the vectors. The helper function can be reused later when we want to create more asteroids.

Finally, we need to update the `initGame()` function to create our initial set of asteroids.

```diff
func initGame() {
+    // Create the asteroids field
+    asteroids = nil
+    for range initialAsteroids {
+        asteroids = append(asteroids, createLargeAsteroid())
+    }

	// Create the player
	player = Player{
```

We will also need to define our selection rectangle in the `init()` function, for the asset in the tile map:

```diff
	// Sprites for the ship and it boost
	spriteRec = rl.Rectangle{X: tileSize * 0, Y: tileSize * 2, Width: tileSize, Height: tileSize}
	boostRec = rl.Rectangle{X: tileSize * 7, Y: tileSize * 5, Width: tileSize, Height: tileSize}

+    // Sprite for the asteroid
+    asteroidRec = rl.Rectangle{X: tileSize * 1, Y: tileSize * 4, Width: tileSize, Height: tileSize}
+
	initGame()
}
```


Now, let's run the game with `go run main.go` and see what we have:

![Asteroids moving around the screen](/images/asteroids/moving_asteroids.gif)

We now have moving Asteroids!

The game is really coming to life now. 

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/59302777c639423b55d07bc68fb465bb8128d69a/go-asteroids/mainego)

## Enable collision detection

Next, let's add some interaction to the elements in the game.

We will start with collision detection between the player ship and the asteroids.

We will start with the consequence. The game should end when an asteroid hits the player. 

We start by creating a Game Over mechanic by adding a global state for it:

```diff
var (
	texTiles      rl.Texture2D
	texBackground rl.Texture2D
	spriteRec     rl.Rectangle
	boostRec      rl.Rectangle
	asteroidRec   rl.Rectangle
	asteroids     []Asteroid
	player        Player
+	gameOver      bool
)
```

We initialise that to be false in the `initGame()` function:
```diff
func initGame() {
+
+	// Start with it not being game over
+	gameOver = false
+
	// Create the asteroids field
	asteroids = nil
```

When the game is over, we want our game to stop. We can do this by only running the update when it is NOT gameover:

```go
func update() {
	// If it is not game over, update the frame
	if !gameOver {
		// Update the player
		player.Update()

		// Update the asteroid field
		for i := range asteroids {
			asteroids[i].Update()
		}

	}
}
```

Next, we draw the Game Over to the screen.

```diff
	// Draw the asteroid field
	for i := range asteroids {
		asteroids[i].Draw()
	}

+    if gameOver {
+        drawCenteredText("Game over", screenHeight/2, 50, rl.Red)
+    }

	// Draw the score to the screen
	rl.DrawText("Score 0", 10, 10, 20, rl.Gray)
```

We use a little helper function called `drawCenteredText` to measure how big our text is and then centre the text.

```go
func drawCenteredText(text string, y, fontSize int32, color rl.Color) {
	textWidth := rl.MeasureText(text, fontSize)
	rl.DrawText(text, screenWidth/2-textWidth/2, y, fontSize, color)
}
```

Great, now we can terminate the game when an Asteroid hits the player.

Let's check for collisions in our `update()` function:

```diff
        for i := range asteroids {
            asteroids[i].Update()
        }

+        checkCollisions()

    }
```

Now we can define our function, which detects when the player has collided with an Asteroid:

```go
func checkCollisions() {
	for i := len(asteroids) - 1; i >= 0; i-- {
		// Check for collision between player and asteroid
		if rl.CheckCollisionCircles(
			player.position,
			player.size.X/4,
			asteroids[i].position,
			asteroids[i].size.X/4,
		) {
			gameOver = true

		}
	}
}
```

We loop through each asteroid, starting from the end of the slice to the front. For each asteroid, we run the `CheckCollisionCircles` function. This takes a position and a radius. We use the player's `position` and `size`. We do the same with the asteroid, including scaling the radius to ensure the boundary is around the object.

When there is a collision, we set `gameOver` to true.

Ok, let's see if this works:

![](/images/asteroids/asteroid_crash.gif)

Great, we are now detecting the collision between the player and the asteroids.

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/610fad31ad7f3a87135b5d37443552284452bfbb/go-asteroids/main.go)

## Making our ship shoot

Now that we have our asteroids able to end our game, it is time for us to equip our ship with the ability to fight back. 

We will provide our ship with lasers to blast the asteroids.

First, let's create a struct for our laser:

```go
type Shot struct {
	position rl.Vector2
	speed    rl.Vector2
	radius   float32
	active   bool
}

func (s *Shot) Draw() {
	if s.active {
		rl.DrawCircleV(s.position, s.radius, rl.Yellow)
	}
}

func (s *Shot) Update() {
	if s.active {
		s.position.X += s.speed.X
		s.position.Y -= s.speed.Y

		if s.position.X < 0 || s.position.X > screenWidth || s.position.Y < 0 || s.position.Y > screenHeight {
			s.active = false
		}
	}
}
```

Here we create a new **Shot**, which has the positional and size attributes with the addition of an `active` flag to determine if the shot should be drawn. We provide a `Draw()` and `Update()` method for rendering and moving the shot.


We need to create a slice to store our shots, then loop through them to update and draw them to the screen. So we start by creating some global state and some constants:

In the **const**
```diff
	playerSpeed      = 6.0
+	shotSpeed        = 8.0
+	maxShots         = 10
	initialAsteroids = 5

```

In the **var**
```diff
	player        Player
	gameOver      bool
+	shots         []Shot
)
```

We then initialise our slice in the `init()` function:
```diff
	asteroidRec = rl.Rectangle{X: tileSize * 1, Y: tileSize * 4, Width: tileSize, Height: tileSize}

+	// Create the shots
+	shots = make([]Shot, maxShots)

	initGame()
```

and update the `initGame()` to initalise the state:

```diff

		asteroids = append(asteroids, createLargeAsteroid())
	}

+	// Create the laser shots
+	for i := range shots {
+		shots[i].active = false
+	}

	// Create the player
	player = Player{
```

Now that we have our shots, we need to draw them and update them. Let's update game loops`draw()` to include the shots.

```diff
		asteroids[i].Draw()
	}

+	// Draw the shots
+	for i := range shots {
+		shots[i].Draw()
+	}

	if gameOver {

```

And the game loops `update()`:
```diff
			asteroids[i].Update()
		}

+		// Update the shots
+		for i := range shots {
+			shots[i].Update()
+		}

		checkCollisions()
```

So we can now create, draw and update the shots when they are active. Let's create a way to fire a shot:

In the player's `Update()`, we add the ability to shoot when the Space bar is pressed:
```diff
	player.isBoosting = false

+	// Fire the lasers
+	if rl.IsKeyPressed(rl.KeySpace) {
+		fireShot()
+	}

	// Accelerate the player with up
	if rl.IsKeyDown(rl.KeyUp) {

```

Now we need the `fireShot()` function:
```go
func fireShot() {
	for i := range shots {
		// Find the first inactive shot
		if !shots[i].active {
			// Start at the players position
			shots[i].position = player.position
			shots[i].active = true

			// Get the players direction
			shotDirection := getDirectionVector(player.rotation)

			// Get the initial velocity
			shotVelocity := rl.Vector2Scale(shotDirection, shotSpeed)
			// Account for the players speed
			playerVelocity := rl.Vector2Scale(player.speed, player.acceleration)

			// Fire the shot, relative to the players speed
			shots[i].speed = rl.Vector2Add(playerVelocity, shotVelocity)

			shots[i].radius = 2
			// Break after one shot
			break
		}
	}
}
```

To fire a shot, we loop through all the shots and find the first inactive one. We then set its position and direction to the same as the players.
As the ship could be moving, we want to fire the laser at a constant speed relative to the ship.
We do this by adding the player's speed onto the speed of our shot, to give it relative speed, so it looks like the shots are always faster than the ship.

Let's check our game to see if it is working by running `go run main.go`

![Ship firing](/images/asteroids/ship_firing.gif)

Great, our ship now has weapons!

Though the lasers are not strong enough, they are passing right through the asteroids. Let's add some shot collisions so we can destroy the asteroids.

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/72071b73782680f245659c396dbfa6390659e69b/go-asteroids/main.go)

## Implement asteroid splitting

We want our asteroids to split when they are hit. To do this, we will need to spawn more asteroid objects when the collision between the shot and the asteroid occurs.

The asteroids should get progressively smaller and faster. The behaviour we will use is:

* Large -> 2 Medium
* Medium -> 4 Small
* Small -> Destroyed

Let's start by adding a counter for how many asteroids we have destroyed:

```diff
func checkCollisions() {
	for i := len(asteroids) - 1; i >= 0; i-- {
		// Check for collision between player and asteroid
		if rl.CheckCollisionCircles(
			player.position,
			player.size.X/4,
			asteroids[i].position,
			asteroids[i].size.X/4,
		) {
			gameOver = true

		}

+		// Check for a collision between shots and the asteroid
+		for j := range shots {
+			// Loop through all the active shots
+			if shots[j].active {
+				// If it has collided with an asteroid
+				if rl.CheckCollisionCircles(
+					shots[j].position,
+					shots[j].radius,
+					asteroids[i].position,
+					asteroids[i].size.X/2,
+				) {
+					// Destroy the shot and split the asteroid
+					shots[j].active = false
+
+					// The asteroid shot split according to our rules
+					splitAsteroid(asteroids[i])
+
+					// Remove the original asteroid from the slice
+					asteroids = append(asteroids[:i], asteroids[i+1:]...)
+
+					// Increase our score
+					asteriodsDestroyed++
+					break
+				}
+			}
+
+		}
+	}
}
```

We loop through all the shots and check if they have hit an asteroid; if they have, we split the asteroid.
We use a separate function that contains our splitting called `splitAsteroid()`.

```go
func splitAsteroid(asteroid Asteroid) {
	// Do nothing for small
	if asteroid.asteroidSize == Small {
		return

	}

	// Work out how many splits to do
	var newSize AsteroidSize
	var split int
	if asteroid.asteroidSize == Large {
		newSize = Medium
		split = 2
	} else {
		newSize = Small
		split = 4
	}

	// Create the new smaller asteroids
	for range split {
		angle := float64(rl.GetRandomValue(0, 360))
		direction := getDirectionVector(float32(angle))
		speed := rl.Vector2Scale(direction, 2.0)
		newAsteroid := createAsteroid(newSize, asteroid.position, speed)
		asteroids = append(asteroids, newAsteroid)
	}
}
```

We start by checking how big the asteroid is. If it's small, we don't need to do anything.
If it is large or medium, we set out many splits we need to do and the new size, then use the `createAsteroid` function we created earlier to create a new asteroid.
We add that asteroid to the `asteroids` slice so I can be rendered into our game.

Finally, we are taking note of how many asteroids we have destroyed. Let's use this as our score.

In the `draw()` function, let's display the score:
```diff
        // Draw the score to the screen
-       rl.DrawText("Score 0", 10, 10, 20, rl.Gray)
+       rl.DrawText(fmt.Sprintf("Score %d", asteriodsDestroyed), 10, 10, 20, rl.Gray)

        rl.EndDrawing()
```

Ok, moment of truth, let's see if this is working. Run `go run main.go`:

![Lasers spliting asteroids](/images/asteroids/asteroid_splitting.gif)

It is working!

We are hitting the asteroid, and it is splitting apart. Our score is increasing, and we need to start dodging the debris.

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/c17b3261b640bbd5c056c4755875426ae6701228/go-asteroids/main.go)

## Introduce a Game flow

Finally, let's make the game more playable. With a failure end state and a victory end state. We will also add a pause feature so players can stop the game.

Add two new states to **var**:
```diff
	gameOver           bool
+	paused             bool
+	victory            bool
	shots              []Shot
```

Set the initial state of the game in the `initGame()` function:
```diff
func initGame() {

+ 	// Start with it not being game over, pauses or victory
+ 	gameOver = false
+ 	victory = false
+ 	paused = false
+ 
+ 	// Reset score
+ 	asteriodsDestroyed = 0
...
```

We need a way to detect when the state changes. The `update` function is the perfect place for that:

```diff
func update() {
+	// If there are no asteroids left, we in
+	if len(asteroids) == 0 {
+		victory = true
+	}
+
+	// Toggle paused
+	if rl.IsKeyDown('P') {
+		paused = !paused
+	}
+
+	// Restart the game
+	if (gameOver || victory) && rl.IsKeyPressed('R') {
+		initGame()
+	}
+
+	// If it is not game over, update the frame
+	if !paused && !victory && !gameOver {
		// Update the player
		player.Update()
```

We call the `initGame` function again when we want to restart, as that function resets all the state, it restarts our game for us.

Finally, we need to update the `draw()` function to draw the UI for our players.

```diff
	if gameOver {
		drawCenteredText("Game over", screenHeight/2, 50, rl.Red)
+		drawCenteredText("Press R to restart", screenHeight/2+60, 20, rl.DarkGray)
+	}
+
+	if victory {
+		drawCenteredText("YOU WIN!", screenHeight/2, 50, rl.Gray)
+		drawCenteredText("Press R to restart", screenHeight/2+60, 20, rl.RayWhite)
	}

	// Draw the score to the screen
	rl.DrawText(fmt.Sprintf("Score %d", asteriodsDestroyed), 10, 10, 20, rl.Gray)
+	pauseTextSize := rl.MeasureText("[P]ause", 20)
+	rl.DrawText("[P]ause", screenWidth-pauseTextSize-10, 10, 20, rl.Gray)

	rl.EndDrawing()
```

Let's look at the final result with `go run main.go`:

![Game states with one asteroid](/images/asteroids/game_states.gif)

The game now has four states: Running, Victory, Paused and GameOver.

The code for this section can be found on my [GitHub Repo](https://github.com/timlittle/blog-code/blob/d7a37cd9bcf31e6b3e7f19582237a579d19190e5/go-asteroids/main.go)

## Conclusion

Congratulations, We’ve built a full Asteroids-style game in Go using Raylib-go.

In this tutorial, we covered:
- Creating windows with Raylib-go
- Drawing textures and sprites from tilemaps
- Implementing player movement with keyboard input
- Adding physics-based acceleration and rotation
- Creating dynamic game objects (asteroids)
- Collision detection between game entities
- Implementing shooting mechanics
- Game flow management (pause, victory, game over states) 

The complete source code is available on [GitHub](https://github.com/timlittle/blog-code/blob/main/go-asteroids/main.go). Fork it, experiment with it, and share your improvements!

This project demonstrates how tools like Raylib make game development accessible while teaching fundamental concepts like collision detection, state management, and real-time rendering that apply across many domains.

If you found this useful, consider giving it a clap or sharing! You can also read this post on [Medium](https://medium.com/timlittle/build-an-asteroids-game-with-raylib-go-4a92475b492c).
