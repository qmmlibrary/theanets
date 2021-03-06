==================
Creating a Network
==================

The first step in using ``theanets`` is creating a model to train and use.
Often, this can be done fairly easily by combining:

- one of the three broad classes of network models,
- a series of layers that map inputs to outputs, and, optionally,
- regularizers to encourage the model to learn differently.

.. _creating-predefined-models:

Predefined Models
=================

There are three basic types of models in the neural networks literature; while
other types of models are certainly possible, ``theanets`` only tries to handle
the common cases with built-in model classes. To define a new type of model, see
:ref:`creating-customizing`.

In ``theanets``, a network model is a subclass of :class:`Network
<theanets.feedforward.Network>`. Its primary defining characteristics are the
``error`` property and the implementation of :func:`Network.setup_vars()
<theanets.feedforward.Network.setup_vars>`. The ``error`` property defines the
error function for the model, which is an important (and sometimes the only)
component of the loss that model trainers attempt to minimize during the
learning process. The ``setup_vars`` method defines the variables that the
network requires for computing an error value.

In the brief discussion below, we assume that the network has some set of
parameters :math:`\theta`. The network computes some function of its inputs
using these parameters, which we represent using the notation
:math:`F_\theta(x)`.

Autoencoder
-----------

An :class:`autoencoder <theanets.feedforward.Autoencoder>` takes an array of
arbitrary data :math:`x` as input. It attempts to recreate that same input at
its output layer.

To evaluate the loss for an autoencoder, only the input data is required. The
model computes the loss using the mean squared error between the network's
output and the input:

.. math::
   \mathcal{L}(X, \theta) = \frac{1}{m} \sum_{i=1}^m \left\| F_\theta(x_i) - x_i \right\|_2^2 + R(X, \theta)

To create an autoencoder in theanets, you can create a network class directly::

  net = theanets.Autoencoder()

or you can use an :class:`Experiment <theanets.main.Experiment>`::

  exp = theanets.Experiment(theanets.Autoencoder)
  net = exp.network

Regression
----------

A :class:`regression <theanets.feedforward.Regressor>` model is much like an
autoencoder, except that at training time, the expected output :math:`Y` must be
provided to the model. Like an autoencoder, a regression model takes as input an
array of arbitrary data :math:`X`, and the difference between the network's
output and the target is computed using the mean squared error:

.. math::
   \mathcal{L}(X, Y, \theta) = \frac{1}{m} \sum_{i=1}^m \left\| F_\theta(x_i) - y_i \right\|_2^2 + R(X, \theta)

To create a regression model in theanets, you can create a network class
directly::

  net = theanets.Regressor()

or you can use an :class:`Experiment <theanets.main.Experiment>`::

  exp = theanets.Experiment(theanets.Regressor)
  net = exp.network

Classification
--------------

A :class:`classification <theanets.feedforward.Classifier>` model takes as input
some piece of data that you want to classify (e.g., the pixels of an image, word
counts from a document, etc.) and outputs a probability distribution over
available labels. The error for this type of model takes an input dataset
:math:`X` and a corresponding set of integer labels :math:`Y`; the error is then
computed as the cross-entropy between the network output and the target labels:

.. math::
   \mathcal{L}(X, Y, \theta) = \frac{1}{m} \sum_{i=1}^m - \log F_\theta(x_i)_{y_i} + R(x, \theta)

To create a classifier model in ``theanets``, you can create a network class
directly::

  net = theanets.Classifier()

or you can use an :class:`Experiment <theanets.main.Experiment>`::

  exp = theanets.Experiment(theanets.Classifier)
  net = exp.network

.. _creating-recurrent-models:

Recurrent models
----------------

The three types of models described above also exist in recurrent formulations,
where time is an explicit part of the data being modeled. In ``theanets``, if
you wish to include recurrent layers in your model, you must use a model class
from the :mod:`theanets.recurrent` module; this is because recurrent models
require data matrices with an additional dimension to represent time.

.. _creating-specifying-layers:

Specifying Layers
=================

