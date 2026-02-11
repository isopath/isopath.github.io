---
title: "matrix.go"
date: 2025-02-05T14:00:00Z
draft: false
tags: ["go", "bubbletea", "terminal", "tui"]
---

Repository: https://github.com/isopath/matrix.go

---

`//go:embed` directive allows programs to include arbitrary files in the go binary at build time.

```go
//go:embed assets/*.txt
var assetsFS embed.FS
```

We use this to include all the text files from the `assets/` directory (used for the text for rain). All values are stored in the variable `assetsFS` of type [`embed.FS`](https://pkg.go.dev/embed#FS), a read-only collection of files (allowing for use with multiple go-routines). This is useful since we can use the `ReadFile` function to return the content of the files which we ship with the binary.

```go
const listHeight = 14

var (
	titleStyle        = lipgloss.NewStyle().MarginLeft(2)
	itemStyle         = lipgloss.NewStyle().PaddingLeft(4)
	selectedItemStyle = lipgloss.NewStyle().PaddingLeft(2).Foreground(lipgloss.Color("170"))
	paginationStyle   = list.DefaultStyles().PaginationStyle.PaddingLeft(4)
	helpStyle         = list.DefaultStyles().HelpStyle.PaddingLeft(4).PaddingBottom(1)
	quitTextStyle     = lipgloss.NewStyle().Margin(1, 0, 2, 4)
)
```

This is used downstream to change the "style" of the list component from bubbles. The way the API is structured one cannot pass custom styles at the point of creating the component, therefore we must create a new instance of "style" using `lipgloss.NewStyle()` and then pass it to the appropriate variable in  `myList<list>.Style`.

`type item string`

Here we define an item of type String.

`func (i item) FilterValue() string { return "" }`

The `FilterValue()` method is used to filter between each of the items. We need this because it is required by the `list.Item` interface. Even though we will later disable filtering (`l.SetFilteringEnabled(false)`), the interface still requires this method to be implemented. Without it, the code won't compile.

```go
func (d itemDelegate) Height() int                             { return 1 }
func (d itemDelegate) Spacing() int                            { return 0 }
func (d itemDelegate) Update(_ tea.Msg, _ list.Model) tea.Cmd { return nil }
```

The `itemDelegate` struct implements the `list.ItemDelegate`. This interface allows us to customize how individual items are displayed and behave within a list component. 
- `Height()` defines the vertical height of each list item (1 in our case).
- `Spacing()` is used to add blank lines appear between list items. (0 to make it compact).
- `Update()` handles messages specific to individual items. Parameters supplied here are `tea.Msg` and `*list.Model` which we will learn about later.

```go
func (d itemDelegate) Render(w io.Writer, m list.Model, index int, listItem list.Item) {
	i, ok := listItem.(item)
	if !ok {
		return
	}

	str := fmt.Sprintf("%d. %s", index+1, i.title)

	fn := itemStyle.Render
	if index == m.Index() {
		fn = func(s ...string) string {
			return selectedItemStyle.Render("> " + strings.Join(s, " "))
		}
	}

	fmt.Fprint(w, fn(str))
}
```

This contains various parameters of types such as `itemDelegate` has `d` which is the receiver to struct (covered earlier). We supply the output through `io.Writer` where *rendered* text will be written.
- `list.Model` displays the parent list state (which item is selected).

`i, ok := listItem.(item)`
This is Go's type assertion feature. `list.Item` allows us to access fields like `.title` and `.filename`. If value of `i` is returned true, conversion has succeeded otherwise it means the type is wrong.

`str := fmt.Sprintf("%d. %s", index+1, i.title)`
Here, we format the display string for providing us with options. `i.title` is used to display the form of text displayed (Matrix, Great Work, etc). We use `index+1` to add human friendly numbering beginning from 1.

`fn := itemStyle.Render`
We use `.Render` for an item style that takes a string and returns a *styled* string.
`m.Index()` simply returns the currently selected item's index in the list. When selected, `fn` is triggered to add variadic parameter in `s ...string`.

`selectedItemStyle.Render("> " + strings.Join(s, " "))`
`Render` applies the selected item style (purple color + 2 spaces padding). 
Under `strings.Join()`, we simply prepend the arrow to the joined string to produce an output such as : `"> 1. Great Work"`.

`fmt.Fprint(w, fn(str))`
Selects `fn` containing the formatted string and returns a styled version of it. 
`w` writes the string to `io.Writer` for obtaining output.

```go
func (m model) Init() tea.Cmd {
	if m.viewing {
		return tick()
	}
	return nil
}

func tick() tea.Cmd {
	return tea.Tick(time.Millisecond*80, func(t time.Time) tea.Msg {
		return tickMsg(t)
	})
}

```

From a broad view, these two functions work together to implement animation timing in the Bubble Tea framework. They create a recurring "heartbeat" that drives the matrix rain effect.

`return tick` performs on the boolean true result of `m.viewing`. tick starts the animation otherwise returns nil if boolean false.

`tick() tea.Cmd` has a command output structure. It waits 80 milliseconds, sends a `tickMsg` back to the application and triggers next animation frame. 80ms is the sweet spot as it is smooth and consumes less CPU resources. `tickMsg` is a basic signal that says "show next frame".

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.WindowSizeMsg:
		m.width = msg.Width
		m.height = msg.Height
		if m.list.Items() != nil {
			m.list.SetWidth(msg.Width)
		}
		if m.viewing {
			m.initColumns()
		}
		return m, nil
```

We implement the `Update` method here to process every single thing that happens (key press, window resize, timer tick, etc). To figure out the exact kind of message sent, we implement the `switch msg := msg.(type)`. In this particular case, we work with `tea.WindowSizeMsg` focusing on dragging to the corner to make it bigger or smaller via `msg.Width` and `msg Height` which adapts accordingly. Next we check whether it is in `m.viewing` mode, if it is, the Columns are recalculated based on new dimensions. 

```go
case tea.KeyMsg:
		switch keypress := msg.String(); keypress {
		case "q", "ctrl+c":
			if m.viewing {
				// If there's no list (direct mode), quit entirely
				if m.list.Items() == nil {
					m.quitting = true
					return m, tea.Quit
				}
				// Otherwise return to list
				m.viewing = false
				m.choice = ""
				return m, nil
			}
			m.quitting = true
			return m, tea.Quit

		case "enter":
			if !m.viewing && m.list.Items() != nil {
				i, ok := m.list.SelectedItem().(item)
				if ok {
					m.choice = i.filename
					content, err := readAsset(m.choice)
					if err != nil {
						m.content = fmt.Sprintf("Error: %v", err)
					} else {
						m.content = content
					}
					m.viewing = true
					m.initColumns()
					return m, tick()
				}
			}
		}

	case tickMsg:
		if m.viewing {
			m.updateColumns()
			return m, tick()
		}
	}

	var cmd tea.Cmd
	if m.list.Items() != nil {
		m.list, cmd = m.list.Update(msg)
	}
	return m, cmd
}
```

Here we have a set of cases defining various actions.  `case "q", "ctrl+c":` simply quits the instance and clears the screen, but we check whether is being viewed or not first. If the `m.quitting` is true, `View()` shows the quit message `Quitting is for losers!` (which is indeed true).

`case "enter":`
When you press Enter, it first checks  whether it is being viewed and has a list, This makes sure we're in the file selection screen where Enter should actually do something. If both conditions are true, it tries to get the currently selected item from the list using `m.list.SelectedItem().(item)`. That type assertion might fail if something weird happened, we use the `, ok` idiom to safely check. If it successfully got the selected item, now the fun begins. We grab the filename (`i.filename`), try to read the file's content using `readAsset()`, and handle any errors gracefully by storing a `Error : %v` message.

Every 80 milliseconds while the animation is running, it receives a `tickMsg`. This is the cue to advance the animation by one frame. It checks `if m.viewing` to make sure we're still in viewing mode (the user might have pressed 'q' to go back to the list), and if so, it calls `m.updateColumns()` which moves each column's offset forward, creating the scrolling effect.

```go
func (m *model) initColumns() {
	if m.width == 0 || len(m.content) == 0 {
		return
	}

	numCols := m.width / 2
	if numCols < 1 {
		numCols = 1
	}

	m.columns = make([]column, numCols)
	runes := []rune(m.content)

	for i := range m.columns {
		m.columns[i] = column{
			x:      i * 2,
			height: rand.Intn(m.height) + 5,
			offset: rand.Intn(len(runes)),
		}
	}
}

