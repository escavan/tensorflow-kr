# Basic Usage

TensorFlow 를 사용하기 위해서는 TensorFlow 가 어떻게 동작하는지 이해할 필요가 있다.

* `Graphs`는 컴퓨터 연산을 나타냅니다.
* `Sessions`이라는 컨텍스트 안에서 그래프는 실행 됩니다.
* `Tensors`는 데이터를 나타냅니다.
* `Variables`은 상태를 유지합니다.
* `Feeds & Fetches`은 임의의 작업의 데이터 입출력에 사용 됩니다.

## Overview

TensorFlow 의 프로그램 시스템은 `Graphs`를 통해서 컴퓨터 연산을 나타낸다.  `Graph` 안에서 노드는
*ops* (작은 단위의 동작) 불리우며, 한개의 OP는 제로 또는 하나 이상의 `Tensors` 가지게 된다.
OP는 몇 가지의 연산을 하기 되고, 제로 또는 하나 이상의 `Tensors` 만들어 냅니다.
`Tensor` 는 다차원 배열을 타입으로 가지게 된다.
예를 들어, 이미지를 다음과 같이 `[batch, height, width, channels]`4차원 배열로 나타낼 수 있다.

TensorFlow의 graph는 컴퓨터 연산의 *description* 설명이다. 어떤것을 연산하든, graph는
반드시 `Session` 안에서 실행되어야 한다. `Session`은 graph 작은 단위 연산들을 CPUs 또는 GPUs
불리우는 `Devices`에 배치시키고, 작은 단위 연산(ops)들이 작동할수 있는 기능을 제공한다.
이런 기능은 단위 연산(ops)에 의해서 tensors 들이 만들어진다. 파이션에서는 `ndarray` 오프젝트로
C 와 C++ 에서는 `TensorFlow::Tensor` 오프젝트를 만들어내게 된다.

## The computation graph

TensorFlow 프로그램들은 대개 graph를 조립하는 '구성 단계'와 session을 이용해 graph 안에
작은 단위의 연산(ops)을 실행시키는 '실행 단계'로 구성돼 있다.

예를 들어, 일반적으로 '구성 단계'에선 neural network를 대표하고 훈련시키기 위한 graph를 만들고,
'실행 단계'에선 트레이닝할 작은 단위의 연산(ops) 세트를 session을 이용해 반복 실행 시킨다.

TensorFlow는 C, C++, Python에서 사용할 수 있다. 현재, Python 라이브러리에서 C/C++에서 제공하지 않는
많은 유용한 함수들을 제공하고 있어 Python을 사용하는 것이 graph를 조립하는데 더 편할 것이다.

session 라이브러리는 세 언어에서 동등한 기능을 사용할 수 있다.

### Building the graph
graph를 만드는 것은 `Constant`와 같이 어떠한 input도 필요하지 않는 작은 단위의 동작(ops)으로 시작한다.
Python 라이브러리에서 단은 단위 연산(ops) 생성자는 구성된 작은 단위 연산(ops)의 결과(output)를 대기하는
객체를 반환한다. 그리고 이 객체들은 다른 작은 단위 연산(ops) 생성자의 input으로 전달할 수 있다.

