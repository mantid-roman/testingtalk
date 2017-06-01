# Testing

---

## Writing tests is annoying.
## Can we generate them?

---

### Basic structure of a test

1. Prepare the inputs
1. Pass the inputs to the tested object
1. Get the outputs from the tested object
1. Compare the outputs with "correct" values

---

### Generation workflow

1. Define a function that executes the code to be tested.
1. Define ranges/combinations of inputs to be generated.
1. Compile the generating code.
1. Run the generator, save the generated tests.
1. Compile the tests.

---

### Function executing code to be tested...

... can be generically defined as:

```c++
void run(const Inputs& inp, Outputs &out) {
  // ... set inputs provided by inp
  // ... run the tested code
  // ... pass outputs to out
}
```

---

### Simple example: generator

```c++
// Tested code
double mysqrt(double x) {
  if (x < 0.0) {
    throw std::runtime_error("Negative argument");
  }
  return sqrt(x);
}

// Function that executes the tested code
void run_mysqtr(const Inputs &inp, Outputs &out) {
  double x = inp["x"];
  double y = mysqrt(x);
  out["y"] = y;
}
// Test generator
void generate_test() {
  Generator gen(run_mysqtr, "run_mysqtr");
  gen.addInputShared("x", new DoubleList({2, -1}));
  gen.addOutputUnique("y", new DoubleOutput);
  gen.generate();
}
```

---

### Simple example: generated test

```c++
Outputs out;
out.add("y", new DoubleOutput);

Generator gen;
gen.addInput("x", new DoubleList({2, -1}));
{ // test 1
  Inputs inp = gen.getInputs();
  // inp["x"] = 2;
  run_mysqtr(inp, out);
  TS_ASSERT_EQUALS(out["y"], 1.4142135623730951e+00);
  gen.next();
}

{ // test 2
  Inputs inp = gen.getInputs();
  // inp["x"] = -1;
  // error: Negative argument std::runtime_error
  TS_ASSERT_THROWS(run_mysqtr(inp, out), std::runtime_error);
  gen.next();
}
```

---

### Define preconditions

```c++
// Unexpected handling of bad input
double mysqrt(double x) {
  if (x < 0.0) {
    return 0.0;
  }
  return sqrt(x);
}
// Specify pre-conditions via EXPECT
void run_mysqtr(const Inputs &inp, Outputs &out) {
  double x = inp["x"];
  EXPECT(x >= 0, out, "x cannot be negative");
  double y = mysqrt(x);
  out["y"] = y;
}
```

---

### Generated test

```c++
{ // test 2
  Inputs inp = gen.getInputs();
  // inp["x"] = -1;
  // expected errors:
  //     x cannot be negative
  run_mysqtr(inp, out);
  // Errors expected but no exception was thrown.
  TS_ASSERT_EQUALS(out["y"], 0.0000000000000000e+00);
  gen.next();
}
```

---

### Checking post-conditions

```c++
double mysqrt(double x) {
  if (x < 0.0) {
    throw std::runtime_error("Negative argument");
  }
  return -sqrt(x);
}

void run_mysqtr(const Inputs &inp, Outputs &out) {
  double x = inp["x"];
  EXPECT(x >= 0, out, "x cannot be negative");
  double y = mysqrt(x);
  out["y"] = y;
  EXPECT(y >= 0, out, "Return value cannot be negative");
}
```

---

### Generated test

```c++
{ // test 1
  Inputs inp = gen.getInputs();
  // inp["x"] = 2;
  // expected errors:
  //     Return value cannot be negative
  run_mysqtr(inp, out);
  // Errors expected but no exception was thrown.
  TS_ASSERT_EQUALS(out["y"], -1.4142135623730951e+00);
  gen.next();
}
```

---

### Input generation: direct product

Multiple inputs can be combined in all combinations. Setting

```c++
  Generator gen;
  gen.setInputGenerationMethod(InputGenerator::DirectProduct);
  gen.addInput("a", new IntList({1, 2}));
  gen.addInput("b", new IntList({-1, -2}));
```
will produce all pairs of `a` and `b`:

    (1, -1), (2, -1), (1, -2), (2, -2)

---

### Single input change

If the inputs are independent there is no need to test all their combinations.

Changing one variable at a time could be enough.

```c++
  Generator gen;
  gen.setInputGenerationMethod(InputGenerator::SingleInput);
  gen.addInput("a", new IntList({1, 2, 3}));
  gen.addInput("b", new IntList({-1, -2, -3}));
```
produces

    (1, -1), (2, -1), (3, -1) // change a while b const
    (1, -2), (1, -3)          // change b while a const

---

### Tuples

Strongly correlated inputs can be put into tuples.

```c++
  Generator gen;
  gen.setInputGenerationMethod(InputGenerator::DirectProduct);
  gen.setTuple(
      {gen.addInput("names", new StringArrayList({{"1st"}, {"1st", "2nd"}})),
       gen.addInput("values",   new IntArrayList({{10},    {20,    30}}))});
  gen.addInput("b", new IntList({1, 2}));
```
produces

    ({"1st"}, {10}, 1), ({"1st", "2nd"}, {20, 30}, 1),
    ({"1st"}, {10}, 2), ({"1st", "2nd"}, {20, 30}, 2)
    
---

### Test template

With the help of another type of generator we can generate arbitrary test functions.

```c++
int myfunc(int x) {
  return 2*(x + 1);
}

void run_myfunc(const Inputs &inp, Outputs &out) {
  int x = inp["x"];
  out["y"] = myfunc(x);
}

void generate_test() {
  Generator gen(run_myfunc, "run_myfunc", "myfunc");
  gen.addInput("x", new IntList({1, 2}));
  gen.addOutput("y", new IntOutput);

  TestTemplate t;
  t.addInput("x", "int", "");
  t.addOutput("y", "int", "");
  t.call("int y = myfunc(x)");
  std::cerr << gen.generateFromTemplate(t) << std::endl;
}
```

---

### Generated tests

```c++
void test_myfunc_1() {
  int x = 1;
  int y = myfunc(x);
  TS_ASSERT_EQUALS(y, 4);
}

void test_myfunc_2() {
  int x = 2;
  int y = myfunc(x);
  TS_ASSERT_EQUALS(y, 6);
}
```

---

## Writing generators is annoying.
## Can we generate them?..
