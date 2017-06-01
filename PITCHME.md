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

### Simple example

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