One of the most critical bits of creating a neural network model is specifying
how the layers of the network are configured. There are very few limits to the
complexity of possible neural network architectures, so it will never be
possible to specify all combinations using a single, easy-to-use markup.
However, ``theanets`` tries to make it easy to create networks with a single
"stack" of many common types of layers.

When you create a network model, the ``layers`` keyword argument is used to
specify the layers for your network. This keyword argument must be a sequence
specifying the layers; there are four options for the values in this sequence.

- If a value is an integer, it is interpreted as the size of a vanilla,
  fully-connected feedforward layer. All options for the layer are set to their
  defaults (e.g., the activation for a hidden layer will be given by the
  ``hidden_activation`` configuration parameter, which defaults to a logistic
  sigmoid).

  For example, to create a network with an input layer containing 4 units,
  hidden layers with 5 and 6 units, and an output layer with 2 units, you can
  just use integers to specify your layers::

    net = theanets.Experiment(theanets.Classifier, layers=(4, 5, 6, 2))

  The first element in the ``layers`` tuple should always be an integer; the
  :class:`Network.setup_layers() <theanets.feedforward.Network.setup_layers>`
  method creates an :class:`Input <theanets.layers.Input>` layer from the first
  element in the list.

- If a value is a tuple, it must contain an integer and may contain a string.
  The integer in the tuple specifies the size of the layer. If there is a
  string, and the string names a valid layer type (e.g., ``'tied'``, ``'rnn'``,
  etc.), then this type of layer will be created. Otherwise, the string is
  assumed to name an activation function (e.g., ``'logistic'``, ``'relu'``,
  etc.) and a standard feedforward layer will be created with that activation.
  (See below for a list of predefined activation functions.)

  For example, to create a model with a rectified linear activation in the
  middle layer::

    net = theanets.Classifier(layers=(4, (5, 'relu'), 6))

  Or to create a model with a recurrent middle layer::

    net = theanets.recurrent.Classifier(layers=(4, (5, 'rnn'), 6))

  Note that recurrent models (that is, models containing recurrent layers) are a
  bit different from feedforward ones; please see
  :ref:`creating-recurrent-models` for more details.

- If a value in this sequence is a dictionary, it must contain either a ``size``
  or an ``nout`` key, which specify the number of units in the layer. It can
  additionally contain an ``activation`` key to specify the activation function
  for the layer (see below), and a ``form`` key to specify the type of layer to
  be constructed (e.g., ``'tied'``, ``'rnn'``, etc.). Additional keys in this
  dictionary will be passed as keyword arguments to
  :func:`theanets.layers.build`.

  For example, you can create a standard feedforward network with dropouts in
  the hidden layer by defining your layers using a dictionary::

    net = theanets.Regressor(layers=(4, dict(size=5, dropout=0.3), 2))

  You can also use a dictionary to specify an non-default activation function
  for a layer in your model::

    net = theanets.Regressor(layers=(4, dict(size=5, activation='tanh'), 2))

- Finally, if a value is a :class:`Layer <theanets.layers.Layer>` instance, it
  is simply added to the network model as-is.

Activation functions
--------------------

An activation function (sometimes also called a transfer function) specifies how
the output of a layer is computed from the weighted sums of the inputs. By
default, hidden layers in ``theanets`` use a logistic sigmoid activation
function. Output layers in :class:`Regressor <theanets.feedforward.Regressor>`
and :class:`Autoencoder <theanets.feedforward.Autoencoder>` models use linear
activations (i.e., the output is just the weighted sum of the inputs from the
previous layer), and the output layer in :class:`Classifier
<theanets.feedforward.Classifier>` models uses a softmax activation.

To specify a different activation function for a layer, include an activation
key chosen from the table below. As described above, this can be included in
your model specification either using the ``activation`` keyword argument in a
layer dictionary, or by including the key in a tuple with the layer size.

