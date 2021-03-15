# Sudoku Readme 

## Recursive Backtracking Algorithm

To start with I wrote a basic backtracking depth first search algorithm for my sudoku. See code below.

    def sudoku_solver(sudoku):
        if solve(sudoku):
            return sudoku
        else:
            return np.full((9,9),-1)
        

    # A backtracking solver for sudoku puzzles
    def solve(sudoku):
        if not check_initial_state(sudoku):
            return False
        
        # Loop through every cell in the sudoku
        for r in range(0, 9):
            for c in range(0, 9):
                # If it's 0
                if (sudoku[r][c] == 0):
                    for number in range(1, 10):
                        # Check if any numbers can be put in the sudoku
                        if is_ok(r, c, number, sudoku):
                            # If the number works put it in the soduku
                            sudoku[r][c] = number
                            
                            # Solve the current state of the sudoku
                            if solve(sudoku):
                                return True
                            # Backtrack if it doesn't work and reset values
                            else:
                                sudoku[r][c] = 0
                    # If no possible numbers return false
                    return False
        # If reaches the end 
        return True

The algorithm starts by calling `check_initial_state`. This checks if the sudoku given is invalid i.e. there is two of the same number in the same column etc. If the sudoku is invalid then it simply returns false. This means we don't search the whole game tree to realise the sudoku is unsolvable. I used these helper functions in my final solution, and so the code is visible in "sudoku.ipynb".

It then proceeds to loop through every cell in the sudoku. If the cell is empty i.e. it is equal to 0, the algorithm checks if any numbers in the range 1-9 can be placed in this cell (r,c) without breaking the rules of sudoku. If the number is ok, the algorithm puts that number in (r,c) to create a new sudoku puzzle to solve. Therefore, the algorithm can call `solve` again to search this new sudoku's game tree. 

If the number we put in the cell (r,c) leads to a sudoku where a cell has no numbers that work, then `solve` returns false. The algorithm has to backtrack and reset the value of any changed cells to 0 and try the next possible number for the cell.

If the algorithm ends up looping round every possible number for an empty cell and none work, then the sudoku is unsolvable, and `solve` returns false. If it loops round every cell and manages to find a number that works for all empty cells, then the sudoku is solved and it returns true.

### Problems with Backtracking

The problem with backtracking is that for sudoku's with many empty cells, the search space becomes very large. This meant for the hard sudoku's it could take hundreds of seconds to solve a sudoku.

## Recursive Backtracking with Constraint Propagation

A good way to speed up my backtracking algorithm was to implement constraint propagation which reduces the search space. Whilst my backtracking algorithm did check if a number was allowed in the cell, it didn't use the rules of sudoku to better pick which numbers to put in each cell. If a cell can only have one possible number, maybe because every other cell in the row had a number, you should put that number in that cell, before trying anything else. After doing this other cells in the sudoku might also only have one possible number. By more rationally deciding what numbers to put in each cell, we can solve the sudoku much faster, and reduce the search space greatly.

To do this, I wanted to be able to go through each cell, and *eliminate* any numbers the cell can't be, and store what possible numbers it can be. In this way, instead of a 9x9 grid of numbers, I wanted to store a 9x9 grid of arrays, where each cell stores the possible numbers for the cell.

### Create `Cell` Sudoku

To implement an eliminate function, I wanted to be able to store more information about each cell. Therefore I created a `class Cell` which stored information such as the cell's value, possible numbers and so on. After converting the sudoku into a 9x9 array of `Cell` instead of `int` I could implement my eliminate function. By default, each `Cell` in the cell sudoku had a `possible_numbers` array `[1,2,3,4,5,6,7,8,9]`, except for the cells that already had a number in its cell.

### Eliminate

The eliminate function starts by looping through each cell in the sudoku. Any numbers in that cells column, row, or box was added to an array. These numbers were then removed from the cell's possible numbers.

### Single Value Finder

After running eliminate on the sudoku, the algorithm checked if any cells had only one possible number. If they did, it made the value of that cell the one possible number. After going through every cell, the algorithm then ran eliminate again to update the possible numbers for each cell of the sudoku.

