# Ubiquitous Computing and IoT Lab 3: Neural network on embedded processor

In the third lab of the Ubiquitous Computing and IoT course, we will run a neural network on the STM32 B-U585I-IOT02A discovery kit. You will write the inference code yourself, in plain C, and then make it faster using the data movement ideas we covered in the lecture.

We will use the same development environment as in Lab 1 and Lab 2: **VS Code with STM32CubeCLT and CMake**. If you have not set that up yet, or if you are on a different machine than in the previous labs, work through Assignment 1 Steps 1–4 of the [Lab 1 document](https://github.com/ERNIS-LAB/BCS3120-2026-P1/blob/main/Lab%201/Lab1_Embedded_Computing_and_RTOS.md) first. This lab assumes you already know how to build with `Ctrl+Shift+B`, how to flash with **Run Task → Flash**, how to read the board with the **Serial Monitor**, and how to start a debug session with `F5`.

In Lab 3, you will need to finish 3 assignments. Here are the assignments:

1. Implement a small neural network inference on STM32
2. Implement MNIST neural network inference on STM32
3. Make the MNIST inference faster by reducing data movement

> After finishing each assignment, please ask the **course team** to check your implementation. You will be graded as **"complete" and get full grade** if all 3 assignments in the lab have been completed and checked.
>
> After finishing all the assignments and checked by the **course team**, you will need to submit all your projects in a single compressed file to the Canvas assignment.
>
> *AI Usage Rule: The text of this lab is created with help from Claude Opus 5. As students, you are allowed to use any AI tools to finish the lab. However, you need to disclose the AI tools you use and how you use them when the course team checks your assignments.*

---

## Assignment 1: Implement a small neural network inference on STM32

A neural network layer is not a mysterious thing once you write it out: it is a matrix multiplication followed by an addition, and then a simple function applied element by element. In this assignment you will write exactly that, for a network small enough that you can check every number by hand.

The network here has 2 inputs, 4 hidden neurons, and 3 outputs. It does nothing useful. That is the point — it is small enough to verify, and once it works you will scale the same code up to a real MNIST digit classifier in Assignment 2.

### Step 1: Get the project onto your machine

Download the project from Canvas (**Week 3 / lab3.zip**) and extract it.

The same two warnings from the previous labs apply:

- **Do not put it in Documents, Desktop, or OneDrive.** A syncing folder locks files while the compiler is writing them, and you get Permission denied errors that look like something else.
- **Use a short path**, without spaces or non-English characters.

Open the folder in VS Code, install the recommended extensions if you are prompted, and update the toolchain paths in `.vscode/settings.json` to match your STM32CubeCLT version — exactly as in [Lab 1, Assignment 1, Step 4](https://github.com/ERNIS-LAB/BCS3120-2026-P1/blob/main/Lab%201/Lab1_Embedded_Computing_and_RTOS.md#step-4-open-the-project-in-vs-code-install-the-extensions-and-fix-the-settings-file). If you copy the `settings.json` from a working Lab 1 or Lab 2 project, that works too.

> **Check:** the Explorer panel shows `CMakeLists.txt` at the top level, and the Output panel (**CMake/Build**) ends with `-- Build files have been written to: .../build/Debug`.

### Step 2: Build and flash the unmodified project

Connect the board to **CN8** (the ST-LINK USB connector), press **Ctrl+Shift+B**, then **Run Task → Flash**.

Open the **Serial Monitor** at **115200 baud**.

> **Check:** you see a startup banner:
>
> ```
> Lab 3 ready
> HCLK 160000000 Hz
> timer overhead 3 cycles
> ```
>
> The third line is the project measuring its own stopwatch — Step 3b explains what it means. Your value may be 2, 3 or 4 rather than exactly 3.
>
> If you see nothing, check the baud rate first. If you see garbage characters, the baud rate is wrong rather than the program.

### Step 3: Look at what the project already gives you

Open `Core/Src/main.c`. A few things are already set up for you, and you should know they are there before you start writing code.

#### 3a. `printf` is already connected to the serial port

Near the bottom of `main.c`, in `USER CODE 4`, you will find the same hook you wrote yourself in Lab 1:

```c
int __io_putchar(int ch)
{
  HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
  return ch;
}
```

Two includes are already in the includes section:

```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
#include <string.h>
/* USER CODE END Includes */
```

`<stdio.h>` declares `printf`. `<string.h>` declares `memcpy`, which you will not need until Assignment 3 — it is there so you do not have to come back and add it later.

`setvbuf(stdout, NULL, _IONBF, 0);` is already at the top of `USER CODE 2` so that output arrives steadily instead of in bursts.

One extra thing has been done for you that you would otherwise spend an afternoon on: printing a `float` with `printf("%f", ...)` does not work by default on this toolchain, because the small standard library that microcontroller projects use leaves floating point support out to save space. It is switched back on by one line at the bottom of `CMakeLists.txt`:

```cmake
target_link_options(${CMAKE_PROJECT_NAME} PRIVATE -u _printf_float)
```

Without that line, `printf("%f", 1.5f)` prints nothing at all. You will need `%f` in this assignment, so it is worth knowing where that comes from.

#### 3b. TIM2 is configured as a cycle counter

This lab is about speed, so you need to measure time — and you need to measure it precisely enough that a 20% improvement is visible.

`MX_TIM2_Init()` in this project configures **TIM2** with:

| Setting | Value | Why |
| --- | --- | --- |
| Prescaler | `0` | The timer counts at the full clock speed, so **1 tick = 1 CPU cycle** |
| Counter Period | `4294967295` | TIM2 is a 32-bit timer, so this is its maximum |
| Counter Mode | Up | Counts upward from 0 |

Two consequences worth understanding:

**One tick is one CPU cycle.** The processor runs at 160 MHz, so one tick is 1/160 000 000 s = 6.25 ns. To convert a tick count to microseconds, divide by 160. To convert to milliseconds, divide by 160 000.

**It counts for 26.8 seconds before wrapping around.** A 32-bit counter reaches 4 294 967 295 and then starts again at zero. At 160 MHz that takes about 27 seconds, which is far longer than anything you will measure here, so you never have to worry about it.

The project starts the counter for you in `USER CODE 2`:

```c
HAL_TIM_Base_Start(&htim2);
```

Once started it runs on its own, forever, in the background. You never stop it or reset it — to time something, you read it before and after and subtract.

#### 3c. The timer overhead line

Reading the counter is not free. The read itself is an instruction, and so is storing the result, so even timing *nothing at all* returns a small non-zero number. The project measures this before printing the banner:

```c
uint32_t o0 = TIM2->CNT;
uint32_t o1 = TIM2->CNT;
uint32_t overhead = o1 - o0;
printf("timer overhead %lu cycles\r\n", overhead);
```

`TIM2->CNT` is the timer's count register, read directly. Two reads back to back, with nothing between them — whatever comes out is the cost of the measurement itself, and it is included in every number you will measure from here on.

Three cycles is small enough to ignore in this lab — the inferences you time are hundreds of thousands of cycles long. But knowing the size of your instrument's error is part of measuring anything properly, and if you ever time something very short, this is the number you subtract.

> **Question:** in Assignment 1 you will measure a few hundred cycles per inference, and in Assignment 2 over a million. In which of the two does a 3-cycle error matter more, and roughly what percentage is it in each case?

#### 3d. The `USER CODE` sections you will use

| Section | Where it is | What goes in it |
| --- | --- | --- |
| `USER CODE Includes` | top of the file | `#include` lines |
| `USER CODE PD` | after the includes | `#define` lines (network dimensions) |
| `USER CODE PV` | after that | global variables (network weights) |
| `USER CODE 0` | before `main()` | your functions |
| `USER CODE 2` | inside `main()`, before the loop | setup, **and all the code you write in this lab** |
| `USER CODE 3` | inside `while(1)` | code that runs repeatedly — you will leave this empty |

As always, anything you write outside these markers can be overwritten if the project is regenerated.

### Step 4: Define the network

#### 4a. The shape of the network

The network has **one hidden layer**:

```
   input                          2 values
      |
      |   Layer 1:  W1 [4 x 2] , B1 [4]
      v
   +--------------------------------------+
   |   hidden layer          4 values     |
   |                                      |
   |   then ReLU on those same 4 values:  |
   |   anything negative becomes 0        |
   +--------------------------------------+
      |
      |   Layer 2:  W2 [3 x 4] , B2 [3]
      v
   output                         3 values
```

**ReLU is not a layer.** It has no weights and no biases, and it does not change how many values there are — it takes the 4 hidden values and overwrites any negative one with 0. That is why your `relu` function modifies the array in place and returns nothing.

Note that there is **no ReLU after Layer 2**. The three final values are used exactly as they come out.

Put the dimensions in `USER CODE PD`:

```c
/* USER CODE BEGIN PD */
#define BATCH_SIZE 5
#define N_IN       2
#define N_H        4
#define N_OUT      3
/* USER CODE END PD */
```

#### 4b. The weights, biases and input data

Put these in `USER CODE PV`:

```c
/* USER CODE BEGIN PV */
const float W1[N_H * N_IN] = { 1.0f,  0.5f,
                              -0.1f,  2.0f,
                               0.1f, -0.2f,
                               0.3f,  0.4f };
const float B1[N_H]        = { 0.1f, -0.2f, 0.3f, 0.4f };

const float W2[N_OUT * N_H] = { 1.0f,  0.5f, -0.1f,  0.1f,
                               -0.2f,  0.3f,  1.0f,  0.5f,
                               -0.1f,  0.1f, -0.2f,  0.3f };
const float B2[N_OUT]       = { 0.1f, -0.2f, 0.3f };

const float INPUT_DATA[BATCH_SIZE][N_IN] = {
    { 1.0f,  0.5f},
    { 0.1f, -0.2f},
    { 1.0f,  0.5f},
    { 0.1f, -0.2f},
    { 0.1f, -0.2f}
};
/* USER CODE END PV */
```

#### 4c. How the weight matrix is stored

This is the one detail that causes the most mistakes, so read it carefully.

A weight matrix for a layer has shape **[output dimension, input dimension]**. For Layer 1 that is [4, 2]: four rows, one per output neuron, each row holding the two weights that neuron applies to the two inputs.

C has no natural way to pass a 2D array around, so the matrix is **flattened**: the rows are written out one after another into a single 1D array. This is called row-major order.

So `W1` written as a matrix looks like this:

|             | input 0 | input 1 |
| ----------- | ------- | ------- |
| **neuron 0** | 1.0     | 0.5     |
| **neuron 1** | -0.1    | 2.0     |
| **neuron 2** | 0.1     | -0.2    |
| **neuron 3** | 0.3     | 0.4     |

and stored in memory as `{1.0, 0.5, -0.1, 2.0, 0.1, -0.2, 0.3, 0.4}` — that is exactly how it is written above, one row per line.

To get the weight connecting **input `j`** to **output neuron `i`**, in a matrix with `N` inputs per row:

```
W[i * N + j]
```

`i * N` skips over the first `i` complete rows, and `+ j` picks the right entry inside row `i`.

> **Question:** what is `W1[5]`? Work out which neuron and which input it belongs to using the formula, then check it against the table above.

#### 4d. Why everything is `const`

Every array above is declared `const`. That is not just good style — it decides **where the data physically lives**.

A `const` global array is placed in **flash memory**: the 2 MB of non-volatile storage that also holds your program. It stays there when the power is off, and it is read in place. Without `const`, the array would instead be copied into **SRAM** at startup, so it would occupy flash *and* RAM at the same time.

For eight weights that makes no difference. In Assignment 2 you will have 101 770 of them, and it matters a great deal.

> **Check this for yourself.** Add this line temporarily to `USER CODE 2` and flash:
>
> ```c
> printf("W1 at %p, INPUT_DATA at %p\r\n", (void*)W1, (void*)INPUT_DATA);
> ```
>
> An address starting with `0x0800` is flash. An address starting with `0x2000` is SRAM. Both should be flash addresses.

### Step 5: Write the three functions

Add these to `USER CODE 0`. Write them in this order, because each one uses the previous.

#### 5a. `dense` — one fully connected layer

A dense (fully connected) layer computes **Y = W·X + B**. Written out for a single output neuron `i`:

```
Y[i] = B[i] + W[i][0]*X[0] + W[i][1]*X[1] + ... + W[i][N-1]*X[N-1]
```

Each output neuron takes its own row of `W`, multiplies it element by element with the input vector, adds everything up, adds its bias, and that is the output.

The function signature:

```c
static void dense(const float *W, const float *B, const float *X,
                  float *Y, int M, int N)
```

| Parameter | Meaning |
| --- | --- |
| `W` | flattened weight matrix of the layer, shape [M, N] |
| `B` | bias vector of the layer, length M |
| `X` | input vector of the layer, length N |
| `Y` | output buffer of the layer, length M — **you write your result here** |
| `M` | output dimension (number of neurons in this layer) |
| `N` | input dimension |

Fill in this template:

```c
static void dense(const float *W, const float *B, const float *X,
                  float *Y, int M, int N)
{
    for (int i = 0; i < M; i++)      /* one output neuron at a time */
    {
        /* 1. Start this output off at its bias value:
         *       Y[i] = B[i];
         *
         * 2. Then loop over the N inputs. For each input j, multiply the
         *    right weight by the right input and add the product into
         *    the output you are building up:
         *
         *       Y[i] = Y[i] + W[...] * X[j];
         *
         *    Use the index formula from Step 4c to pick the weight.
         */
    }
}
```

> **Note:** `Y` is the buffer the caller gave you. You are building each output value up inside it, one product at a time. Do not create a second array inside the function — write directly into `Y`.

#### 5b. `relu` — the activation function

ReLU stands for Rectified Linear Unit, and it is much simpler than the name suggests: negative values become zero, everything else is left alone.

```
relu(x) = max(0, x)
```

It modifies the array **in place**, which is why it returns nothing:

```c
static void relu(float *x, int len)
{
    /* For each of the len elements: if it is negative, set it to zero. */
}
```

#### 5c. `nn_infer` — the whole network

This runs one input sample through both layers:

```c
void nn_infer(const float *in, float *out)
{
    /* You need a temporary array to hold the hidden layer values:
     *
     *     float h[N_H];
     *
     * Then, in order:
     *   1. dense(...)  from `in` into `h`      -- Layer 1
     *   2. relu(...)   on `h`                  -- activation
     *   3. dense(...)  from `h` into `out`     -- Layer 2
     *
     * Think carefully about which M and N each dense call needs.
     */
}
```

> **Questions before you build:**
>
> - Layer 1 has `M = N_H` and `N = N_IN`. What are `M` and `N` for Layer 2, and why are they not the same?
> - Why is there no `relu` call after the second `dense`?

### Step 6: Run the inference and time it

Now you will run all 5 input samples through the network, print the results, and measure how long each inference takes.

#### 6a. The timer is already running

You do not need to start anything. As you saw in Step 3b, `USER CODE 2` already contains:

```c
HAL_TIM_Base_Start(&htim2);
```

and it has been counting since the board booted. All you have to do is read it.

#### 6b. How to read the timer

Reading the counter looks like this:

```c
uint32_t t0 = TIM2->CNT;

/* ... the code you want to measure ... */

uint32_t t1 = TIM2->CNT;

uint32_t cycles = t1 - t0;
```

`TIM2->CNT` is the timer's count register — the same read the project used in Step 3c. `cycles` is then the number of CPU cycles the measured code took.

> You may also see `__HAL_TIM_GET_COUNTER(&htim2)` in ST examples and online. It is a HAL macro that expands to exactly the same register read. Either works; this lab uses `TIM2->CNT` because it is shorter and makes it obvious that you are reading hardware.

To turn cycles into a time:

```c
float microseconds = cycles / 160.0f;      /* 160 MHz -> 160 cycles per us */
```

> **Note on subtraction:** because both values are `uint32_t`, the subtraction still gives the right answer even in the rare case where the counter wrapped around between the two reads. You do not need to handle that specially.

> **Note on the overhead:** every `cycles` value you compute this way includes the few cycles of measurement overhead from Step 3c. You can subtract it if you want an exact figure, but for everything in this lab it is far too small to matter — leave it in, and remember it is there.

#### 6c. Write the inference loop

Put your loop in `USER CODE 2`, **after** the lines that are already there and **before** the `while(1)` loop. The test runs once, at startup, and then the program falls into an empty `while(1)` and sits there.

> **Why not inside `while(1)`?** Because the measurement would then be repeated forever, and the serial output would scroll past faster than you can read it. Everything in this lab is a one-shot experiment: run it, read the numbers, change something, flash again. If you want to see the results a second time, press the black reset button on the board.

Fill in this template:

```c
/* USER CODE BEGIN 2 */
/* ... the lines already in the project stay here ... */

for (int i = 0; i < BATCH_SIZE; i++)
{
    float output[N_OUT];

    /* 1. Read the timer into t0.
     *
     * 2. Run one inference:
     *       nn_infer(INPUT_DATA[i], output);
     *
     *    (INPUT_DATA[i] is row i of the 2D array, which is exactly the
     *     pointer to N_IN floats that nn_infer expects.)
     *
     * 3. Read the timer into t1.
     *
     * 4. Print the sample number, then all N_OUT output values, then the
     *    number of cycles. Use %f for the floats and %lu for the cycles.
     */
}
/* USER CODE END 2 */
```

Leave `USER CODE 3`, the body of `while(1)`, empty.

> **Hints:**
>
> - `float output[N_OUT];` inside the loop gives you a fresh output buffer for each sample.
> - To print the three output values you need a second, inner `for` loop over `j`.
> - `printf` with no `\n` does not move to a new line, so you can build one line out of several `printf` calls and end it with `printf("\r\n")`.
> - `%lu` is the format for `uint32_t`. Using `%d` will produce warnings and may print the wrong value.

### Step 7: Check your results

Build, flash, and open the Serial Monitor.

> **Check:** after the startup banner you see five lines, printed once. The three output numbers must be exactly these:
>
> ```
> Sample 0  1.860000 0.490000 0.445000  | 425 cycles (2.66 us)
> Sample 1  0.200000 0.305000 0.325000  | 332 cycles (2.08 us)
> Sample 2  1.860000 0.490000 0.445000  | 332 cycles (2.08 us)
> Sample 3  0.200000 0.305000 0.325000  | 318 cycles (1.99 us)
> Sample 4  0.200000 0.305000 0.325000  | 318 cycles (1.99 us)
> ```
>
> **The output values must match exactly.** The cycle counts will not — they depend on your compiler version and will vary a little from run to run. Anything in the range of roughly 300 to 500 cycles is normal.

Samples 0 and 2 are the same input, and so are samples 1, 3 and 4. That is deliberate: identical inputs must give identical outputs, and it gives you a free consistency check.

> **Question — why is sample 0 the slowest?** It runs the same 20 multiply-accumulates as sample 2, on the same input values, yet it takes noticeably longer. Nothing about the arithmetic differs.
>
> The reason is that sample 0 is the **first time** this code has ever run. The instructions of `dense`, `relu` and `nn_infer`, and the weights themselves, are all sitting in flash and have never been read before. The processor has to fetch them the slow way. By the time sample 1 runs, they have been fetched once already and the next read is much cheaper.
>
> The effect is large here only because the whole inference is tiny — 425 against 332 cycles is a 28% difference. In Assignment 2 each inference is thousands of times longer, so the same fixed cost becomes invisible. For now, just notice that **the first measurement of anything is not representative**, and that a careful benchmark either discards it or runs the thing once before starting the clock.

**If your numbers are wrong, the pattern tells you where to look:**

| What you see | Most likely cause |
| --- | --- |
| Nothing prints, or the line stops before the numbers | `-u _printf_float` is missing from `CMakeLists.txt`, so `%f` prints nothing |
| All three outputs are the same for every sample | You are ignoring `X` or `i` somewhere in `dense` |
| Sample 1 is wrong but sample 0 is right | Your `relu` is not doing anything. Sample 1 is the only one whose hidden layer contains a negative value before activation |
| Everything is wrong, values are huge or `nan` | The weight index formula, or `h` is used before it is written |
| Values slightly off in the last decimal | Not a bug. Floating point addition is not exactly associative |
| Outputs correct, cycles is a huge number like 4294967290 | You subtracted in the wrong order (`t0 - t1`) |

> **(Check with the course team when you finish this assignment)**

---

## Assignment 2: Implement MNIST neural network inference on STM32

The network in Assignment 1 does nothing useful. In this assignment you will run exactly the same code on a real one: a classifier trained in PyTorch on MNIST, the standard dataset of handwritten digits.

The interesting part is how little has to change. The network is 12 000 times larger, but the maths is identical, and if you wrote Assignment 1 in a general enough way, the functions themselves need no editing at all.

**Before you start:** make a copy of your Assignment 1 project folder and call it `lab3_assignment1`. Keep working in the original folder. You hand in each assignment separately, and you will want a known-good Assignment 1 to fall back on.

### Step 1: Add the network to the project

Download **Week 3 / lab3_materials.zip** from Canvas and unzip it. It contains two files:

| File | Contents |
| --- | --- |
| `mlp_params.h` | the trained weights and biases |
| `mnist_data.h` | ten test images and their correct labels |

Copy both into `Core/Inc/` in your project. The folder should then look like this:

```
Core
├── Inc
│   ├── main.h
│   ├── mlp_params.h        <- new
│   ├── mnist_data.h        <- new
│   ├── stm32u5xx_hal_conf.h
│   └── stm32u5xx_it.h
├── Src
└── Startup
```

### Step 2: Look inside the two header files

Open them. They are very long — over 12 000 lines between them — but the structure is simple, and you need to know exactly what names they define, because you will be typing those names into your own code.

#### 2a. `mlp_params.h`

Scroll to the top. Everything you need is in the first 15 lines:

```c
#define FC1_IN_DIM   784        /* 28 x 28 pixels, flattened into one vector */
#define FC1_OUT_DIM  128        /* hidden neurons */
#define FC1_W_SIZE   100352     /* = 784 * 128 */
#define FC1_B_SIZE   128

#define FC2_IN_DIM   128
#define FC2_OUT_DIM  10         /* one output per digit, 0 to 9 */
#define FC2_W_SIZE   1280       /* = 128 * 10 */
#define FC2_B_SIZE   10

static const float fc1_weight_flat[FC1_W_SIZE];
static const float fc1_bias_flat[FC1_B_SIZE];
static const float fc2_weight_flat[FC2_W_SIZE];
static const float fc2_bias_flat[FC2_B_SIZE];
```

`FC` stands for "fully connected", which is another name for the dense layer you already wrote. **FC1 is Layer 1 and FC2 is Layer 2.**

Note the naming carefully, because it is different from Assignment 1:

| Assignment 1 | MNIST network |
| --- | --- |
| `N_IN` | `FC1_IN_DIM` |
| `N_H` | `FC1_OUT_DIM`, and also `FC2_IN_DIM` — they are the same number |
| `N_OUT` | `FC2_OUT_DIM` |
| `W1` | `fc1_weight_flat` |
| `B1` | `fc1_bias_flat` |
| `W2` | `fc2_weight_flat` |
| `B2` | `fc2_bias_flat` |

> **Question:** `FC1_OUT_DIM` and `FC2_IN_DIM` are both 128. Why must they always be equal, no matter how the network is designed?

`_flat` in the array names is a reminder that the weight matrices are **flattened row-major**, exactly as in Step 4c of Assignment 1. `fc1_weight_flat` is the [128, 784] matrix written out one row at a time, so the weight from input pixel `j` to hidden neuron `i` is:

```c
fc1_weight_flat[i * FC1_IN_DIM + j]
```

Same formula you already used. Only the numbers are bigger.

#### 2b. `mnist_data.h`

```c
#define BATCH_SIZE 10
#define MNIST_DIM  784

static const float MNIST_DATA[BATCH_SIZE][MNIST_DIM];   /* ten test images */
static const int   MNIST_LABELS[BATCH_SIZE];            /* the correct digit for each */
```

**Each image is already a flat vector of 784 floats.** A 28×28 image has been unrolled row by row into one long list, and each pixel has been scaled to the range 0.0 to 1.0 — the same scaling used when the network was trained. Your code does no image processing at all; the numbers go straight into the network.

Scroll to the bottom of the file and you will find the answers:

```c
static const int MNIST_LABELS[BATCH_SIZE] = {
    7, 2, 1, 0, 4, 1, 4, 9, 5, 9
};
```

These are the first ten images of the standard MNIST test set, so those are the digits a human sees. Your network has never been shown them during training.

> **Question:** how many weights and biases are there in the whole network? Add up `FC1_W_SIZE + FC1_B_SIZE + FC2_W_SIZE + FC2_B_SIZE`. Write the number down — you will need it in Step 3c and again in Assignment 3.

### Step 3: Wire the headers into `main.c`

#### 3a. Include them

```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
#include <string.h>
#include "mlp_params.h"
#include "mnist_data.h"
/* USER CODE END Includes */
```

#### 3b. Remove the Assignment 1 definitions

Your Assignment 1 network is gone. Comment out the whole of `USER CODE PD`:

```c
/* USER CODE BEGIN PD */
/* Dimensions now come from mlp_params.h and mnist_data.h.
 * Assignment 1 values, kept for reference:
 *   #define BATCH_SIZE 5
 *   #define N_IN 2
 *   #define N_H  4
 *   #define N_OUT 3
 */
/* USER CODE END PD */
```

and the whole of `USER CODE PV` — `W1`, `B1`, `W2`, `B2` and `INPUT_DATA` are all replaced by the arrays in `mlp_params.h`.

> **`BATCH_SIZE` in particular must go.** `mnist_data.h` defines it as 10 and yours says 5. If you leave both in, the compiler warns about a redefinition and you end up running whichever one happened to win — a bug that produces plausible but wrong results rather than an error.

Finally, comment out the Assignment 1 inference loop in `USER CODE 2`. It uses `INPUT_DATA`, which no longer exists. Leave the project's own startup lines — the banner, the timer start, the overhead measurement — where they are.

#### 3c. Update `nn_infer` to the new names

You have just deleted `N_IN`, `N_H`, `N_OUT`, `W1`, `B1`, `W2` and `B2`. Your `nn_infer` still refers to every one of them, so the project will not compile until you fix it.

The replacements come straight from the table in Step 2a:

| `nn_infer` used | now write |
| --- | --- |
| `float h[N_H];` | `float h[FC1_OUT_DIM];` |
| `W1`, `B1` | `fc1_weight_flat`, `fc1_bias_flat` |
| `N_H`, `N_IN` (Layer 1's `M` and `N`) | `FC1_OUT_DIM`, `FC1_IN_DIM` |
| `W2`, `B2` | `fc2_weight_flat`, `fc2_bias_flat` |
| `N_OUT`, `N_H` (Layer 2's `M` and `N`) | `FC2_OUT_DIM`, `FC2_IN_DIM` |

Step 4 goes through this in detail. Make the substitutions now so that you can build.

**Your `dense` and `relu` functions need no change at all.** They take the dimensions as arguments, so they already work for any layer size.

#### 3d. Build it

Build now, before writing any new code, so that any remaining name problems surface on their own.

> **If you see `'N_H' undeclared`** or a similar message naming `W1`, `N_IN`, `INPUT_DATA` or `BATCH_SIZE`, you missed one of the substitutions above. The error message names the identifier and the line — go there and replace it.

> **Check:** the build succeeds, and the size table looks something like this:
>
> ```
> Memory region         Used Size  Region Size  %age Used
>              RAM:        4728 B       768 KB      0.60%
>              ROM:       54616 B         2 MB      2.60%
>            SRAM4:           0 B        16 KB      0.00%
> ```

Now look at that ROM figure and think about it, because it should bother you.

You just added 400 KB of weights to the project, and ROM barely moved. Where did they go?

**They are not in the binary.** Nothing calls `nn_infer` yet — you commented the Assignment 1 loop out in Step 3b — so as far as the linker can tell, `nn_infer` is dead code, and the weight arrays are data that nothing reads. The toolchain removes both. Every function and every array is placed in its own section, and at link time anything unreachable is discarded, so your program does not carry code or data it never uses.

This is normally invisible and helpful. Here it matters, because it means **the size table is not measuring what you think it is measuring until the code actually runs.** You will look at it again in Step 6, once the inference loop exists, and the number will be very different.

> **RAM is 4728 B and should stay roughly there** all through this assignment. If RAM ever jumps by 400 KB, a `const` has gone missing — see Assignment 1 Step 4d.

### Step 4: Adapt the inference

#### 4a. Check your `nn_infer`

You made the substitutions in Step 3c to get the project building. Now check that what you wrote is actually right, because a project that compiles is not the same as a network that computes the correct thing.

Your `nn_infer` should follow this structure:

```c
void nn_infer(const float *in, float *out)
{
    /* 1. Declare the hidden layer array. It now holds FC1_OUT_DIM floats.
     *
     * 2. Layer 1:  dense(...) from `in` into the hidden array.
     *              Use fc1_weight_flat and fc1_bias_flat.
     *              M = FC1_OUT_DIM,  N = FC1_IN_DIM
     *
     * 3. ReLU on the hidden array, length FC1_OUT_DIM.
     *
     * 4. Layer 2:  dense(...) from the hidden array into `out`.
     *              Use fc2_weight_flat and fc2_bias_flat.
     *              M = FC2_OUT_DIM,  N = FC2_IN_DIM
     */
}
```

> **Watch the `M` and `N` of the second call.** A very common mistake is to pass `FC2_OUT_DIM` and `FC1_IN_DIM`, mixing up the two layers. Remember what the two arguments mean: `M` is how many values come *out* of this layer, `N` is how many go *in*. Both of those wrong versions compile without a single warning.

One thing worth noticing: the hidden array is now `FC1_OUT_DIM` floats — 128 of them, so 512 bytes — where in Assignment 1 it was 4 floats, or 16 bytes. It is a local variable, so it lives on the stack. That is fine on this board, but on a smaller microcontroller a 512-byte local array is the kind of thing that silently overflows the stack.

#### 4b. Write `argmax`

The network outputs 10 numbers, one per digit. The predicted digit is the **position** of the largest one, not the value. Add this to `USER CODE 0`:

```c
static int argmax(const float *x, int n)
{
    /* Return the index of the largest element of x.
     *
     * Start by assuming element 0 is the largest, remembering both its
     * index and its value. Then walk through the rest, and whenever you
     * find something bigger, update both.
     */
}
```

> **Note:** the output values are not probabilities and do not add up to 1. They are raw scores, and they can be negative. Comparing them is still valid — the largest score wins — but do not initialise your "best value so far" to `0`, because a sample where every score is negative would then give the wrong answer.

### Step 5: Write the MNIST inference loop

Replace your loop from Assignment 1. It stays in the same place — `USER CODE 2`, before the `while(1)` loop. The structure is the same, but you now print the **predicted digit**, the **correct label**, and the **time**.

```c
/* USER CODE BEGIN 2 */
/* ... the lines already in the project stay here ... */

float output[FC2_OUT_DIM];

for (int i = 0; i < BATCH_SIZE; i++)
{
    /* 1. Read the timer into t0.
     * 2. nn_infer(MNIST_DATA[i], output);
     * 3. Read the timer into t1.
     *
     * 4. Get the predicted digit with argmax(output, FC2_OUT_DIM).
     *    The correct answer is MNIST_LABELS[i], which is an int, so
     *    print it with %d.
     *
     * 5. Print:  sample number, prediction, label, cycles, milliseconds.
     *    Milliseconds = cycles / 160000.0f
     */
}
/* USER CODE END 2 */
```

Also count how many of the ten the network got right, and print that after the loop. It is two extra lines and it makes the result much easier to read at a glance.

### Step 6: Check your results

> **Check:** your output should look similar to this:
>
> ```
> Sample 0  Pred: 7  Label: 7  | 991122 cycles (6.19 ms)
> Sample 1  Pred: 2  Label: 2  | 974366 cycles (6.09 ms)
> Sample 2  Pred: 1  Label: 1  | 990896 cycles (6.19 ms)
> Sample 3  Pred: 0  Label: 0  | 974234 cycles (6.09 ms)
> Sample 4  Pred: 4  Label: 4  | 975128 cycles (6.09 ms)
> Sample 5  Pred: 1  Label: 1  | 990966 cycles (6.19 ms)
> Sample 6  Pred: 4  Label: 4  | 974329 cycles (6.09 ms)
> Sample 7  Pred: 9  Label: 9  | 990887 cycles (6.19 ms)
> Sample 8  Pred: 5  Label: 5  | 975147 cycles (6.09 ms)
> Sample 9  Pred: 9  Label: 9  | 974563 cycles (6.09 ms)
> Accuracy 10/10
> ```
>
> **The exact cycle counts will differ on your build** — anything of roughly this order, around a million cycles per inference, is fine. What matters is that you get all ten digits right and that every inference takes about the same time.

Ten out of ten. This network scores about 97% on the full 10 000-image test set, so a run of ten correct is likely but not guaranteed — a different set of ten images would probably contain one it gets wrong.

**If any prediction is wrong**, that is a real problem — this network gets all ten. The most common causes:

| Symptom | Cause |
| --- | --- |
| Every prediction is the same digit | `argmax` always returns 0, or `output` is not being filled |
| Predictions look random | Weight indexing wrong — check `W[i * N + j]`, and that the second `dense` call uses `FC2_IN_DIM` as its input dimension |
| All outputs are `nan` | `h` is read before being written, or the two `dense` calls have their arguments swapped |
| Accuracy is fine but time is under 1000 cycles | The compiler removed your inference because nothing uses the result. Make sure `output` really feeds into `argmax` and `printf` |

### Step 7: Look at the size table again

Scroll back through the build output to the memory table. It should now look very different from the one in Step 3d:

```
Memory region         Used Size  Region Size  %age Used
             RAM:        ?????? B      768 KB      ?.??%
             ROM:        ?????? B        2 MB     ??.??%
```

> **Questions:**
>
> 1. By roughly how much did ROM grow between Step 3d and now? You added an inference loop and an `argmax` function — perhaps fifty lines of code. Does fifty lines of code explain the growth?
> 2. Take the number of parameters you counted in Step 2 and multiply by 4 bytes per `float`. Compare it with the growth you just measured.
> 3. In Step 3d the weights were discarded because nothing used them. What changed?

The weights only enter the binary once something actually reads them. That is worth remembering: **the size of your program depends on what your code reaches, not on what you wrote.**

> **RAM should still be small** — a few kilobytes. The weights are `const`, so they stay in flash and are read from there. Assignment 3 is about what that costs you.

### Step 8: How much work is this, really?

Before you finish, work out these three numbers and keep them for Assignment 3.

> **Questions:**
>
> 1. A **MAC** (multiply-accumulate) is one `weight * input` multiplication plus one addition. How many MACs does one inference perform? Layer 1 does `FC1_W_SIZE` of them and Layer 2 does `FC2_W_SIZE` — the weight counts you already have from Step 2.
> 2. This processor's floating-point unit can complete **one MAC per clock cycle** at best. So what is the smallest number of cycles this inference could possibly take?
> 3. Compare that with what you measured. How many times slower is your code than the theoretical best?

The gap you just calculated is what Assignment 3 is about.

> **(Check with the course team when you finish this assignment)**

---

## Assignment 3: Make the MNIST inference faster by reducing data movement

Your inference does around 100 000 multiply-accumulates and takes roughly a million cycles — about 10 cycles for every multiply-accumulate. The arithmetic is not the problem — the processor could do one MAC per cycle. The time is going somewhere else, and that somewhere else is **moving data around**.

In this assignment you will count the data movement, then remove some of it, twice, and measure what each change buys you.

**Before you start:** make a copy of your Assignment 2 project folder and call it `lab3_assignment2`. Keep working in the original folder.

### Step 1: Record your baseline

Before changing anything, write down the cycle count you measured in Assignment 2. Every result in this assignment is a comparison against that number, so you need it written down rather than remembered.

Use the table at the end of this assignment to record your results as you go.

### Step 2: Count the weight movement

Every weight in the network has to travel from where it is stored to the processor before it can be multiplied by anything. That journey is the data movement we are talking about.

> **Questions — work these out on paper before continuing:**
>
> 1. During **one inference**, how many times is each individual weight read? Look at your `dense` function and trace what happens to `W[i * N + j]` as the loops run.
> 2. How many weight reads happen in Layer 1 in total? And in Layer 2?
> 3. How many weight reads in one complete inference?
> 4. Each weight is a `float`, which is 4 bytes. **How many bytes of weights are read per inference?**
> 5. Your loop runs 10 samples. How many bytes is that in total?

You should end up with a number in the region of 400 kilobytes moved for a *single* classification of a *single* 28×28 image. That is the scale of the problem.

### Step 3: Where those bytes come from

Your weights are `const`, so they live in **flash memory** — you confirmed this in Assignment 1 Step 4d.

Flash is non-volatile and large, but it is **slow**. The processor runs at 160 MHz, and the flash cannot keep up with that. To compensate, the memory controller inserts **wait states**: cycles where the processor does nothing but wait for the data to arrive.

Find out how many your board is using. Add this to `USER CODE 2`:

```c
printf("flash latency = %lu wait states\r\n",
       (FLASH->ACR & FLASH_ACR_LATENCY) >> FLASH_ACR_LATENCY_Pos);
```

**SRAM has no wait states.** The processor reads it at full speed. This board has 768 KB of it, and you are currently using almost none.

> **Question:** you found in Step 2 that roughly 400 KB of weights are read per inference. Would they fit in 768 KB of SRAM?

### Step 4: Optimization 1 — move the weights from flash to SRAM

The idea is simple: copy the weights into SRAM once at startup, then read them from SRAM during every inference. The copy itself costs time, but you pay it once instead of on every inference.

#### 4a. Create the SRAM buffers

In `USER CODE PV`. Note that these are **not** `const` — that is the whole point, they must live in RAM:

```c
/* USER CODE BEGIN PV */
float fc1_weight_ram[FC1_W_SIZE];
float fc1_bias_ram[FC1_B_SIZE];
float fc2_weight_ram[FC2_W_SIZE];
float fc2_bias_ram[FC2_B_SIZE];
/* USER CODE END PV */
```

#### 4b. Copy the weights once, before the inference loop

`memcpy` copies a block of bytes from one place to another. It is declared in `<string.h>`, which the project already includes for you (Assignment 1, Step 3a).

**Placement matters here.** Your inference loop is in `USER CODE 2`. The copy has to happen **above it**, so that the weights are already in SRAM by the time the first inference runs — and it must be **outside** the loop, so you pay for it once rather than ten times.

```c
/* USER CODE BEGIN 2 */
/* ... banner, timer start, overhead measurement ... */

memcpy(fc1_weight_ram, fc1_weight_flat, sizeof(fc1_weight_ram));
memcpy(fc1_bias_ram,   fc1_bias_flat,   sizeof(fc1_bias_ram));
memcpy(fc2_weight_ram, fc2_weight_flat, sizeof(fc2_weight_ram));
memcpy(fc2_bias_ram,   fc2_bias_flat,   sizeof(fc2_bias_ram));
printf("weights copied to SRAM\r\n");

for (int i = 0; i < BATCH_SIZE; i++)
{
    /* ... your inference loop, unchanged ... */
}
/* USER CODE END 2 */
```

> **If you put the copies inside the loop**, every inference would first move 400 KB and then read it back — slower than never copying at all. The measured time would go *up*, and the printed message would appear ten times, which is the clue that tells you what happened.

> **Use `sizeof(fc1_weight_ram)`, not a number you worked out yourself.** `sizeof` gives the size of the array **in bytes** and stays correct if the network ever changes. Writing `FC1_W_SIZE` instead would copy only a quarter of the data, because that is a count of *floats*, not of *bytes* — and the resulting bug produces plausible-looking wrong answers rather than a crash.

#### 4c. Point the inference at the SRAM copies

Change `nn_infer` to pass `fc1_weight_ram`, `fc1_bias_ram`, `fc2_weight_ram` and `fc2_bias_ram` to `dense` instead of the `_flat` arrays. The `M` and `N` arguments do not change — only where the numbers are read from.

#### 4d. Measure

Build, flash, and record the new cycle count.

> **Check:** the predictions must be **exactly the same** as before — all ten correct. You changed where the numbers are stored, not what they are. If any prediction changed, something is wrong with your `memcpy` calls, and the most likely culprit is a `sizeof` you replaced with a float count.

> **Check the size table too:**
>
> ```
> RAM:      411808 B       768 KB     52.36%
> ```
>
> RAM went from about 4.7 KB to over 400 KB — you are now using more than half the memory on the chip. This optimization is not free: you bought speed with memory, and you can only do it because the network happens to fit.

> **Questions:**
>
> 1. Did the **number** of weight reads per inference change? Recount using your answer from Step 2.
> 2. If the number did not change, what did?
> 3. The weights are now stored **twice** — once in flash, once in SRAM. Look at the ROM figure as well as the RAM figure. Did ROM go down? Why not?
> 4. The `memcpy` moves 400 KB once, at startup, and you then run ten inferences. Roughly how many inferences do you need to run before the copy has paid for itself?

### Step 5: Count the output movement

Now look at a different stream of data. Open your `dense` function and look at this line:

```c
Y[i] = Y[i] + W[i * N + j] * X[j];
```

Three arrays are touched here. You have already dealt with `W`. Now think about **`X`, the input vector**, which lives in SRAM.

The outer loop runs once per output neuron. Inside it, the inner loop walks the entire input vector from `X[0]` to `X[N-1]`. So the whole input is read again from the start for **every single output neuron**.

> **Questions — work these out before continuing:**
>
> 1. Layer 1 has `N = 784` inputs and `M = 128` outputs. How many times is `X[0]` read during Layer 1?
> 2. How many reads of `X` happen in Layer 1 in total? And in Layer 2?
> 3. How many **distinct** input values are there across both layers? (784 pixels, plus 128 hidden values.)
> 4. Compare your answers to questions 2 and 3. Each input value is read many times over, but it never changes. That repetition is pure waste — and unlike the weights, it is waste you can actually remove.

### Step 6: Optimization 2 — compute four outputs at once

#### 6a. The idea

The processor has a small number of extremely fast storage locations inside the core itself, called **registers**. A value in a register costs nothing to read — it is part of the instruction. The FPU on this chip has 32 of them for floating-point values, and they are the only storage faster than SRAM.

At the moment your inner loop loads one input value, uses it once, and throws it away. The next output neuron loads exactly the same value again.

Instead, **keep four output neurons in registers at the same time**. Then each input value you load can be used four times before you move on:

```
current version              four outputs at once

load X[j]                    load X[j]
  use for output 0             use for output 0
                               use for output 1
load X[j] again                use for output 2
  use for output 1             use for output 3
                             (done -- move to X[j+1])
load X[j] again
  use for output 2

load X[j] again
  use for output 3
```

Four outputs stay put while the inputs and weights stream past them.

#### 6b. Which dataflow is this?

In the lecture you saw dataflows named after whichever value is held still while the others move. This loop is a **hybrid**, and it is worth being precise about it, because two different values are being held still on two different timescales.

| Value | How long it stays in a register | How many MACs it serves |
| --- | --- | --- |
| The four running totals | the **whole** inner loop, all `N` iterations | 784 each, in Layer 1 |
| `X[j]` | one iteration of the inner loop | 4 |
| `W[...]` | not at all — loaded, used, discarded | 1 |

The four totals are held for the entire pass over the input, and each is written to SRAM exactly once at the end. That is **output stationary**.

Within a single iteration, `X[j]` is loaded once and feeds four multiply-accumulates before being discarded. That is a short window of **input stationary** behaviour, nested inside the output-stationary loop.

The weights are the streaming operand. Every weight is loaded, used for exactly one multiply, and never seen again.

> **Question:** the third dataflow from the lecture is **weight stationary** — hold a weight still and stream many inputs past it. Why can that not help here? Look at the table above and ask how many times any single weight is used during one inference. What would have to be different for a weight to be worth keeping in a register?

The answer to that question is the same reason Optimization 1 could only make the weight reads *cheaper* and never *fewer*.

#### 6c. Write it

```c
static void dense(const float *W, const float *B, const float *X,
                  float *Y, int M, int N)
{
    int i = 0;

    /* Main loop: four output neurons per pass */
    for (; i + 4 <= M; i += 4)
    {
        /* 1. Four running totals, one per output neuron, each starting
         *    at its own bias:  B[i], B[i+1], B[i+2], B[i+3]
         *
         * 2. One inner loop over j. Inside it:
         *       - load X[j] into a local float, ONCE
         *       - add W[(i+0)*N + j] * that value into total 0
         *       - add W[(i+1)*N + j] * that value into total 1
         *       - add W[(i+2)*N + j] * that value into total 2
         *       - add W[(i+3)*N + j] * that value into total 3
         *
         * 3. After the inner loop, write all four totals into
         *    Y[i], Y[i+1], Y[i+2], Y[i+3]
         */
    }

    /* Tail loop: whatever is left when M is not a multiple of 4.
     * This is your dense() from Assignment 1, unchanged. */
    for (; i < M; i++)
    {
        /* one output neuron, exactly as before */
    }
}
```

> **The tail loop is not optional.** Layer 1 has `M = 128`, which divides by 4 exactly. Layer 2 has `M = 10`, which does not — the main loop handles neurons 0 to 7 and leaves 8 and 9 behind. Without the tail loop, the scores for two digits are never computed, and `argmax` chooses from whatever happened to be in memory.
>
> Note that `i` is declared **before** both loops and not reset between them. That is what lets the second loop pick up where the first stopped.

> **Load `X[j]` into a local variable inside the inner loop**, and use that variable four times. If you write `X[j]` out four times instead, you have written down the same optimization you were trying to remove.

Build, flash, and record the cycle count.

> **Check:** all ten predictions must be identical to before. If digits 8 or 9 in the output look wrong, or the accuracy drops to 8/10, your tail loop is missing or wrong.

#### 6d. What changed

> **Questions:**
>
> 1. Recount the reads of `X` per inference with the new version. Compare with your answer from Step 5 question 2. By what factor did it drop?
> 2. Did the number of **weight** reads change? Look at the inner loop and count.
> 3. There is a second reason this version is faster, and it has nothing to do with data movement. A floating-point multiply-add takes several cycles to produce its result, and the next addition to the *same* running total has to wait for it. With four separate running totals, does the processor have to wait? What can it do instead?

Question 3 is worth discussing with the course team. It is the difference between a processor that is busy and a processor that is merely occupied.

### Step 7: Put the results together

Fill in this table:

| Version | Cycles | Time (ms) | Speedup vs baseline |
| --- | --- | --- | --- |
| Assignment 2 baseline | | | 1.00x |
| + weights in SRAM | | | |
| + four outputs at once | | | |

And these numbers, from your counting:

| Quantity | Value |
| --- | --- |
| MACs per inference | |
| Weight reads per inference | |
| Weight bytes read per inference | |
| Reads of `X` — original `dense` | |
| Reads of `X` — four-at-once `dense` | |
| Theoretical minimum cycles (1 MAC per cycle) | |
| Your best result, as a multiple of the minimum | |

> **Questions to discuss with the course team:**
>
> 1. Which of the two optimizations gave the bigger improvement? Does that match what you would have predicted from the data movement counts alone?
> 2. Both optimizations left the **number of weight reads** unchanged at roughly 100 000 per inference. Explain why that number cannot be reduced by rearranging the loops. What would have to change about the problem, or about the network, to reduce it?
> 3. You are still some distance from the theoretical minimum. Name one thing you think is still costing time, and describe an experiment that would confirm or rule it out.
> 4. Suppose this board were classifying one image per second from a camera, running on a battery. Which of the two optimizations would you actually keep, and what does each one cost you besides code complexity?

> **(Check with the course team when you finish this assignment)**

---

## Reference: things you used in this lab

### Timing with TIM2

| Code | What it does |
| --- | --- |
| `HAL_TIM_Base_Start(&htim2);` | Starts the counter — already in the project, in `USER CODE 2` |
| `TIM2->CNT` | Reads the current count |
| `t1 - t0` | Elapsed CPU cycles |
| `cycles / 160.0f` | Microseconds, at 160 MHz |
| `cycles / 160000.0f` | Milliseconds, at 160 MHz |

### Printing

| Format | Use for |
| --- | --- |
| `%d` | `int` |
| `%f` | `float` — needs `-u _printf_float` in `CMakeLists.txt` |
| `%lu` | `uint32_t` |
| `%p` | a pointer, to see which memory an array lives in |

`0x0800....` is flash. `0x2000....` is SRAM.

### Memory on the STM32U585

| | Size | Speed | Contents |
| --- | --- | --- | --- |
| Flash | 2 MB | slow, has wait states | your program, and everything declared `const` |
| SRAM | 768 KB | fast, no wait states | variables, the stack, anything not `const` |
| Registers | a few dozen values | free | whatever the compiler decides to keep there |

---

> After finishing all the assignments and checked by the **course team**, you will need to submit all your projects in a single compressed file to the Canvas assignment.
