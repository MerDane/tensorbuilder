# Tensor Builder

TensorBuilder is light-weight extensible library that enables you to easily create complex deep neural networks through a functional [fluent](https://en.wikipedia.org/wiki/Fluent_interface) [immutable](https://en.wikipedia.org/wiki/Immutable_object) API based on the Builder Pattern. Tensor Builder also comes with a DSL based on [applicatives](http://learnyouahaskell.com/functors-applicative-functors-and-monoids) and function composition that enables you to express more clearly the structure of your network, make changes faster, and reuse code.

### Goals

* Be a light-wrapper around Tensor-based libraries
* Enable users to easily create complex branched topologies while maintaining a fluent API (see [Builder.branch](http://cgarciae.github.io/tensorbuilder/tensorbuilder.m.html#tensorbuilder.tensorbuilder.Builder.branch))
* Let users be expressive and productive through a DSL

## Features
* **Branching**: Enable to easily express complex complex topologies with a fluent API. See [Branching](https://cgarciae.gitbooks.io/tensorbuilder/content/branching/).
* **Scoping**: Enable you to express scopes for your tensor graph using methods such as `tf.device` and `tf.variable_scope` with the same fluent API. [Scoping](https://cgarciae.gitbooks.io/tensorbuilder/content/scoping/).
* **DSL**: Use an abbreviated notation with a functional style to make the creation of networks faster, structural changes easier, and reuse code. See [DSL](https://cgarciae.gitbooks.io/tensorbuilder/content/dsl/).
* **Patches**: Add functions from other Tensor-based libraries as methods of the Builder class. TensorBuilder gives you a curated patch plus some specific patches from `TensorFlow` and `TFLearn`, but you can build you own to make TensorBuilder what you want it to be. See [Patches](https://cgarciae.gitbooks.io/tensorbuilder/content/patches/).

## Installation
Tensor Builder assumes you have a working `tensorflow` installation. We don't include it in the `requirements.txt` since the installation of tensorflow varies depending on your setup.

#### From github
1. `pip install git+https://github.com/cgarciae/tensorbuilder.git@0.0.5`

#### From pip
Coming soon!

## Getting Started

Create neural network with a [5, 10, 3] architecture with a `softmax` output layer and a `tanh` hidden layer through a Builder and then get back its tensor:

    import tensorflow as tf
    import tensorbuilder as tb

    x = tf.placeholder(tf.float32, shape=[None, 5])
    keep_prob = tf.placeholder(tf.float32)

    h = (
      tb
      .build(x)
      .tanh_layer(10) # tanh(x * w + b)
      .dropout(keep_prob) # dropout(x, keep_prob)
      .softmax_layer(3) # softmax(x * w + b)
      .tensor()
    )

    print(h)

## The Guide
Check out the guide [here](https://cgarciae.gitbooks.io/tensorbuilder/content/).

## Documentation
* Complete documentation: [here](http://cgarciae.github.io/tensorbuilder/).
* Builder API: [here](http://cgarciae.github.io/tensorbuilder/tensorbuilder.m.html).


## Examples

Here are many examples to you give a taste of what it feels like to use TensorBuilder and teach you some basic patterns.

#### Showoff
Next is some more involved code so you see all the features in action. Its for learning the MNIST for images of 20x20 gray-scaled by using 3 relu-CNN branches with max pooling, then merging the branches through a fully connected relu layer with dropout, and finally connecting it to a softmax output layer.

    import tensorflow as tf
    import tensorbuilder as tb
    import tensorbuilder.patch

    # Define variables
    y = tf.placeholder(tf.float32, shape=[None, 400])
    x = tf.placeholder(tf.float32, shape=[None, 400])
    keep_prob = tf.placeholder(tf.float32)

    #Create the convolution function to be used by each brach
    conv_branch = (

        dl.convolution2d(32, [5, 5])
        .relu()
        .max_pool_2d(2) #This method is taken from `tflearn`

        .convolution2d(64, [5, 5])
        .relu()
        .max_pool_2d(2)

        .flatten()
    )

    [h, loss, trainer] = dl.pipe(
        x,
        dl.reshape([-1, 20, 20, 1]),
        [
            conv_branch #Reuse code
        ,
            conv_branch
        ,
            conv_branch
        ],

        dl
        .relu_layer(1024) # this fully connects all 3 branches into a single relu layer
        .dropout(keep_prob)

        .linear_layer(10), # create a linear connection
        [
            dl.softmax() # h
        ,
            (dl
            .softmax_cross_entropy_with_logits(y)
            .map(tf.reduce_mean), #calculte loss
            [
                dl # loss
            ,
                dl.map(tf.train.AdadeltaOptimizer(0.01).minimize) # trainer
            ])
        ],
        dl.tensors() #get the list of tensors from the previous BuilderTree
    )

    print h, loss, trainer

Notice that:

1. We where able to reuse code easily by specifying the logic for the branches separately using the same syntax
2. Branches are expressed naturally as a list thanks to the DSL, the indentation levels match the depth of the tree. Nested branches are just as easy.
3. Most methods presented are functions from `tensorflow` that your are probably used to.


**WARNING: Examples next probably work but might be removed**


##############################
##### FUNCTIONS
##############################


##############################
##### builder
##############################

The following example shows you how to construct a `tensorbuilder.tensorbuilder.Builder` from a tensorflow Tensor.

    import tensorflow as tf
    import tensorbuilder as tb

    a = tf.placeholder(tf.float32, shape=[None, 8])
    a_builder = tb.build(a)

    print(a_builder)

The previous is the same as

    a = tf.placeholder(tf.float32, shape=[None, 8])
    a_builder = a.builder()

    print(a_builder)

##############################
##### branches
##############################

Given a list of Builders and/or BuilderTrees you construct a `tensorbuilder.tensorbuilder.BuilderTree`.

    import tensorflow as tf
    import tensorbuilder as tb

    a = tf.placeholder(tf.float32, shape=[None, 8]).builder()
    b = tf.placeholder(tf.float32, shape=[None, 8]).builder()

    tree = tb.branches([a, b])

    print(tree)

`tensorbuilder.tensorbuilder.BuilderTree`s are usually constructed using `tensorbuilder.tensorbuilder.Builder.branch` of the `tensorbuilder.tensorbuilder.Builder` class, but you can use this for special cases



##############################
##### BUILDER
##############################


##############################
##### fully_connected
##############################

This method is included by many libraries so its "sort of" part of TensorBuilder. The following builds the computation `tf.nn.sigmoid(tf.matmul(x, w) + b)`

    import tensorflow as tf
    import tensorbuilder as tb

    x = tf.placeholder(tf.float32, shape=[None, 5])

    h = (
    	tb
      .build(x)
    	.fully_connected(3, activation_fn=tf.nn.sigmoid)
    	.tensor()
    )

    print(h)

Using `tensorbuilder.patch` the previous is equivalent to

    import tensorflow as tf
    import tensorbuilder as tb

    x = tf.placeholder(tf.float32, shape=[None, 5])

    h = (
    	tb
      .build(x)
    	.sigmoid_layer(3)
    	.tensor()
    )

    print(h)



##############################
##### map
##############################

If you have a function f that takes a tensor as first arguments and then some \*args and \*\*kwargs, you can use the `map` method to use that function naturally. Take this TFLearn code for example

    import tflearn as tl

    net = tl.input_data(shape=[None, 784])
    net = tl.fully_connected(net, 64)
    net = tl.dropout(net, 0.5)
    net = tl.fully_connected(net, 10, activation='softmax')
    net = tl.regression(net, optimizer='adam', loss='categorical_crossentropy')

    model = tl.DNN(net)
    model.fit(X, Y)

We can rewrite it with TensorBuilder simply by using `map`.

    import tflearn as tl
    import tensorbuilder as tb

    model = (
      tb.build(tl.input_data(shape=[None, 784]))
      .map(tl.fully_connected, 64)
      .map(tl.dropout, 0.5)
      .map(tl.fully_connected, 10, activation='softmax')
      .map(tl.regression, optimizer='adam', loss='categorical_crossentropy')
      .map(tl.DNN)
      .tensor()
    )

    model.fit(X, Y)



##############################
##### then
##############################

The following *manually* constructs the computation `tf.nn.sigmoid(tf.matmul(x, w) + b)` while updating the `tensorbuilder.tensorbuiler.Builder.variables` dictionary.

    import tensorflow as tf
    import tensorbuilder as tb
    import tensorbuilder.slim_patch

    x = tf.placeholder(tf.float32, shape=[None, 40])
    keep_prob = tf.placeholder(tf.float32)

    def sigmoid_layer(builder, size):
    	x = builder.tensor()
    	m = int(x.get_shape()[1])
    	n = size

    	w = tf.Variable(tf.random_uniform([m, n], -1.0, 1.0))
    	b = tf.Variable(tf.random_uniform([n], -1.0, 1.0))

    	y = tf.nn.sigmoid(tf.matmul(x, w) + b)

    	return tb.build(y)

    h = (
    	tb.build(x)
    	.then(sigmoid_layer, 3)
    	.tensor()
    )

Note that the previous if equivalent to

    import tensorflow as tf
    import tensorbuilder as tb
    h = (
    	tb.build(x)
    	.sigmoid_layer(3)
    	.tensor()
    )

    print(h)

##############################
##### branch
##############################

The following will create a sigmoid layer but will branch the computation at the logit (z) so you get both the output tensor `h` and `trainer` tensor. Observe that first the logit `z` is calculated by creating a linear layer with `fully_connected(1)` and then its branched out

    import tensorflow as tf
    import tensorbuilder as tb
    import tensorbuilder.slim_patch

    x = tf.placeholder(tf.float32, shape=[None, 5])
    y = tf.placeholder(tf.float32, shape=[None, 1])

    [h, trainer] = (
        tb.build(x)
        .linear_layer(1)
        .branch(lambda z:
        [
            z.sigmoid()
        ,
            z.sigmoid_cross_entropy_with_logits(y)
            .map(tf.train.AdamOptimizer(0.01).minimize)
        ])
        .tensors()
    )

    print(h)
    print(trainer)

Note that you have to use the `tensorbuilder.tensorbuilder.BuilderTree.tensors` method from the `tensorbuilder.tensorbuilder.BuilderTree` class to get the tensors back.

Remember that you can also contain `tensorbuilder.tensorbuilder.BuilderTree` elements when you branch out, this means that you can keep branching inside branch. Don't worry that the tree keep getting deeper, `tensorbuilder.tensorbuilder.BuilderTree` has methods that help you flatten or reduce the tree.
The following example will show you how create a (overly) complex tree and then connect all the leaf nodes to a single `sigmoid` layer

    import tensorflow as tf
    import tensorbuilder as tb
    import tensorbuilder.slim_patch

    x = tf.placeholder(tf.float32, shape=[None, 5])
    keep_prob = tf.placeholder(tf.float32)

    h = (
        tb.build(x)
        .linear_layer(10)
        .branch(lambda base:
        [
            base
            .relu_layer(3)
        ,
            base
            .tanh_layer(9)
            .branch(lambda base2:
            [
            	base2
            	.sigmoid_layer(6)
            ,
            	base2
            	.dropout(keep_prob)
            	.softmax_layer(8)
            ])
        ])
        .sigmoid_layer(6)
    )

    print(h)

##############################
##### BUILDER TREE
##############################

##############################
##### builders
##############################

    import tensorflow as tf
    import tensorbuilder as tb
    import tensorbuilder.slim_patch

    x = tf.placeholder(tf.float32, shape=[None, 5])
    y = tf.placeholder(tf.float32, shape=[None, 1])

    [h_builder, trainer_builder] = (
        tb.build(x)
        .linear_layer(1)
        .branch(lambda z:
        [
            z.sigmoid()
        ,
            z.sigmoid_cross_entropy_with_logits(y)
            .map(tf.train.AdamOptimizer(0.01).minimize)
        ])
        .builders()
    )

    print(h_builder)
    print(trainer_builder)

##############################
##### tensors
##############################

    import tensorflow as tf
    import tensorbuilder as tb
    import tensorbuilder.slim_patch

    x = tf.placeholder(tf.float32, shape=[None, 5])
    y = tf.placeholder(tf.float32, shape=[None, 1])

    [h_tensor, trainer_tensor] = (
        tb.build(x)
        .linear_layer(1)
        .branch(lambda z:
        [
            z.sigmoid())
        ,
            z.sigmoid_cross_entropy_with_logits(y)
            .map(tf.train.AdamOptimizer(0.01).minimize)
        ])
        .tensors()
    )

    print(h_tensor)
    print(trainer_tensor)

##############################
##### fully_connected
##############################

The following example shows you how to connect two tensors (rather builders) of different shapes to a single `softmax` layer of shape [None, 3]

    import tensorflow as tf
    import tensorbuilder as tb
    import tensorbuilder.slim_patch

    a = tf.placeholder(tf.float32, shape=[None, 8]).builder()
    b = tf.placeholder(tf.float32, shape=[None, 5]).builder()

    h = (
    	tb.branches([a, b])
    	.fully_connected(3, activation_fn=tf.nn.softmax)
    )

    print(h)

The next example show you how you can use this to pass the input layer directly through one branch, and "analyze" it with a `tanh_layer` filter through the other, both of these are connect to a single `softmax` output layer

    import tensorflow as tf
    import tensorbuilder as tb
    import tensorbuilder.slim_patch

    x = tf.placeholder(tf.float32, shape=[None, 5])

    h = (
    	tb.build(x)
    	.branch(lambda x:
    	[
    		x
    	,
    		x.tanh_layer(10)
    	])
    	.softmax_layer(3)
    )

    print(h)