func (m *model) updateColumns() {
	runes := []rune(m.content)
	if len(runes) == 0 {
		return
	}

	for i := range m.columns {
		m.columns[i].offset++
		if m.columns[i].offset >= len(runes) {
			m.columns[i].offset = 0
		}

		if rand.Float32() < 0.02 {
			m.columns[i].height = rand.Intn(m.height) + 5
		}
	}
}
```

`initColumns()` sets up all the columns first -- it's like setting the stage before show begins. For sanity check, we use  `m.width == 0 || len(m.content) == 0`. We then divide the terminal width by 2 (`numCols := m.width / 2`). Why 2? Because we want to space the columns out, creating gaps between cascading characters. 

Next, we allocate memory for all these columns using `make([]column, numCols)`.  We then convert the entire file content into runes `[]rune(m.content)` to properly handle emojis, Chinese, Arabic characters or other Unicode content. 
`rand.Intn(m.height)` gives a random number between 0 and the terminal height, then we add 5 to ensure every column is at least 5 characters tall. The *offset* is also randomized `rand.Intn(len(runes))`. This determines where in the text file each column starts reading.

Then  we loop through every single column created and update it. For each column, we increment its offset by  `m.columns[i].offset++`. Now, if the offset has reached or exceeded the length of the runes array, we wrap it back to zero (`m.columns[i].offset = 0`). This creates an infinite loop through the content. It's like a circular buffer.

Every frame, for each column, we roll the dice with a 2% chance `rand.Float32() < 0.02`. Think about it: `rand.Float32()` gives me a random number between 0.0 and 1.0.

```go
func (m model) View() string {
	if m.viewing {
		return m.matrixView()
	}
	if m.quitting {
		return quitTextStyle.Render("Quitting is for losers.")
	}
	if m.list.Items() != nil {
		return "\n" + m.list.View()
	}
	return ""
}

