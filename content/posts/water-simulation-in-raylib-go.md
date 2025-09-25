+++
date = '2025-09-25T04:05:13Z'
draft = false
title = "Build a Water Simulation in Go with Raylib-go"
description = "Create a lightweight water simulation for 2D games using cellular automata and simple physics rules in Go with raylib-go"
featuredImage = '/images/water-simulation/featured.png'
categories = ["Game Development", "Programming"]
tags = ["go", "raylib", "game-development", "physics-simulation", "cellular-automata", "simulation", "graphics-programming", "tutorial"]
keywords = ["water simulation go", "raylib go tutorial", "cellular automata", "game physics", "fluid simulation", "go programming", "2d game development"]
+++

In this blog post, we will use [raylib-go](https://github.com/gen2brain/raylib-go) to create a lightweight water simulation for 2D games.

![Water simulation](/images/water-simulation/watersim.gif)
*Water simulation*

This post aims to create a simulation of water which flows naturally and presents the illusion of flow and volume. [Fluid Simulation](https://en.wikipedia.org/wiki/Computational_fluid_dynamics) is a huge topic. To keep things simple, we will use cellular automation to update each cell.

Each cell will follow a set of rules:

- **Gravity** — droplets fall straight down if there's space.
- **Side Flow** — when blocked, water spreads left and right.
- **Pressure** — when blocked on both sides, it flows diagonally.

## Game setup

First, we set up the Go module and install the raylib-go package.

```go
mkdir go-watersim
go mod init watersim
go get -v -u github.com/gen2brain/raylib-go/raylib
```

We create a **Game** struct to hold our game state and to manage the game loop. We create a `Draw()` method to draw its state to the screen by using the `Draw()` function on our Droplet, which represents the water.

```go
type Game struct {
 Width    int
 Height   int
 State    [][]Droplet // 2D grid of water droplets [y][x]
 tileSize int
}

func (g *Game) Draw() {
 // Loop through each cell and call the cells Draw() method
 for y := range g.State {
  for x := 0; x < len(g.State[y]); x++ {
   g.State[y][x].Draw(x, y, g.tileSize)
  }
 }
}
```

Next, we create the **Droplet** entity and create a `Draw()` method to draw a single water droplet to the screen, represented by a blue square. To work with the cellular simulation, we convert the coordinates to cells.

```go
type Droplet struct {
 volume float64 // How much water this cell contains (0.0 to 1.0)
 size   int
}

func (d *Droplet) Draw(x, y, tileSize int) {
 // Convert grid coordinates to pixel coordinates
 pixelX := x * tileSize
 pixelY := y * tileSize

 if d.volume > 0 {
  // Draw the blue water rectangle
  rl.DrawRectangle(int32(pixelX), int32(pixelY), int32(tileSize), int32(tileSize), rl.Blue)
 }
}
```

Next, we need to create the **Game** and its initial cellular state. To assist, we create helper functions to set up the **Game** and to create the initial state.

```go
func NewGame(width, height, tileSize int) *Game {
 // Create a new game
 g := &Game{Width: width, Height: height, tileSize: tileSize}

 // Create the new game state
 // divide pixel dimensions by tile size to get grid size
 g.State = CreateGameState(g.Width/g.tileSize, g.Height/g.tileSize, tileSize)

 return g
}

func CreateGameState(newWidth, newHeight, tileSize int) [][]Droplet {
 // Create a new game state
 newState := make([][]Droplet, newHeight)

 // Loop through each row
 for y := range newHeight {

  // Create the columns
  newState[y] = make([]Droplet, newWidth)

  // Loop through each cell and create a Droplet
  for x := range newState[y] {
   newState[y][x] = Droplet{
    size: tileSize,
   }
  }
 }
 return newState
}
```

Finally, we update our `main()` function to initialise the raylib window and uses the new helper functions to create the game state and draw it to the screen.

```go
 // Setup the new game
 var game = NewGame(800, 400, 10)

 // Initialize Raylib graphics window
 rl.InitWindow(int32(game.Width), int32(game.Height), "Water simulation")
 defer rl.CloseWindow()

 // Create a single water droplet and add it to the screen
 droplet := Droplet{size: game.tileSize, volume: 1.0}
 game.State[100/game.tileSize][400/game.tileSize] = droplet

 // Setup the frame per second rate
 rl.SetTargetFPS(20)

 // Main loop
 for !rl.WindowShouldClose() {

  // Begin to draw and set the background to black
  rl.BeginDrawing()
  rl.ClearBackground(rl.Black)

  // Draw the game state
  game.Draw()

  rl.EndDrawing()
 }
```

This provides us with a cellular grid, with a single water droplet drawn to the screen. Let's confirm this by running the program.

```go
go run main.go
```

![Water droplet on a black background](/images/water-simulation/hello.png)
*Water droplet on a black background*

We see our Raylib window with a single blue square drawn on it, representing our water droplet. Not very impressive at the moment!

The code for this section of the guide is [available on GitHub](https://github.com/timlittle/blog-code/blob/c243a8102fb53497ea93cad1ca7ed1875d2defca/go-watersim/main.go).

## Logic Rules

We begin to bring life to the game by following logical rules to move the water in a fluid-like manner.

![Flow and pressure rules for simulation](/images/water-simulation/rules.png)
*Flow and pressure rules for simulation*

We start with the impact of gravity, which flows downwards and fills the cell **below** it. If there is no more water, the interaction is complete. If there is water remaining, the droplet will flow to the **sides** and **diagonally**.

## Gravity

Gravity pulls the water droplet down at a constant rate. For our simulation, we need to check that the cell directly below the droplet has space, then transfer part of our volume to it.

To ensure there is space, we loop through the state from the bottom upwards. This ensures lower cells are processed first, to prevent water from "teleporting" through other cells.

We create an `Update()` method on the **Game** struct to process each frame. We apply the rules to the cell and then draw the new state to the screen.

```go
func (g *Game) Update() {
 // Create a new state to avoid modifying the current state while reading it
 newState := CreateGameState(len(g.State[0]), len(g.State), g.tileSize)

 // Copy current state to new state
 for y := range g.State {
  copy(newState[y], g.State[y])
 }

 // Process the simulation from the bottom upwards
 for y := len(g.State) - 1; y >= 0; y-- {
  for x := range g.State[y] {

   // Only process cells that contain water
   if g.State[y][x].volume > 0 {

    // Check if we are at the bottom boundary
    if y+1 < len(g.State) {
     processWaterCell(x, y, &newState)
    }
   }
  }
 }

 // Replace old state with new calculated state
 g.State = newState

}
```

We create a new frame and copy the current state into it. We loop from the bottom to ensure we are not overfilling.

For each cell, we run the `processWaterCell()` method on it, which will apply our logical rules to the **Droplet**.

```go
func processWaterCell(x, y int, newState *[][]Droplet) {
 // Try to flow downwards, as if by gravity
 fill(&(*newState)[y][x], &(*newState)[y+1][x], 1.0, 0.5)
}
```

This function is simple for now, it transfers the volume from the current cell to the one below it.

We use a `fill()` function, which transfers volume from one cell to another, and we use the `remainder()` function to calculate how much space is left in the target cell before transferring the volume to the new one. We rate limit the transfer with the **flowRate** to help the simulation feel smooth.

```go
// Calculate how much more water a droplet can hold
func remainder(droplet Droplet, maxVolume float64) float64 {
 return maxVolume - droplet.volume
}

// Fill transfers water between two droplets at a controlled rate
func fill(current, target *Droplet, maxVolume, flowRate float64) {

 // Calculate how much water can be transferred
 transfer := remainder(*target, maxVolume)

 // Limit transfer to the flow rate (prevents instant teleportation)
 if transfer > flowRate {
  transfer = flowRate
 }

 // Move water from source to target
 current.volume -= transfer
 target.volume += transfer
}
```

Finally, we include our `Update()` method in our game loop to update the droplets:

```diff
                // Draw the game state
                game.Draw()

+               // Update the game state based on the rules
+               game.Update()
+
                rl.EndDrawing()
```

Now we can run the simulation and confirm we have a water droplet affected by gravity.

```go
go run .
```

![Water droplet affected by gravity](/images/water-simulation/gravity.gif)
*Water droplet affected by gravity*

It works!

The water droplet is falling to the ground and stopping when it reaches the bottom of the screen.

However, it is getting duplicated as it falls, so two cells look like they are full.

We can resolve this by changing the height based on its volume and checking if the cell above has volume. If it does, we can fill from the top, if not, we fill from the bottom.

```go
func (g *Game) Draw() {
 // Loop through each cell and call the cells Draw() method
 for y := range g.State {
  for x := 0; x < len(g.State[y]); x++ {

   // Check if there is water above this cell
   hasWaterAbove := y > 0 && g.State[y-1][x].volume > 0

   g.State[y][x].Draw(x, y, g.tileSize, hasWaterAbove)
  }
 }
}
```

We update our `Game.Draw()` method to check if there is water in the cell above. We pass this to the `Droplet.Draw()` method to decide on where to start filling from.

```go
func (d *Droplet) Draw(x, y, tileSize int, hasWaterAbove bool) {
 // Convert grid coordinates to pixel coordinates
 pixelX := x * tileSize
 pixelY := y * tileSize

 if d.volume > 0 {
  // Calculate visual height based on water volume
  // Full volume (1.0) = full tile height, half volume (0.5) = half height
  height := int(float64(tileSize) * d.volume)

  // Fill up from the bottom
  offsetY := tileSize - height

  // If water above, fill from the top instead
  if hasWaterAbove {
   offsetY = 0
  }

  // Draw the blue water rectangle
  rl.DrawRectangle(int32(pixelX), int32(pixelY+offsetY), int32(tileSize), int32(height), rl.Blue)
 } 
```

The `Droplet.Draw()` method calculates how high each cell should be based on volume and offsets the drawing by the volume amount to fill from the bottom. If there is water above it, we fill from the top.

Let's run our game now and see what the result is:

```go
go run .
```

![Smooth water fall](/images/water-simulation/smooth-fall.gif)
*Smooth water fall*

It looks much smoother now. The droplet is falling in a continuous motion to the bottom.

The code for this section can be found in the [GitHub repository](https://github.com/timlittle/blog-code/blob/420c5ed97bbc7e10a4f82aa66484fc482d216cd2/go-watersim/main.go)

## Start a flow of water

Currently, we are generating a single droplet. However, we want a flow of water to see how the droplets interact with each other.

We do this by creating a helper function to generate **Droplet** objects.

```go
func CreateWaterGenerator(x, y, tileSize int, state *[][]Droplet) {
 droplet := Droplet{size: tileSize, volume: 1.0}
 (*state)[y][x] = droplet
}
```

We replace the manual creation of the **Droplet** in the main game loop and count the number of frames to add water regularly.

```diff
-       // Create a single water droplet and add it to the screen
-       droplet := Droplet{size: game.tileSize, volume: 1.0}
-       game.State[100/game.tileSize][400/game.tileSize] = droplet
+       // Set up a counter, so we can spawn new water at a rate
+       frameCount := 0
+       flowStartX := 100 / game.tileSize
+       flowStartY := 100 / game.tileSize
+       CreateWaterGenerator(flowStartX, flowStartY, game.tileSize, &game.State)

        // Setup the frame per second rate
        rl.SetTargetFPS(20)

        // Main loop
        for !rl.WindowShouldClose() {
+               frameCount++

                // Begin to draw and set the background to black
                rl.BeginDrawing()
                rl.ClearBackground(rl.Black)

+               // Add new water every 5 frames (creates continuous water stream)
+               if frameCount%5 == 0 {
+                       CreateWaterGenerator(flowStartX, flowStartY, game.tileSize, &game.State)
+                       CreateWaterGenerator(flowStartX+1, flowStartY, game.tileSize, &game.State)
+                       CreateWaterGenerator(flowStartX-1, flowStartY, game.tileSize, &game.State)
+               }
+
```

Now, we run the program, we expect there to be water dripping from a point at the top of the screen.

```go
go run .
```

![Water flowing but stacking](/images/water-simulation/stack.gif)

Ah, that does not look right!

On the plus side, we have our **Droplet** objects continually following. On the downside, they are stacking on top of each other. We address that by adding sideways flow in the next section.

The code for this section can be found in the [GitHub repository](https://github.com/timlittle/blog-code/blob/0f21c6e10f5f04ade3408d9ec925c5c1ae6b5434/go-watersim/main.go)

## Flow (Sideways)

The stacking is happening as droplets are incompressible and cannot flow downwards anymore. To fix this, we update our code to flow sideways.

We start by checking that there is water still in the cell. If there is, we start to flow the water out to the side cells.

```diff
 func processWaterCell(x, y int, newState *[][]Droplet) {
        // Try to flow downwards, as if by gravity
        fill(&(*newState)[y][x], &(*newState)[y+1][x], 1.0, 0.5)
+
+       // If all water flowed down, no need to continue
+       if (*newState)[y][x].volume == 0 {
+               return
+       }
+
+       // If water can still flow down, don't try other directions yet
+       if canFlowDown(x, y, newState) {
+               return
+       }
+
+       // Water spreads sideways when blocked below
+       tryHorizontalFlow(x, y, newState)
+
 }
```

The `canFlowDown()` checks if there is still space to flow downwards, if there is, we exit and process the remainder in the next frame.

The `tryHorizontalFlow()` method cascades to the left and the right. It checks the next 3 cells and adds a fraction of the remaining volume to the cells. This allows the water to "settle" when it cannot flow downwards.

```go
func canFlowDown(x, y int, state *[][]Droplet) bool {
 return y+1 < len(*state) && (*state)[y+1][x].volume < 1.0
}

func tryHorizontalFlow(x, y int, state *[][]Droplet) {
 current := &(*state)[y][x]

 // Only cascade if there's water below
 hasWaterBelow := y+1 < len(*state) && (*state)[y+1][x].volume > 0.5
 if !hasWaterBelow {
  return
 }

 // Cascade right - distribute to multiple cells
 for offset := 1; offset <= 3 && x+offset < len((*state)[y]); offset++ {
  target := &(*state)[y][x+offset]
  if target.volume < current.volume {
   flowRate := (current.volume - target.volume) * 0.1 / float64(offset)
   fill(current, target, 1.0, flowRate)
  }
 }

 // Cascade left - distribute to multiple cells
 for offset := 1; offset <= 3 && x-offset >= 0; offset++ {
  target := &(*state)[y][x-offset]
  if target.volume < current.volume {
   flowRate := (current.volume - target.volume) * 0.1 / float64(offset)
   fill(current, target, 1.0, flowRate)
  }
 }
}
```

Now, when we run, we expect to see the water flow to the sides, with the surrounding cells taking a percentage of the water.

```go
go run .
```

![Sideways Flow](/images/water-simulation/sideways-flow.gif)
*Sideways Flow*

Excellent, the water is flowing to the side, spreading out and flowing along the bottom of the screen.

There is still stacking, and the effect is rather blocky at the edges. This is a result of volume still being in the current cell after the downward and sideways movement. Let's fix that in the next section.

The code for this section can be found in the [GitHub repository](https://github.com/timlittle/blog-code/blob/6c08e2f6680699cf0b2216ca659d0f4db00b7e0a/go-watersim/main.go)

## Pressure (diagonal)

To present a smoother flow of water, we can introduce more pressure dynamics. As water is incompressible, we would need the water to flow in all downward trajectories. Let's update the `processWaterCell()` function to try diagonal transfer when there is still volume.

```diff
 func processWaterCell(x, y int, newState *[][]Droplet) {
        // Try to flow downwards, as if by gravity
        fill(&(*newState)[y][x], &(*newState)[y+1][x], 1.0, 0.5)

        // If all water flowed down, no need to continue
        if (*newState)[y][x].volume == 0 {
                return
        }

        // If water can still flow down, don't try other directions yet
        if canFlowDown(x, y, newState) {
                return
        }

        // Water spreads sideways when blocked below
        tryHorizontalFlow(x, y, newState)

+       if (*newState)[y][x].volume > 0 {
+               tryDiagonalFlow(x, y, newState)
+       }
+
 }
```

We add a `tryDiagonalFlow()` function to flow diagonally. It checks that the target cell is within bounds and then transfers volume if there is space.

```go
func tryDiagonalFlow(x, y int, state *[][]Droplet) {
 current := &(*state)[y][x]

 // Flow diagonally down-right if space is available
 if x+1 < len((*state)[y]) && y+1 < len(*state) && (*state)[y+1][x+1].volume < 1.0 {
  fill(current, &(*state)[y+1][x+1], 1.0, 0.25)
 }

 // Flow diagonally down-left if space is available
 if x > 0 && y+1 < len(*state) && (*state)[y+1][x-1].volume < 1.0 {
  fill(current, &(*state)[y+1][x-1], 1.0, 0.25)
 }
}
```

Let's run the program. We expect the water flow to be smoother and less blocky.

```go
go run .
```

![Smooth flow](/images/water-simulation/smooth-flow.gif)
*Smooth flow*

Great! The water is smoother and is slowly filling the screen in tiny increments.

The code for this section can be found in the [GitHub repository](https://github.com/timlittle/blog-code/blob/9e99488d1bbc7134a3bbd9a832155d483cdec205/go-watersim/main.go)

## Obstacles

At the moment, the water is falling to the bottom and flowing outwards. It is not very impressive.

Let's update the scene to include obstacles that the water interacts with. The water should be blocked by the obstacles and to flow over it like natural water.

We start by adding a flag to our cell to indicate if it is an obstacle.

```diff
 type Droplet struct {
-       volume float64 // How much water this cell contains (0.0 to 1.0)
-       size   int
+       volume     float64 // How much water this cell contains (0.0 to 1.0)
+       size       int
+       isObstacle bool // Is this cell an obstacle?
 }
```

We updated the `Draw()` method to draw an obstacle in brown rather than blue.

```diff
 func (d *Droplet) Draw(x, y, tileSize int, hasWaterAbove bool) {
        // Convert grid coordinates to pixel coordinates
        pixelX := x * tileSize
        pixelY := y * tileSize

+       if d.isObstacle {
+               // Draw obstacle as brown rectangle
+               rl.DrawRectangle(int32(pixelX), int32(pixelY), int32(tileSize), int32(tileSize), rl.Brown)
+       }
+
        if d.volume > 0 {
                // Calculate visual height based on water volume
                // Full volume (1.0) = full tile height, half volume (0.5) = half height
                height := int(float64(tileSize) * d.volume)
```

Next, we update our collision logic in our rules to ensure we are only flowing to cells which are not obstacles.

We start with the `canFlowDown` function:

```diff
 func canFlowDown(x, y int, state *[][]Droplet) bool {
-       return y+1 < len(*state) && (*state)[y+1][x].volume < 1.0
+       return y+1 < len(*state) && (*state)[y+1][x].volume < 1.0 && !(*state)[y+1][x].isObstacle
 }
```

Then the `tryHorizontalFlow()` function:

```diff
 func tryHorizontalFlow(x, y int, state *[][]Droplet) {
        current := &(*state)[y][x]

        // Only cascade if there's water below
        hasWaterBelow := y+1 < len(*state) && (*state)[y+1][x].volume > 0.5
        if !hasWaterBelow {
                return
        }

        // Cascade right - distribute to multiple cells
        for offset := 1; offset <= 3 && x+offset < len((*state)[y]); offset++ {
                target := &(*state)[y][x+offset]
-               if target.volume < current.volume {
+               if target.volume < current.volume && !target.isObstacle {
                        flowRate := (current.volume - target.volume) * 0.1 / float64(offset)
                        fill(current, target, 1.0, flowRate)
                }
        }

        // Cascade left - distribute to multiple cells
        for offset := 1; offset <= 3 && x-offset >= 0; offset++ {
                target := &(*state)[y][x-offset]
-               if target.volume < current.volume {
+               if target.volume < current.volume && !target.isObstacle {
                        flowRate := (current.volume - target.volume) * 0.1 / float64(offset)
                        fill(current, target, 1.0, flowRate)
                }
        }
 }
```

And finally the `tryDiagonalFlow()` function:

```diff
 func tryDiagonalFlow(x, y int, state *[][]Droplet) {
        current := &(*state)[y][x]

        // Flow diagonally down-right if space is available
-       if x+1 < len((*state)[y]) && y+1 < len(*state) && (*state)[y+1][x+1].volume < 1.0 {
+       if x+1 < len((*state)[y]) && y+1 < len(*state) && (*state)[y+1][x+1].volume < 1.0 && !(*state)[y+1][x+1].isObstacle {
                fill(current, &(*state)[y+1][x+1], 1.0, 0.25)
        }

        // Flow diagonally down-left if space is available
-       if x > 0 && y+1 < len(*state) && (*state)[y+1][x-1].volume < 1.0 {
+       if x > 0 && y+1 < len(*state) && (*state)[y+1][x-1].volume < 1.0 && !(*state)[y+1][x-1].isObstacle {
                fill(current, &(*state)[y+1][x-1], 1.0, 0.25)
        }
 }
```

We will also need to update the main gravity logic in the `processWaterCell()` function:

```diff
 func processWaterCell(x, y int, newState *[][]Droplet) {
-       // Try to flow downwards, as if by gravity
-       fill(&(*newState)[y][x], &(*newState)[y+1][x], 1.0, 0.5)
+       // Try to flow downwards, as if by gravity (but not into obstacles)
+       if y+1 < len(*newState) && !(*newState)[y+1][x].isObstacle {
+               fill(&(*newState)[y][x], &(*newState)[y+1][x], 1.0, 0.5)
+       }
```

Great!

Now we create some helper functions to create the obstacles on the screen.

```go
func CreateHorizontalObstacle(x, y, size int, state *[][]Droplet) {
 for offset := range size {
  (*state)[y][x+offset].isObstacle = true
  (*state)[y+1][x+offset].isObstacle = true
  (*state)[y+2][x+offset].isObstacle = true
 }
}
func CreateVerticleObstacle(x, y, size int, state *[][]Droplet) {
 for offset := range size {
  (*state)[y+offset][x].isObstacle = true
  (*state)[y+offset][x+1].isObstacle = true
  (*state)[y+offset][x+2].isObstacle = true
 }
}
```

While we are looking at object creation, we can simplify the water generation code by using the same looping method.

```diff
func CreateWaterGenerator(x, y, tileSize int, state *[][]Droplet) {
-       droplet := Droplet{size: tileSize, volume: 1.0}
-       (*state)[y][x] = droplet
+       for xOffset := 0; xOffset <= 4; xOffset++ {
+               droplet := Droplet{size: tileSize, volume: 1.0}
+               (*state)[y][x+xOffset] = droplet
+
+       }
 }
```

Finally, we can update our `main()` function to add obstacles to block the path of the water and to DRY up the water generation code:

```diff
  func main() {
        // Setup the new game
        var game = NewGame(800, 400, 10)

        // Initialize Raylib graphics window
        rl.InitWindow(int32(game.Width), int32(game.Height), "Water simulation")
        defer rl.CloseWindow()

        // Set up a counter, so we can spawn new water at a rate
        frameCount := 0
-       flowStartX := 100 / game.tileSize
+       flowStartX := 400 / game.tileSize
        flowStartY := 100 / game.tileSize
        CreateWaterGenerator(flowStartX, flowStartY, game.tileSize, &game.State)

+       CreateVerticleObstacle(30, 20, 10, &game.State)
+
+       CreateHorizontalObstacle(0, 30, 50, &game.State)
+       CreateHorizontalObstacle(40, 20, 40, &game.State)
+

        // Setup the frame per second rate
        rl.SetTargetFPS(20)

        // Main loop
        for !rl.WindowShouldClose() {
                frameCount++

                // Begin to draw and set the background to black
                rl.BeginDrawing()
                rl.ClearBackground(rl.Black)

                // Add new water every 5 frames (creates continuous water stream)
-               if frameCount%5 == 0 {
+               if frameCount%3 == 0 {
                        CreateWaterGenerator(flowStartX, flowStartY, game.tileSize, &game.State)
-                       CreateWaterGenerator(flowStartX+1, flowStartY, game.tileSize, &game.State)
-                       CreateWaterGenerator(flowStartX-1, flowStartY, game.tileSize, &game.State)
                }
```

Moment of truth…

Let's run the code and see if it is flowing over the obstacles as we expect:

```go
go run .
```

![Water flowing down obstacles](/images/water-simulation/watersim.gif)
*Water flowing down obstacles*

Amazing, the water is flowing downwards until it reaches the obstacles, then it flows across the top of the obstacle until it reaches the side of the screen or another obstacle.

There is still stacking when water is falling on top of other water.

**Insight:** This doesn't really behave like water.
What we've built behaves more like sand than water. This approach is lightweight and fun for games that need falling particles like sand traps, dust, lava.

It might have been better to call this Sand Simulation!

![Sand Simulation](/images/water-simulation/sand.gif)
*Sand Simulation*

The code for this section can be found in the [GitHub repository](https://github.com/timlittle/blog-code/blob/5e6e9e069fd64aaa8c140d91f142c3d1cce289b3/go-watersim/main.go)

## Closing the gap

This method works for a simple game that requires sand physics. Think of a sand avalanche in a side scroller that traps the player.

However, the water implementation would need some modification. It needs to calculate pressure, velocity and momentum to get the natural fluid motion.

We could go deeper, we would need to add water tension, viscosity and turbulence. The simulation would need to solve the [Navier–Stokes equations](https://en.wikipedia.org/wiki/Navier%E2%80%93Stokes_equations) for flow behaviours. An excellent example of this is [Sebastian Lague's Simulating Fluids](https://www.youtube.com/watch?v=rSKMYc1CQHE).

## Conclusion

In this post, we:

- Created a simple water/sand simulator using raylib-go
- Use a simple set of rules to simulate complex behaviours
- Added obstacles that the water would flow across

If you found this useful, consider giving it a clap or sharing! You can also check out the [GitHub repo](https://github.com/timlittle/blog-code/blob/main/go-watersim/main.go) and try modifying the rules to create lava, smoke, or sand effects.

Next up, I am going to look into building a simple game with raylib-go, to explore more of the raylib library and to have a bit of fun with the output.
