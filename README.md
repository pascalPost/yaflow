# yaflow

Yet Another Flow Solver

# Documentation

The documentation is build based on _jupyter-book_. To build the documentation
run from the repo folder
```
jupyter-book build ./
```

# Tests

Before merge to master, all tests have to pass:

    CPU:
        sanitizers
        valgrind

    GPU:
        cuda-memcheck --report-api-errors all
        cuda-memcheck --tool initcheck
        cuda-memcheck --tool racecheck
        cuda-memcheck --tool synccheck
        cuda-memcheck --leak-check full