func (m model) matrixView() string {
	if len(m.columns) == 0 || len(m.content) == 0 {
		return ""
	}

	grid := make([][]rune, m.height)
	for i := range grid {
		grid[i] = make([]rune, m.width)
		for j := range grid[i] {
			grid[i][j] = ' '
		}
	}

	runes := []rune(m.content)

	for _, col := range m.columns {
		if col.x >= m.width {
			continue
		}

		for row := 0; row < col.height && row < m.height; row++ {
			charIdx := (col.offset + row) % len(runes)
			if charIdx < 0 {
				charIdx += len(runes)
			}

			if row < m.height && col.x < m.width {
				grid[row][col.x] = runes[charIdx]
			}
		}
	}

	var result strings.Builder
	for rowIdx, row := range grid {
		for _, char := range row {
			if char != ' ' {
				color := getRandomColor()
				style := lipgloss.NewStyle().Foreground(lipgloss.Color(color))
				result.WriteString(style.Render(string(char)))
			} else {
				result.WriteRune(' ')
			}
		}
		if rowIdx < len(grid)-1 {
			result.WriteString("\n")
		}
	}

	return result.String()
}
```

Further, we proceed to displaying the rain on terminal screen. We achieve this by using `View()` with various conditions. If `m.viewing` is true, the `matrixView()` creates the rain effect, if not we let the user quit because we assume they are a loser xD. If the user is not viewing and not quitting, then they must be in list mode. We check if the list exists `m.list.Items() != nil`, and if so, return the list's view with a newline prepended. 

Now, here's the big idea: We create a 2D grid that represents every single character position on the screen. Think of it like graph paper where each cell can hold one character. We create this grid using `make([][]rune, m.height)` which gives a slice of slices - essentially a 2D array where the first dimension is rows (height) and the second is columns (width).

We iterate through all the columns. For each column, we first check if its x-position is off the screen (`col.x >= m.width`). If someone resized the window smaller after the columns were created, some columns might be positioned beyond the right edge. We skip those with `continue` because trying to draw them would cause an out-of-bounds error.

Next, we initialize this entire grid to spaces. Allocate space for `m.width` and fill all positions with `' '`. This creates a blank canvas. Let's break down `(col.offset + row) % len(runes)`. On adding the offset and row we get a position in the source text. The `%` wraps the index around if it exceeds the text length. This creates that circular buffer effect where columns seamlessly loop through the content.

We then use a `strings.Builder` which is Go's efficient way of building strings piece by piece. If the character is NOT a space, we generate a random color, create a [Lipgloss](https://pkg.go.dev/github.com/charmbracelet/lipgloss#section-readme) style with that color, and render the character with that styling. Finally, `result.String` gives us the beautiful rain effect as a single string!


---
### Extensions
1. So, while almost finishing the initial stages of matrix in go in the back of my mind I thought why not to increase the complexity xD. Here I go (pun intended):
	- [Image to ASCII Negatives](https://arena.ai/c/019c14eb-c963-7bcb-999f-919e56ed8d53)
2. Star Wars intro theme rendered into the terminal.