=========  ============================  =============================================
Key        Description                   :math:`g(z) =`
=========  ============================  =============================================
linear     linear                        :math:`z`
sigmoid    logistic sigmoid              :math:`(1 + e^{-z})^{-1}`
logistic   logistic sigmoid              :math:`(1 + e^{-z})^{-1}`
tanh       hyperbolic tangent            :math:`\tanh(z)`
softplus   smooth relu approximation     :math:`\log(1 + \exp(z))`
softmax    categorical distribution      :math:`e^z / \sum e^z`
relu       rectified linear              :math:`\max(0, z)`
trel       truncated rectified linear    :math:`\max(0, \min(1, z))`
trec       thresholded rectified linear  :math:`z \mbox{ if } z > 1 \mbox{ else } 0`
tlin       thresholded linear            :math:`z \mbox{ if } |z| > 1 \mbox{ else } 0`
rect:max   truncation                    :math:`\min(1, z)`
rect:min   rectification                 :math:`\max(0, z)`
norm:mean  mean-normalization            :math:`z - \bar{z}`
norm:max   max-normalization             :math:`z / \max |z|`
norm:std   variance-normalization        :math:`z / \mathbb{E}[(z-\bar{z})^2]`
=========  ============================  =============================================

.. _creating-specifying-regularizers:

Specifying Regularizers
=======================

One heuristic that can prevent parameters from overtraining on small datasets is
based on the observation that "good" parameter values are typically small: large
parameter values often indicate overfitting. One way to encourage a model to use
small parameter values is to assume that the parameter values are sampled from a
posterior distribution over parameters, conditioned on observed data. In this
way of thinking about parameters, we can manipulate the prior distribution of
the parameter values to express our knowledge as modelers of the problem at
hand.

If you want to set up a more sophisticated model like a classifier with sparse
hidden representations, you can add regularization hyperparameters when you
create your experiment::

  exp = theanets.Experiment(
      theanets.Classifier,
      layers=(784, 1000, 784),
      hidden_l1=0.1)

Here we've specified that our model has a single, overcomplete hidden layer, and
the activity of the hidden units in the network will be penalized with a 0.1
coefficient.

Decay
-----

In "weight decay," we assume that parameters are drawn from a zero-mean Gaussian
distribution with an isotropic, modeler-specified standard deviation. In terms
of loss functions, this equates to adding a term to the loss function that
computes the :math:`L_2` norm of the parameter values in the model:

.. math::
   J(\cdot) = \dots + \frac{\lambda}{2} \| \theta \|_2^2

If the loss :math:`J(\cdot)` represents some approximation to the log-posterior
distribution of the model parameters given the data

.. math::
   J(\cdot) = \log p(\theta|x) \propto \dots + \frac{\lambda}{2} \| \theta \|_2^2

then the term with the :math:`L_2` norm on the parameters is like an unscaled
Gaussian distribution.

Sparsity
--------

Sparse models have been shown to capture regularities seen in the mammalian
visual cortex [Ols94]_. In addition, sparse models in machine learning are often
more performant than "dense" models (i.e., models without restriction on the
hidden representation) [Lee08]_. Furthermore, sparse models tend to yield latent
representations that are more interpretable to humans than dense models
[Tib96]_.

.. _creating-customizing:

Customizing
===========

The ``theanets`` package tries to strike a good balance between defining
everything known in the neural networks literature, and allowing you as a
programmer to create new stuff with the library. For many off-the-shelf use
cases, the hope is that something in ``theanets`` will work with just a few
lines of code. For more complex cases, you should be able to create an
appropriate subclass and integrate it into your workflow with a little more
effort.

.. _creating-custom-layers:

Defining Custom Layers
----------------------

Layers are the real workhorse in ``theanets``; custom layers can be created to
do all sorts of fun stuff. To create a custom layer, just subclass :class:`Layer
<theanets.layers.Layer>` and give it the functionality you want. As a very
simple example, let's suppose you wanted to create a normal feedforward layer
but did not want to include a bias term::

  import theanets
  import theano.tensor as TT

  class MyLayer(theanets.layers.Layer):
      def transform(self, inputs):
          return TT.dot(inputs, self.find('w'))

      def setup(self):
          self.log_setup(self.add_weights('w'))

Once you've set up your new layer class, it will automatically be registered and
available in :func:`theanets.layers.build` using the name of your class::

  layer = theanets.layers.build('mylayer', nin=3, nout=4)

or, while creating a model::

  net = theanets.Autoencoder(
      layers=(4, ('mylayer', 'linear', 3), 4),
      tied_weights=True,
  )

