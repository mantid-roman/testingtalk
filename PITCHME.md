# Test generation

---

### Types of test generation

* Code based: code coverage
* Model based: model provides inputs and outputs

---

### Can they be combined?

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