Python 라이브러리로 사용하는 TensorFlow는 작은 단위 연산(ops) 생성자가 노드를 추가한 
*default graph*를 가지고 있다. default graph는 많은 어플리케이션용으로 충분하다.
[Graph class](../api_docs/python/framework.md#Graph) documentation에서 어떻게 많은 graph를
명시적으로 관리할 수 있는지 알 수 있다.

```python
import tensorflow as tf

# Create a Constant op that produces a 1x2 matrix.  The op is
# added as a node to the default graph.
#
# The value returned by the constructor represents the output
# of the Constant op.
matrix1 = tf.constant([[3., 3.]])

# Create another Constant that produces a 2x1 matrix.
matrix2 = tf.constant([[2.],[2.]])

# Create a Matmul op that takes 'matrix1' and 'matrix2' as inputs.
# The returned value, 'product', represents the result of the matrix
# multiplication.
product = tf.matmul(matrix1, matrix2)
```

default graph는 3개의 노드(`constant()` ops 2개와 `matmul()` op 한개)를 가지고 있다.
실제 매트릭스들을 곱하고 곱셈한 연산의 결과를 얻기 위해선, session에서 graph를 실행해야 한다.

### Launching the graph in a session

Launching follows construction.  To launch a graph, create a `Session` object.
Without arguments the session constructor launches the default graph.

See the [Session class](../api_docs/python/client.md#session-management) for
the complete session API.

```python
# Launch the default graph.
sess = tf.Session()

# To run the matmul op we call the session 'run()' method, passing 'product'
# which represents the output of the matmul op.  This indicates to the call
# that we want to get the output of the matmul op back.
#
# All inputs needed by the op are run automatically by the session.  They
# typically are run in parallel.
#
# The call 'run(product)' thus causes the execution of three ops in the
# graph: the two constants and matmul.
#
# The output of the op is returned in 'result' as a numpy `ndarray` object.
result = sess.run(product)
print(result)
# ==> [[ 12.]]

# Close the Session when we're done.
sess.close()
```

Sessions should be closed to release resources. You can also enter a `Session`
with a "with" block. The `Session` closes automatically at the end of the
`with` block.

```python
with tf.Session() as sess:
  result = sess.run([product])
  print(result)
```

The TensorFlow implementation translates the graph definition into executable
operations distributed across available compute resources, such as the CPU or
one of your computer's GPU cards. In general you do not have to specify CPUs
or GPUs explicitly. TensorFlow uses your first GPU, if you have one, for as
many operations as possible.

If you have more than one GPU available on your machine, to use a GPU beyond
the first you must assign ops to it explicitly. Use `with...Device` statements
to specify which CPU or GPU to use for operations:

```python
with tf.Session() as sess:
  with tf.device("/gpu:1"):
    matrix1 = tf.constant([[3., 3.]])
    matrix2 = tf.constant([[2.],[2.]])
    product = tf.matmul(matrix1, matrix2)
    ...
```

Devices are specified with strings.  The currently supported devices are:

*  `"/cpu:0"`: The CPU of your machine.
*  `"/gpu:0"`: The GPU of your machine, if you have one.
*  `"/gpu:1"`: The second GPU of your machine, etc.

See [Using GPUs](../how_tos/using_gpu/index.md) for more information about GPUs
and TensorFlow.

### Launching the graph in a distributed session

To create a TensorFlow cluster, launch a TensorFlow server on each of the
machines in the cluster. When you instantiate a Session in your client, you
pass it the network location of one of the machines in the cluster:

```python
with tf.Session("grpc://example.org:2222") as sess:
  # Calls to sess.run(...) will be executed on the cluster.
  ...
```

This machine becomes the master for the session. The master distributes the
graph across other machines in the cluster (workers), much as the local
implementation distributes the graph across available compute resources within
a machine.

You can use "with tf.device():" statements to directly specify workers for
particular parts of the graph:

```python
with tf.device("/job:ps/task:0"):
  weights = tf.Variable(...)
  biases = tf.Variable(...)
```

See the [Distributed TensorFlow How To](../how_tos/distributed/) for more
information about distributed sessions and clusters.

## Interactive Usage

The Python examples in the documentation launch the graph with a
[`Session`](../api_docs/python/client.md#Session) and use the
[`Session.run()`](../api_docs/python/client.md#Session.run) method to execute
operations.

For ease of use in interactive Python environments, such as
[IPython](http://ipython.org) you can instead use the
[`InteractiveSession`](../api_docs/python/client.md#InteractiveSession) class,
and the [`Tensor.eval()`](../api_docs/python/framework.md#Tensor.eval) and
[`Operation.run()`](../api_docs/python/framework.md#Operation.run) methods.  This
avoids having to keep a variable holding the session.

```python
# Enter an interactive TensorFlow Session.
import tensorflow as tf
sess = tf.InteractiveSession()

x = tf.Variable([1.0, 2.0])
a = tf.constant([3.0, 3.0])

# Initialize 'x' using the run() method of its initializer op.
x.initializer.run()

# Add an op to subtract 'a' from 'x'.  Run it and print the result
sub = tf.sub(x, a)
print(sub.eval())
# ==> [-2. -1.]

# Close the Session when we're done.
sess.close()
```

## Tensors

TensorFlow programs use a tensor data structure to represent all data -- only
tensors are passed between operations in the computation graph. You can think
of a TensorFlow tensor as an n-dimensional array or list. A tensor has a
static type, a rank, and a shape.  To learn more about how TensorFlow handles
these concepts, see the [Rank, Shape, and Type](../resources/dims_types.md)
reference.

## Variables

Variables maintain state across executions of the graph. The following example
shows a variable serving as a simple counter.  See
[Variables](../how_tos/variables/index.md) for more details.

```python
# Create a Variable, that will be initialized to the scalar value 0.
state = tf.Variable(0, name="counter")

# Create an Op to add one to `state`.

one = tf.constant(1)
new_value = tf.add(state, one)
update = tf.assign(state, new_value)

# Variables must be initialized by running an `init` Op after having
# launched the graph.  We first have to add the `init` Op to the graph.
init_op = tf.initialize_all_variables()

# Launch the graph and run the ops.
with tf.Session() as sess:
  # Run the 'init' op
  sess.run(init_op)
  # Print the initial value of 'state'
  print(sess.run(state))
  # Run the op that updates 'state' and print 'state'.
  for _ in range(3):
    sess.run(update)
    print(sess.run(state))

# output:

# 0
# 1
# 2
# 3
```

The `assign()` operation in this code is a part of the expression graph just
like the `add()` operation, so it does not actually perform the assignment
until `run()` executes the expression.

You typically represent the parameters of a statistical model as a set of
Variables. For example, you would store the weights for a neural network as a
tensor in a Variable. During training you update this tensor by running a
training graph repeatedly.

## Fetches

To fetch the outputs of operations, execute the graph with a `run()` call on
the `Session` object and pass in the tensors to retrieve. In the previous
example we fetched the single node `state`, but you can also fetch multiple
tensors:

```python
input1 = tf.constant([3.0])
input2 = tf.constant([2.0])
input3 = tf.constant([5.0])
intermed = tf.add(input2, input3)
mul = tf.mul(input1, intermed)

with tf.Session() as sess:
  result = sess.run([mul, intermed])
  print(result)

# output:
# [array([ 21.], dtype=float32), array([ 7.], dtype=float32)]
```

All the ops needed to produce the values of the requested tensors are run once
(not once per requested tensor).

## Feeds

The examples above introduce tensors into the computation graph by storing them
in `Constants` and `Variables`. TensorFlow also provides a feed mechanism for
patching a tensor directly into any operation in the graph.

A feed temporarily replaces the output of an operation with a tensor value.
You supply feed data as an argument to a `run()` call. The feed is only used for
the run call to which it is passed. The most common use case involves
designating specific operations to be "feed" operations by using
tf.placeholder() to create them:

```python

input1 = tf.placeholder(tf.float32)
input2 = tf.placeholder(tf.float32)
output = tf.mul(input1, input2)

with tf.Session() as sess:
  print(sess.run([output], feed_dict={input1:[7.], input2:[2.]}))

# output:
# [array([ 14.], dtype=float32)]
```

A `placeholder()` operation generates an error if you do not supply a feed for
it. See the
[MNIST fully-connected feed tutorial](../tutorials/mnist/tf/index.md)
([source code](https://www.tensorflow.org/code/tensorflow/g3doc/tutorials/mnist/fully_connected_feed.py))
for a larger-scale example of feeds.
