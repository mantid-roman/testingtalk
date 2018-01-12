# Testing-2

---

## Writing tests is annoying.
## Can we generate them?

---

### Basic structure of a test

1. Prepare the inputs
1. Pass the inputs to the tested object
1. Run executable code of the tested object
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

---

### YAML templates

* Further reduce amouunt of manual work
* Use python for test generation
* Can generate tests for variety of target languages
* Really easy to generate tests for python code

---

### Example

```yaml
config:
    target_test_file: test.py
test:
    name: my_test
    begin: import math
    inputs:
        - name: x
          type: float
          default: 0
    outputs:
        - name: y
          type: float
    run: y = math.sqrt(x)
    cases:
        - name: one
          inputs:
              x: [1, 2, 3]
          outputs: [y] 
        - name: two
          inputs:
              x: [5, 4, 3]
          outputs: [y] 
```

---

### Generating code

```python
gen = PyGenerator(template=yaml.load(template))
suite = gen.generate_test_suite()
formatter = PyGenerator.FormatterClass()
formatter.add_code(suite)
tests = formatter.make_output()
```

---

### Generated tests

```python
import unittest
class Testmy_test(unittest.TestCase):
    def test_one(self):
        import math
        x = 1
        y = math.sqrt(x)
        self.assertEqual(y, 1.0)
    def test_one_1(self):
        import math
        x = 2
        y = math.sqrt(x)
        self.assertEqual(y, 1.4142135623730951)
    def test_one_2(self):
        import math
        x = 3
        y = math.sqrt(x)
        self.assertEqual(y, 1.7320508075688772)
    def test_two(self):
        import math
        x = 5
        y = math.sqrt(x)
        self.assertEqual(y, 2.23606797749979)
    def test_two_1(self):
        import math
        x = 4
        y = math.sqrt(x)
        self.assertEqual(y, 2.0)
    def test_two_2(self):
        import math
        x = 3
        y = math.sqrt(x)
        self.assertEqual(y, 1.7320508075688772)
```

---

### Exceptions

```yaml
run: y = math.sqrt(x)
cases:
    - name: must_be_positive
      inputs:
          x: [-3]
```

---

### Generated test

```python
def test_must_be_positive(self):
    import math
    x = -3
    def run():
        y = math.sqrt(x)
    self.assertRaises(ValueError, run)
```

---

### Exceptions improved

```yaml
run:
    call: math.sqrt
    args: [x]
    returns: [y]
cases:
    - name: must_be_positive
      inputs:
          x: [-3]
```

---

### Generated test

```python
def test_must_be_positive(self):
    import math
    x = -3
    self.assertRaises(ValueError, math.sqrt, x)
```

---

### Pre- and Postconditions

```yaml
run: y = math.sqrt(x)
pre:
    assert: x >= 0
    message: x must be positive
post:
    assert: math.abs(x - y**2) <= 1e-15
    message: y**2 != x
```

---

### Test Mantid algorithm

```yaml
test:
    name: Rebin
    begin: |
        from mantid.simpleapi import *
        from mantid.testing import workspace_helper
        algorithm = AlgorithmManager.create('Rebin')
    inputs:
        - name: InputWorkspace
          type: str
          code: |
              input = workspace_helper.create($value)
              algorithm.setProperty('InputWorkspace', input)
        - name: Params
          type: list(float)
          code: |
              params_str = ','.join([str(p) for p in Params])
              algorithm.setProperty('Params', params_str)
        - names: [PreserveEvents, FullBinsOnly, IgnoreBinErrors]
          type: bool
          code: |
              algorithm.setProperty($qname, $value)
    outputs:
        - name: x_bins
          type: list(float)
          code:
              x_bins = output.readX(0)
        - name: y_bins
          type: list(float)
          code:
              y_bins = output.readY(0)
    run: |
        algorithm.execute()
        output = algorithm.getProperty('OutputWorkspace')
    cases:
        - name: one
          inputs:
              InputWorkspace: [Good, Empty, Bad]
              Params: [[10, 0.1, 20], [10, -0.1, 20]]
              PreserveEvents: [False, True]
              FullBinsOnly: [False, True]
              IgnoreBinErrors: [False, True]
          outputs: [x_bins, y_bins]
```

---

### Using definitions

```yaml
config:
    defs:
        Data1: [1, 2, 3, ...]
        Data2: [10, 20, 30, ...]
test:
    cases:
        - name: one
            inputs:
                x: [$Data1, $Data2]
```

---

### Extending test cases

```yaml
cases:
    - name: base
        inputs:
            x: [$Data1, $Data2]
    - name: extended a
      extends: base
        inputs:
            a: [...]
        outputs: [A1, A2, ...]
    - name: extended b
      extends: base
        inputs:
            b: [...]
        outputs: [B1, B2, ...]
```

---

## Can templates be generated?