This example shows how fast it is to create a model that will learn the subspace
of your dataset that spans the most variance---the same subspace spanned by the
principal components.

.. _creating-custom-regularizers:

Defining Custom Regularizers
----------------------------

To create a custom regularizer in ``theanets``, you need to subclass the
appropriate model and provide an implementation of the
:func:`theanets.feedforward.Network.loss` method.

Let's keep going with the example above. Suppose you created a linear autoencoder
model that had a larger hidden layer than your dataset::

  net = theanets.Autoencoder(layers=(4, ('linear', 8), 4), tied_weights=True)

Then, at least in theory, you risk learning an uninteresting "identity" model
such that some hidden units are never used, and the ones that are have weights
equal to the identity matrix. To prevent this from happening, you can impose a
sparsity penalty::

  net = theanets.Autoencoder(
      layers=(4, ('linear', 8), 4),
      tied_weights=True,
      hidden_l1=0.1,
  )

But then you might run into a situation where the sparsity penalty drives some
of the hidden units in the model to zero, to "save" loss during training.
Zero-valued features are probably not so interesting, so we can introduce
another penalty to prevent feature weights from going to zero::


  class RICA(theanets.Autoencoder):
      def loss(self, **kwargs):
          loss = super(RICA, self).loss(**kwargs)
          w = kwargs.get('weight_inverse', 0)
          if w > 0:
              loss += w * sum((1 / (p * p).sum(axis=0)).sum()
                              for l in self.layers for p in l.params)
          return loss

This code adds a new regularizer that penalizes the inverse of the squared
length of each of the weights in the model's layers.

.. _creating-custom-errors:

Defining Custom Error Functions
-------------------------------

It's pretty straightforward to create models in ``theanets`` that use different
error functions from the predefined :class:`Classifier
<theanets.feedforward.Classifier>` (which uses categorical cross-entropy) and
:class:`Autoencoder <theanets.feedforward.Autoencoder>` and :class:`Regressor
<theanets.feedforward.Regressor>` (which both use mean squared error, MSE). To
define by a model with a new cost function, just create a new :class:`Network
<theanets.feedforward.Network>` subclass and override the ``error`` property.

For example, to create a regression model that uses mean absolute error (MAE)
instead of MSE::

  class MaeRegressor(theanets.Regressor):
      @property
      def error(self):
          return TT.mean(abs(self.outputs[-1] - self.targets))

Your cost function must return a theano expression that reflects the cost for
your model.


References
==========

.. [Glo11] X Glorot, A Bordes, Y Bengio. "Deep sparse rectifier neural
           networks." In *Proc AISTATS*, 2011.

.. [Hot33] H Hotelling. "Analysis of a Complex of Statistical Variables Into
           Principal Components." *Journal of Educational Psychology*
           **24**:417-441 & 498-520, 1933.

.. [Hyv97] A Hyvärinen, "Independent Component Analysis by Minimization of
           Mutual Information." University of Helsinki Tech Report, 1997.

.. [Jut91] C Jutten, J Herault. "Blind separation of sources, part I: An
           adaptive algorithm based on neuromimetic architecture." *Signal
           Processing* **24**:1-10, 1991.

.. [Le11] QV Le, A Karpenko, J Ngiam, AY Ng. "ICA with reconstruction cost for
          efficient overcomplete feature learning." In *Proc NIPS*, 2011.

.. [Lee08] H Lee, C Ekanadham, AY Ng. "Sparse deep belief net model for visual
           area V2." In *Proc. NIPS*, 2008.

.. [Ols94] B Olshausen, DJ Field. "Emergence of simple-cell receptive fields
           properties by learning a sparse code for natural images." *Nature*
           **381** 6583:607-609, 1994.

.. [Sut13] I Sutskever, J Martens, G Dahl, GE Hinton. "On the importance of
           initialization and momentum in deep learning." In *Proc ICML*, 2013.
           http://jmlr.csail.mit.edu/proceedings/papers/v28/sutskever13.pdf

.. [Tib96] R Tibshirani. "Regression shrinkage and selection via the lasso."
           *Journal of the Royal Statistical Society: Series B (Methodological)*
           267-288, 1996.
