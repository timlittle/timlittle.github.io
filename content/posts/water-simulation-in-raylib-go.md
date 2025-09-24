+++
date = '2025-09-23T04:05:13Z'
draft = true
title = "Water Simulation in Golang and Raylib"
subtitle = "Building a realistic water physics simulation using cellular automata in Go"
featuredImage = '/images/water-simulation/featured.png'
categories = ["Programming", "Game Development"]
+++

Building realistic water physics can seem daunting, but with cellular automata and some simple rules, we can create convincing fluid behavior. In this tutorial, we'll build a water simulation from scratch using Go and raylib-go.

## Game Setup

First, we need to set up the Go module and install the raylib-go package:

```go
mkdir go-watersim
go mod init watersim
go get -v -u github.com/gen2brain/raylib-go/raylib
```

Set up a Game struct with a draw() method to render its state:

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

Set up a Droplet struct with the draw() method implemented to draw a blue square representing a single water droplet. We convert the window into cells for cellular simulation:

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

Next, we'll set up the initial game state with helper functions:

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

Finally, create our main() function, which initializes the raylib window:

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

Run the program:

```bash
go run main.go
```

![Water droplet on a black background](/images/water-simulation/featured.png)

Now we can see our Raylib window with a single blue square drawn on it, representing our water droplet. Not very impressive at the moment!

## Rules

Next, we start to automate this simulation. We can begin to bring life to this by following some logical rules to move the water in a fluid-like manner.

![Flow and pressure rules for simulation](/images/water-simulation/rules.png)

## Gravity

Let's start with gravity. This will pull the water droplet down at a constant rate. For our simulation, we need to check the cell directly below the droplet for space, then transfer part of our volume to it.

We start by creating an Update() method on the game to update the state in each frame:

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

We create a new frame and copy the current state into it. We then loop from the bottom of the current array to the top, so we process the state in order.

For each cell of the state, we run the processWaterCell() method on it, which will apply our logical rules to the Droplet:

```go
func processWaterCell(x, y int, newState *[][]Droplet) {
    // Try to flow downwards, as if by gravity
    fill(&(*newState)[y][x], &(*newState)[y+1][x], 1.0, 0.5)
}
```

This function is simple for now—it transfers the volume from the current cell to the one below it. However, we need the fill() function, which transfers volume from one cell to another if there's space:

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

We use the remainder helper function to calculate how much space is left in the target cell before transferring the current volume to the new one. We rate limit the transfer to help the simulation feel smooth.

Finally, we include our Update() method in our game loop:

```diff
                // Draw the game state
                game.Draw()

+               // Update the game state based on the rules
+               game.Update()
+
                rl.EndDrawing()
```

Now we can run the simulation:

```bash
go run .
```

![Water droplet affected by gravity](/images/water-simulation/gravity.gif)

It's working! The water droplet is falling to the ground and stopping when it reaches the bottom of the screen.

However, it's getting duplicated as it falls, so two cells look like they're full. We can resolve this by checking the height based on volume and checking if the cell above has volume:

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

We can update our Game.Draw() method to first check if there's water above the current cell. We can pass this to the Droplet.Draw() method to make a decision on where to start filling from:

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
}
```

The **Droplet.Draw()** method calculates how high the cell should be based on the volume and then offsets the drawing by the volume amount to fill from the bottom. If there's water above it, we fill from the top.

Let's run our game now and see the result:

```bash
go run .
```

![Smooth water fall](/images/water-simulation/smooth-fall.gif)

It looks much smoother now! The droplet is falling in a continuous motion to the bottom.

## Start a Flow of Water

Currently, we're generating a single droplet and then allowing it to fall. However, we want a flow of water so we can see how the droplets interact with each other:

```go
func CreateWaterGenerator(x, y, tileSize int, state *[][]Droplet) {
    droplet := Droplet{size: tileSize, volume: 1.0}
    (*state)[y][x] = droplet
}
```

Next, we update the main game loop by counting the number of frames that have passed and then adding water regularly:

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

![Water flowing, though stacking](/images/water-simulation/water-flow.gif)

Water flowing, though stacking.

## Flow (Sideways)

Edit the process to add horizontal flow:

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

Add the functions:

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

![Sideways Flow](/images/water-simulation/sideways-flow.gif)

Flowing now!

## Pressure (Diagonal)

Add diagonal flow to the process:

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

Add a function for diagonal flow:

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

![Smooth flow](/images/water-simulation/smooth-flow.gif)

Much smoother! The water now flows naturally, spreading sideways when blocked and finding its way around obstacles.

## Conclusion

We've built a convincing water simulation using simple cellular automata rules:

1. **Gravity** - Water flows downward when possible
2. **Horizontal flow** - Water spreads sideways when blocked below
3. **Diagonal flow** - Water finds alternative paths through diagonal movement

The key to realistic fluid behavior is processing cells in the right order (bottom-up) and using rate-limited transfers to create smooth motion. This foundation can be extended with additional features like:

- Obstacles and terrain
- Different fluid types
- Pressure systems
- Evaporation and condensation

The complete source code is available on [GitHub](https://github.com/timlittle/blog-code/tree/main/go-watersim).

---

*Also available on [Medium](https://medium.com/@tim_little/water-simulation-in-golang-and-raylib-91c895891263)*