### Only Choice

After giving values to cells with only one possible number, the algorithm can check the sudoku to see if any cell's only have one possible number, even if the length of the possible numbers for all cells is greater than one.

#### Only Choice Row, Col, and Box

To do this the algorithm starts by looping round every cell in the sudoku. If the cell (r,c) was not solved i.e. didn't have a value, the algorithm looped through the cell's possible numbers. It then checked the possible numbers of every other unsolved cell in the row. If the current possible number `i` we are looking at for (r,c) was not a possible number for any other cell in the row, then the value of (r,c) has to be `i`.

After doing this for the row, the algorithm runs `eliminate` and `single_value_finder` again, in case the search space could be reduced further after finding the only choices for the row.

The algorithm worked in a similar way for the columns, and boxes.

### Stalled

Sometimes `only_choice`, `eliminate` and `single_possible_value_finder` can't reduce the search space any further. In these situations, the constraint propagation has stalled. 

To monitor this, the algorithm keeps track of the number of solved values in the sudoku. If after all the different `only_choice` functions, `eliminate` and `single_possible_value_finder` the number of solved values is the same, the constraint propagation is stuck. 

If we end up stuck, the algorithm returns to backtracking. It'll have to guess the value for a cell, and run `solve` again. It is possible that the guessed value is wrong in this case, so the search space increases. However, we can use constraint propagation on the guessed sudoku. Therefore the search space is still far smaller than a standard backtracking search.

### New Search Algorithm

My new algorithm was the code below. At the start of solve the algorithm calls `reduce_sudoku`, to use constraint propagation to reduce the search space. For most sudokus this function can solve the sudoku. In the case that it can't, the algorithm resorts back to backtracking. It guesses a number at an empty cell, and runs `solve` again. 

    def solve(sudoku):
        # Constrain the search space
        reduce_sudoku(sudoku)

        # In case during constraint propagation an invalid value has been put in
        if not check_initial_state(sudoku):
            return False

        # Loop through every cell in the sudoku
        for r in range(0, 9):
            for c in range(0, 9):
                # If it's 0
                if sudoku[r][c] == 0:
                    for number in range(1, 10):
                        # Check if any numbers can be put in the sudoku
                        if is_ok(r, c, number, sudoku):
                            # Create copy of current state of sudoku for backtracking
                            old_sudoku = copy_sudoku(sudoku)

                            # If the number works put it in the soduku as a guess
                            sudoku[r][c] = number

                            # Solve the current state of the sudoku
                            if solve(sudoku):
                                return True
                            # Backtrack if it doesn't work and reset values
                            else:
                                backtrack(sudoku, old_sudoku)
                                # Free up memory space
                                del old_sudoku

                    # If no possible numbers return false
                    return False
        # If reaches the end sudoku is solved
        return True

### Reduce Sudoku Algorithm

My algorithm to reduce the sudoku with constraint propagation is the code below, as described above.

    def reduce_sudoku(sudoku):
        cell_sudoku, solved_values = make_cell_sudoku(sudoku)

        stalled = False
        while not stalled:
            solved_values_before = solved_values

            # Row
            eliminate(cell_sudoku)
            stalled, solved_values = single_possible_value_finder(cell_sudoku, sudoku, solved_values, False)
            eliminate(cell_sudoku)
            only_choice_row(cell_sudoku)

            # Col
            eliminate(cell_sudoku)
            stalled, solved_values = single_possible_value_finder(cell_sudoku, sudoku, solved_values, False)
            eliminate(cell_sudoku)
            only_choice_col(cell_sudoku)

            # Box
            eliminate(cell_sudoku)
            stalled, solved_values = single_possible_value_finder(cell_sudoku, sudoku, solved_values, False)
            eliminate(cell_sudoku)
            only_choice_box(cell_sudoku)

            eliminate(cell_sudoku)
            stalled, solved_values = single_possible_value_finder(cell_sudoku, sudoku, solved_values, True)
            eliminate(cell_sudoku)

            # If we can't do anything then we are stalled
            if not stalled:
                stalled = solved_values == solved_values_before