# Hello World!
Let us get our first MPC program to run.
To make sure we can compile and the run the simplest program let us write a "Hello world!" program.
In the folder ```Programs/Source``` we will make a new file called ```hello_world.mpc```.
The file needs the ```.mpc``` extension, which tells the MP-SPDZ compiler that this is a program.
Open the file and add the following line of code
```
print_ln("Hello world!")
```
This line of code works as expected as it prints "Hello World!".

To run the program we need to first compile it using the MP-SPDZ compiler and then run it using a protocol.
To compile we run
```
./compile.py hello_world.mpc 
```
We do not need to provide a path to the program we have written we only provide the name of the file as long as it reside in the ```Programs/Source``` folder.
To run the program we execute the Script
```
Scripts/mascot.sh hello_world
```
The program we have written is now executed using the MASCOT protocol for multi-party computations.
Different protocol are available, and can be used to evaluate the program.
The list of available protocols and their security guarantees are available at [Protocols](https://mp-spdz.readthedocs.io/en/latest/readme.html#protocols).
The protocol decides which mathematical property used to do the multiparty computation.
In the case where we use MASCOT, we are are working in a finite field of size $2^n$.
If we for example were working on binary values, we could either use binary secret sharing in the Tiny protocol, or even use garbled circuits with the BMR protocol.
To keep this simple we will stick with the MASCOT.

Run the program and the output should look like this
```
Running MP-SPDZ/Scripts/../mascot-party.x 0 hello_world -pn 18477 -h localhost -N 2
Running MP-SPDZ/Scripts/../mascot-party.x 1 hello_world -pn 18477 -h localhost -N 2
Using statistical security parameter 40
Hello world!
The following benchmarks are including preprocessing (offline phase).
Time = 0.000687384 seconds 
Data sent = 0 MB in ~0 rounds (party 0 only; use '-v' for more details)
Global data sent = 0 MB (all parties)
```

As it get quite tedious to run both the compiler and the program each time a change is made, MP-SPDZ provide a script which can do this at once by using the following command
``` 
Scripts/compile-run.py mascot hello_world 
```

# First Multiplication
Now that we have written our own program and made it run, let us begin to do more interesting stuff with multi-party computation.
The first program we write that actually does stuff will be a simple multiplications program, that is we multiply two input values from two parties.

To get inputs from the parties we need to load them into the ```.mpc``` program. 
This is done using the function
```sint.get_input_from(i)```
This function will get an input from party ```i``` and return it as a ```sint``` which is the *secure integer* type used in MP-SPDZ.
Do not worry about this type as of now, as we will dive into it later.

Hence to start our multiplication program we need to get input from both parties.
So first create a new file called ```first_mul.mpc``` in ```Programs/Source``` and add the first two lines
```
a = sint.get_input_from(0)
b = sint.get_input_from(1)
```
We can then do simple arithmetic operations on these ```sint``` and the compiler of MP-SPDZ will perform the underlying steps needed to do the operations we want.
In our case we do multiplication, so we just use the ```*``` operations, hence add the line
```
mul = a * b
```
Now the product of the two input are stored in the ```mul``` variable.
However as this is secret integer type, we need to open/reveal the value for us to print it and see the results.
We can do this with the following lines of code
```
res = mul.reveal()
print_ln('Multiplication: %s', res)
```
Here ```res``` will be the plain-text values of the secret integer ```mul```, hence we can now print it using ```print_ln```
Note that the ```print_ln``` does not work in the same fashion as python ```print```, the MP-SPDZ print function works more like ```printf``` known form the programming language C. 
In our case we print the string "Multiplication: " followed by a string denoted as ```%s``` which will take the text value of ```res```

We are now ready to run our program
```
./compile.py first_mul.mpc 
Scripts/mascot.sh first_mul - I
```
In the script we add the argument ```-I``` to make the execution interactive. 
This will prompt us to input the values for the parties before executing the computations.
By inputting the values ```2 3``` we gain the following output
```
Running MP-SPDZ/Scripts/../mascot-party.x 0 first_mul -I -pn 17723 -h localhost -N 2
Running MP-SPDZ/Scripts/../mascot-party.x 1 first_mul -I -pn 17723 -h localhost -N 2
Using statistical security parameter 40
Please input integers:
2
3
Multiplication: 6
The following benchmarks are including preprocessing (offline phase).
Time = 3.43371 seconds 
Data sent = 20.5228 MB in ~55 rounds (party 0 only; use '-v' for more details)
Global data sent = 41.0456 MB (all parties)
```
As expected the output of the computation is ```6```.
You can run the program a few times to see that output is always equal to the product of the two input values.

If we do not want to input the input values each time we can use the input-data files from the ```Player-Data``` folder.
The input of each party is stored in a file with the name ```Input-Pi-0``` for Party ```i```. 
The values of the integers are stored in simple text format separated by spaces.
By adding the input vales of two and three for the both ```P0``` and ```P1```, we can run the protocol with interactive mode, by removing the ```-I``` from the execution script, or running
```
Scripts/compile-run.py mascot first_mul 
```
Giving the same output as before.

By writing this simple program we have now executed our first program using a multiplication, which is the simplest non-trivial thing to do within multi-party computation.

# Secure Integer and other MP-SPDZ types
In the```first_mul.mpc``` program we use the secret integer ```sint``` to denote the input of the parties. 
More precisely the ```sint``` is a type defined in the high-level interface of MP-SPDZ.
This specific type denotes a secret integer in the protocol-specific domain.
In other words the value of the integer is secret shared across the parties, and the way this secret sharing is realized depends on the protocol used.

When we use the ```sint.get_input_from(i)``` a few things happen.
First, party ```i``` reads the input from his file.
Secondly, he secret shares it with the other parties, this is the part that is protocol specific. However when reading formal protocol this will equate to the *Input* command in a MPC setting.
Now that we have a secret shared values, we can perform many different operations such as ```+, -, *, /```, and even comparisons ```==, != <, >``` etc.

A thing to note about the high-level interface is that a basic type like the ```sint``` is stored in a so-called register in the virtual machine of MP-SPDZ.
This means that this value is bound to a single thread and can only be accessed by that thread.
A container type such as ```Array``` is stored on so-called memory in the virtual machine.
Such types are not bound to a thread, hence can be access across multiple thread, both allowing for shared data or cross thread communication.

So when designing the MPC program these types needs to be considered.
However it is important to note that a list of basic types (e.g. a list of ```sint```) does not need to be a container type (e.g an ```Array```) the basic type can be used as vector, where each operation is element-wise.
We can read a vector as input by using
```sint.get_input_form(i, size=n)```
which reads ```n``` inputs from party ```i``` into a vector.

# MP-SPDZ Control Flow
Due to the compilation step of in MP-SPDZ we cannot use control flow as we would do in python.
The compiler read the python like code in the ```.mpc``` file and turn it into byte-code that the MP-SPDZ virtual machine can read.
Due to this compilation step we cannot write the MPC program like a python program, we need to tell te compiler what we are doing.
Python loops written in the ```.mpc``` program will be unrolled, hence can have impact on the performance, likewise can only depend on values known at compile time.

## Looping
To do a simple for-loop, we need to use the syntax
```
@for_range(n)
def _(i):
    ...
```
Here ```n``` is the range we loop over as in Python, however it needs to be a public integer, so we cannot for example loop over a ```sint``` type.
Another important aspect is that results from the loop iterations needs to be passed outside of the loop using an container type (e.g. ```Array```) or by using the ```update``` function on basic types.

A simple loop adding together numbers would look like this
```
a = sint.get_input_from(0, size=10)
n = 10
res = sint(0)
@for_range(n)
def _(i):
    res.update(res + a[i])
```
Here we see that the total sum is updated using the ```update``` function.

The MP-SPDZ compiler can also optimize the loop by running multiple loop-bodies in parallel.
This is done by using the ```@for_range_opt(n)``` notation instead.

## Functions
The same applies to functions, if we want to use a function definition during the MPC program execution we need to use the high-level interface.
The syntax is quite simple and works like you would expect in python.
```
@function
def mul_acc(a,b,c):
    return a * b + c
```
The arguments to the functions can be either container types or basic types, while the return value must be a basic type.
In our case we take 3 integers as input, multiply the first two and add the result to the third.

## If statements
Branching is non-trivial in MPC, hence we have two types of branching, one based on secret shared values (```sint```) and one based on public/clear values.
We first look at the public value case.

Again due to the compilation of MP-SPDZ to have a runtime if-statement, we need to use the correct annotation and syntax.
A simple if-statement on a clear/public values looks as follows
```
@if_(x>0)
def _():
    ...
```
Here the condition of the brach is written in the annotation. Hence this conditional piece of code is executed if ```x``` is greater than 0.
Likewise the if-else-statement looks as follows
```
@if_e(x > 0)
def _():
    ...
@else_
def _():
    ... 
```
This works as expected, we only need to add the ```e``` to the first annotation to use the else branch.
Note that to make values live beyond the scope of the if/else branch we need to make use of memory values such an ```Array``` or a ```MemValue``` (i.e. a single value stored in memory of the virtual machine).

Next the branching based on secret values (```sint```) works a bit different.
Here we cannot execute code based on the value of the secret.
First of all we might not know the exact values, and furthermore executing code based on the exact value will reveal something about the value, thus remove its secrecy.
What we can do however is use the value as a MUX i.e. as a bit selecting the output.
The syntax looks as follows
```
c.if_else(a, b)
```
The return value is ```a``` if ```c = 1``` and ```b``` if ```c = 0```. If ```c``` have other values the return is undefined.
At first this secret value based if seems strange, however since ```c``` is itself a secret values and defines the conditions we can easily make useful construction from this.
For example finding the max of two different secret values can be done as
```
(a < b).if_else(b, a)
```
Which in Python would be equivalent to
```
if a < b:
    return b
else:
    return a
```

To perform complex operations we need to take more case, since we only have a MUX on the secret value.
A simple case could be if ```x < 10``` we want to multiply with 2 and if ```x >= 10``` we want to multiply with 3.
To solve this issue we need to compute both branch of the if statement and use the MUX to select the correct output.
Because if both branches are computed the operations executed are independent of the value of the secret, hence the secrecy is not compromised.
```
x = sint(6)
t = x*2
f = x*3

out = (x < 10).if_else(t, f)
```
Running the program will give the output ```12``` as expected, and changing ```x=sint(12)``` to compute the other branch give the output ```36```.

# Number of points inside a circle
Let us use what we have learn to write a simple program, where party ```0``` provides a circle and party ```1``` provides a series of point and both parties learn the number of points within the circle.

Given the circle center is as $(x_0, y_0)$ with radius $r$ we can check the point $(x,y)$ using the formula 
```math
\sqrt{(x-x_0)^2 +(y-y_0)^2} < r
```
Computing square-roots does not result in integers so by rewriting we get
```math
(x-x_0)^2 + (y-y_0)^2 -r^2 < 0
```
which can be computed by integers only.

The code could looks as follows
```
circ_center = sint.get_input_from(0, size = 2)
radius = sint.get_input_from(0)

num_points = sint.get_input_from(1)
num_points = num_points.reveal()

count = sint(0)

@for_range(num_points)
def _(i):
    point = sint.get_input_from(1, size = 2)
    cal = sint(0)
    cal += (circ_center[0] - point[0]).square()
    cal += (circ_center[1] - point[1]).square()
    cal -= radius.square()
    inside = (cal < 0)
    count.update(count + inside)

num_inside = count.reveal()

print_ln('Total number of point in circle: %s', num_inside)
```

# Benchmarking using MP-SPDZ
The core of MP-SPDZ is to benchmark MPC protocols in different security settings.
Using our example of counting the number of points inside a circle we can benchmark our protocols.
As we have been running using MASCOT so far let us see how it performs.
On my machine the following are the results

|       | Time (Offline)  | Communication|
| - | - |- |
|Mascot | 1.05s (0.94) | 59.5MB|

We can then test the same program but using a different protocol.
MASCOT was based on OT to perform the computation. 
We now benchmark the same program using the HighGear protocol which is based on somewhat homomorphic encryption. 

|       | Time (Offline)  | Communication per party|
| - | - |- |
|Mascot | 1.05s (0.94s) | 59.5MB|
|HighGear| 27.9s (27.9s) | 414.3 MB|

The results are as expected since homomophic encryption is slower than the OT based approach.
The benchmarks can be better suited the specific program by changing the batch-size of the preprocessing by using ``` --batch-size ``` when running the program.
In our case it changes little regarding the performance of HighGear

Both of the protocols benchmarked have the security model of Malicious, dishonest majority.
A setting could be where we are ensure that the majority is honest, hence use a protocol in this setting.
We benchmark the malicious, honest majority version of Shamir
|       | Time (Offline)  | Communication per party|
| - | - |- |
|Mascot | 1.05s (0.94s) | 59.5MB|
|HighGear| 27.9s (27.9s) | 414.3 MB|
|Shamir(Malicious)| 0.091s (0.081s) | 2.44 MB

Furthermore we can relax the security even more by looking at the semi-honest version of Shamir.
In this setting every party must follow the protocol, and can only use the knowledge gained during the execution to try to extract information.
This could be in a case where every party is trusted, but someone might be spying on the communication.
The result become

|       | Time (Offline)  | Communication per party|
| - | - |- |
|Mascot | 1.05s (0.94s) | 59.5MB|
|HighGear| 27.9s (27.9s) | 414.3 MB|
|Shamir(Malicious)| 0.091s (0.081s) | 2.44 MB
|Shamir(Semi-Honest) | 0.015s (0.008s) | 0.33 MB

From the few benchmarks conducted the power of MP-SPDZ becomes clear.
The program can be benchmark using different protocol to see which performs the best.
Furthermore the program can be optimized to both decrease time consumption and communication very efficiently since MP-SPDZ tracks this information.
To gain a better overview of what the time and communication is spent on the execution of the protocol can be done in verbose mode ```-v``` which report time and communication of each part of the execution.