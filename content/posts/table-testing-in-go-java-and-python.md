+++
date = '2025-09-10T03:02:55Z'
draft = false
title = "Table Testing in Go, Java, and Python: A Practical Guide"
subtitle = "Write cleaner, DRYer unit tests with table testing across three popular languages."
featuredImage = '/images/table-testing/featured.jpg'
categories = ["Programming", "Testing"]
+++

Have you ever written the same unit test repeatedly, with the only difference being the input parameters?

Table (or parameterized) testing is a technique that keeps your tests DRY and maintainable by running a single test function against multiple scenarios.

In this post, we'll explore table testing in **Go, Java, and Python** using **Conway's Game of Life** as an example.

## What is table testing?

The concept is simple, we write a test which accepts test cases with different parameters. The cases have the name, the starting state, and the expected state built into it.

These test cases are passed into a test function, which iterates through the test cases and runs the test with the parameters.

## Examples: Conway's Game of Life

I'll take the example of writing a unit test for the rules of [Conway's Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life). These rules are:

- Any live cell with fewer than 2 live neighbours dies.
- Any live cell with 2 or 3 live neighbours lives on to the next generation.
- Any live cell with more than 3 live neighbours dies.
- Any dead cell with exactly 3 live neighbours becomes a live cell.

To test this, we could write a function for these rules and then a separate unit test for each one:

```go
func TestIsCellAliveLiveCellShouldNotLiveWithLessThan2(t *testing.T) {
 got := IsCellAlive(1, 1)
 want := 0
 if got != want {
  t.Errorf("got %d, want %d", got, want)
 }
}
```

## Table testing in Go

We will start with Go, as I recently did for a previous [post](https://medium.com/@tim_little/building-conways-game-of-life-in-go-with-raylib-go-step-by-step-5e66d5133779?postPublishedType=initial).

Go has native support for subtests through `t.Run()`, which makes table testing a first-class citizen in the language.

```go
func TestIsCellAlive(t *testing.T) {
 testCases := []struct {
  name       string
  want       int
  current    int
  neighbours int
 }{
  {name: "Live cell should not live with <2", neighbours: 1, current: 1, want: 0},
  {name: "Live cell should live with 2", neighbours: 2, current: 1, want: 1},
  {name: "Live cells should live with 3", neighbours: 3, current: 1, want: 1},
  {name: "Dead cells should live with 3", neighbours: 3, current: 0, want: 1},
  {name: "Live cell should not live with >3", neighbours: 4, current: 1, want: 0},
  {name: "Dead cells should not live if already dead", neighbours: 0, want: 0},
 }

 for _, tt := range testCases {
  t.Run(tt.name, func(t *testing.T) {
   got := IsCellAlive(tt.current, tt.neighbours)

   if got != tt.want {
    t.Errorf("got %d, want %d", got, tt.want)
   }
  })
 }
}
```

We create a slice of structs with the input parameters, current value and the desired value. We pass these to the `t.Run()` function in a for loop, which runs it as a sub test.

```
--- PASS: TestIsCellAlive (0.00s)
    --- PASS: TestIsCellAlive/Live_cell_should_not_live_with_<2 (0.00s)
    --- PASS: TestIsCellAlive/Live_cell_should_live_with_2 (0.00s)
    --- PASS: TestIsCellAlive/Live_cells_should_live_with_3 (0.00s)
    --- PASS: TestIsCellAlive/Dead_cells_should_live_with_3 (0.00s)
    --- PASS: TestIsCellAlive/Live_cell_should_not_live_with_>3 (0.00s)
    --- PASS: TestIsCellAlive/Dead_cells_should_not_live_if_already_dead (0.00s)
```

When we run `go test -v` we can see the result of the test, as well as the name we pass to the `t.Run` function.

## Table Testing in Java

For Java, it does not have testing directly built into the language like Go. However, we can use Junit, which is a popular and widely used testing framework for Java. We can use [Junit's ParameterizedTest](https://docs.junit.org/5.0.2/api/org/junit/jupiter/params/ParameterizedTest.html) functionality for the table testing.

```java
    static Stream<Arguments> isCellAliveTestCases() {
        return Stream.of(
            Arguments.of("Live cell should not live with <2", 1, 1, 0),
            Arguments.of("Live cell should live with 2", 1, 2, 1),
            Arguments.of("Live cells should live with 3", 1, 3, 1),
            Arguments.of("Dead cells should live with 3", 0, 3, 1),
            Arguments.of("Live cell should not live with >3", 1, 4, 0),
            Arguments.of("Dead cells should not live if already dead", 0, 0, 0)
        );
    }
    
    @ParameterizedTest(name = "{0}")
    @MethodSource("isCellAliveTestCases")
    void testIsCellAlive(String name, int current, int neighbours, int expected) {
        int result = GameOfLife.isCellAlive(current, neighbours);
        assertEquals(expected, result);
    }
```

We create a stream of tests and pass them through as parameters to the test function. The test function can unpack the parameters and feed them into our assert function. This approach depends on Junit, however, it eliminates the for loop.

```
|   '-- testIsCellAlive(String, int, int, int) [OK]
|     +-- Live cell should not live with <2 [OK]
|     +-- Live cell should live with 2 [OK]
|     +-- Live cells should live with 3 [OK]
|     +-- Dead cells should live with 3 [OK]
|     +-- Live cell should not live with >3 [OK]
|     '-- Dead cells should not live if already dead [OK]
```

The output of the tests is similar to Go, they list the test and the success or failure of the test.

## Table testing in Python

Similar to Java, Python does not have testing built in, however, we can use the PyTest framework and use [PyTests parameterize](https://docs.pytest.org/en/stable/how-to/parametrize.html) functionality for a Pythonic approach to passing the tests into our test function.

```python
    @pytest.mark.parametrize("name,current,neighbours,expected", [
        ("Live cell should not live with <2", 1, 1, 0),
        ("Live cell should live with 2", 1, 2, 1),
        ("Live cells should live with 3", 1, 3, 1),
        ("Dead cells should live with 3", 0, 3, 1),
        ("Live cell should not live with >3", 1, 4, 0),
        ("Dead cells should not live if already dead", 0, 0, 0),
    ])
    def test_is_cell_alive(self, name, current, neighbours, expected):
        result = is_cell_alive(current, neighbours)
        assert result == expected
```

We define the test cases within the decorator, these are unpacked and passed into the function. We can run the `assert` function to confirm the functionality.

```
test_game_of_life.py::TestGameOfLife::test_is_cell_alive[Live cell should not live with <2-1-1-0] PASSED [ 50%]
test_game_of_life.py::TestGameOfLife::test_is_cell_alive[Live cell should live with 2-1-2-1] PASSED [ 58%]
test_game_of_life.py::TestGameOfLife::test_is_cell_alive[Live cells should live with 3-1-3-1] PASSED [ 66%]
test_game_of_life.py::TestGameOfLife::test_is_cell_alive[Dead cells should live with 3-0-3-1] PASSED [ 75%]
test_game_of_life.py::TestGameOfLife::test_is_cell_alive[Live cell should not live with >3-1-4-0] PASSED [ 83%]
test_game_of_life.py::TestGameOfLife::test_is_cell_alive[Dead cells should not live if already dead-0-0-0] PASSED [ 91%]
```

The test output is the same, however, formatted to a single line per test. Rather than the nested output from both Java and Go.

## Conclusion

Table testing keeps your test suites concise, consistent, and easy to maintain. Whether you're using **Go**, **Java**, or **Python**, the core idea stays the same: define your cases, iterate over them, and let your test runner handle the rest.

Have you tried table testing in other languages like JavaScript, Rust, or C? Share your favourite approach in the comments. I'd love to see how other communities solve this!

---

*Also available on [Medium](https://medium.com/@tim_little/table-testing-in-go-java-and-python-a-comprehensive-guide-to-data-driven-testing-b8f5c5f5c5f5)*
