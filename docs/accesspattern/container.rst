.. _container:

Data Container
=================

We have hinted in previous sections that we differentiate between
two "kinds" of data sources, which we called *iteration-based*
and *index-based* respectively. Of main interest in this section
are index-based data sources, which we will henceforth refer to
as **Data Container**. For a data source to qualify as such, it
must at the very least be able to provide the following
information:

1. The total number of observations :math:`N`, that the data
   source contains.

2. A way to query a specific observation or set of observations.
   This must be done using indices, where every observation has a
   unique index :math:`i \in I` assigned to it from the set of
   indices :math:`I = \{1, 2, ..., N\}`.

If a data source implements the required interface to be
considered a data container, a lot of additional much more
complex functionality comes for free. Yet the required interface
is rather unobtrusive and simple to implement.

- What makes a Julia type a data container are the implemented
  functions. That means that any custom type can be marked as a
  data container by simply implementing the required interface.
  This methodology is often called "duck typing". In other words,
  there is no abstract type that needs to be sub-typed. This fact
  makes the interface much less intrusive and allows package
  developers to opt-in more easily, without forcing them to make
  any architectural compromises.

- There is no requirement that the actual observations of a data
  container are stored in the working memory. Instead the data
  container could very well just be an interface to a remote data
  storage that requests the data on demand when queried.

- A data container can - but need not - be the data itself. For
  example a Julia array is both data, as well as data container.
  That means that querying specific observations of that array
  will again return an array. On the other hand, if the data
  container is a custom type that simply serves as an interface
  to some remote data set, then the type of the data container is
  distinct from the type of the data (which is likely an array)
  it returns.

We will spend the rest of this document on discussing data
containers in all its details. First, we will provide a rough
overview of how the interface looks like. After that, we will
take a closer look at every single function individually, and
even see some code examples showing off their behaviour.

Interface Overview
-------------------------

For any Julia type to be considered a data container it must
implement a minimal set of functions. All of these functions are
defined in a small utility package called `LearnBase.jl
<https://github.com/JuliaML/LearnBase.jl>`_. This means that in
order to implement the interface for some custom type, one has to
import that package first. More importantly, it implies that one
does **not** need to depend on ``MLDataUtils.jl`` itself. This
allows package developers to keep dependencies at a minimum,
while still taking part in the JuliaML ecosystem.

There are only two methods that *must* be implemented for every
data container. In other words, implementing these two methods is
sufficient and necessary for a type to be considered a data
container.

=======================================  ===================================================================
Required methods                         Brief description
=======================================  ===================================================================
``nobs(data, [obsdim])``                 Returns the total number of observations in ``data``
``getobs(data, idx, [obsdim])``          Returns the observation(s) from ``data`` indexed by ``idx``
=======================================  ===================================================================

Aside from the required interface, there are a number of optional
methods that can be implemented. The main motivation to provide
these methods as well, is that they can allow for a significant
boost in performance.

=======================================  ===================================================================
Optional methods                         Brief description
=======================================  ===================================================================
``getobs(data)``                         Returns all observations contained ``data`` in native form
``getobs!(buf, data, [idx], [obsdim])``  Inplace version of ``getobs(data, idx, obsdim)`` using ``buf``
``gettargets(data, idx, [obsdim])``      Returns the target(s) for the observation(s) in ``data`` at ``idx``
``datasubset(data, idx, obsdim)``        Returns an object representing a lazy subset of ``data`` at ``idx``
=======================================  ===================================================================

Out of the box, this package implements the full data container
interface for all subtypes of ``AbstractArray``. Furthermore,
``Tuples`` can be used to link multiple data containers together,
and thus are considered quasi data container. They are accepted
everywhere data containers are expected, but they do have very
special semantics in the context of this package. For more
information about how ``Tuples`` are interpreted, take a look at
:ref:`tuples`.

Number of Observations
------------------------

Every data container must be able to report how many observations
it contains and can provide. To that end it must implement the
function :func:`nobs`. For some data containers the meaning of
"observations" can be ambiguous and depend on a user convention.
For such cases it is possible to specify and additional argument,
that denotes the observation dimension.

.. function:: nobs(data, [obsdim]) -> Int

   Return the total number of observations that the given `data`
   container can provide.

   The optional parameter `obsdim` can be used to specify which
   dimension denotes the observations, if that concept makes
   sense for the type of `data`.

   :param data: The object representing a data container.

   :param obsdim: \
        Optional. If it makes sense for the type of `data`, then
        `obsdim` can be used to specify which dimension of `data`
        denotes the observations. It can be specified in a
        typestable manner as a positional argument, or as a more
        convenient keyword parameter. See :ref:`obsdim` for more
        information.

   :return: The number of observations in `data` as an Integer

We mentioned before that :func:`nobs` is implemented for any
subtype of ``AbstractArray``. This is also true for arrays of any
order, even higher order arrays (e.g. images).

.. code-block:: jlcon

   julia> y = rand(5)
   5-element Array{Float64,1}:
    0.542858
    0.28541
    0.613669
    0.217321
    0.018931

   julia> nobs(X)
   5

If there is more than one array dimension, all but the
observation dimension are implicitly assumed to be features (i.e.
part of that observation). This implies that the observations
have to be explicitly laid out along some dimension.

.. code-block:: jlcon

   julia> X = rand(2,5)
   2×5 Array{Float64,2}:
    0.175347  0.61498   0.621127   0.0697848  0.454302
    0.196735  0.283014  0.0961759  0.94303    0.584028

   julia> nobs(X)
   5

As you can see, the default assumption is that the last array
dimension enumerates the observations. This can be overwritten by
explicitly specifying the ``obsdim``.

.. code-block:: jlcon

   julia> nobs(X, ObsDim.First())
   2

   julia> nobs(X, obsdim = :first)
   2

   julia> nobs(X, obsdim = 1)
   2

Note how ``obsdim`` can either be provided using type-stable
positional arguments from the namespace ``ObsDim``, or by using a
more flexible and convenient keyword argument. We will discuss
observation dimensions in more detail in a later section.

Request Observation(s)
------------------------------

.. function:: getobs(data, idx, [obsdim])

   :param data: The object representing a data container.

   :param idx: \
        The index or indices of the observation(s) in `data`
        that the subset should represent. Can be of type ``Int``
        or some subtype ``AbstractVector{Int}``.

   :param obsdim: \
        Optional. If it makes sense for the type of `data`, then
        `obsdim` can be used to specify which dimension of `data`
        denotes the observations. It can be specified in a
        typestable manner as a positional argument, or as a more
        convenient keyword parameter. See :ref:`obsdim` for more
        information.

   :return: Should return the observation(s) indexed by `idx`.
        In what form is completely up to the user and can be
        specific to whatever task you have in mind! In other
        words there is **no** contract that the type of the
        return value has to fullfill.

The following methods can also be provided and are optional:

.. function:: getobs(data)

   By default this function will be the identity function for any
   type of `data` that does not prove a custom method for it.
   If that is not the behaviour that you want for your type,
   you need to provide this method yourself.

   :param data:
        The data of your custom user type. It should represent
        your dataset of interest and somehow know how to return
        the full dataset.

   :return: Should return all observations in `data`.
        In what form is completely up to the user and can be
        specific to whatever task you have in mind! In other
        words there is **no** contract that the type of the
        return value has to fullfill.

.. function:: getobs!(buffer, data, [idx], [obsdim])

   Inplace version of :func:`getobs`. If this method is provided
   for the type of `data`, then :func:`eachobs` and
   :func:`eachbatch` (among others) can preallocate a buffer that
   is then reused every iteration.

   :param buffer: \
        The preallocated storage to copy the given observations
        of `data` into. *Note:* The type and structure should be
        equivalent to the return value of :func:`getobs`, since
        this is how `buffer` is preallocated by default.

   :param data: The object representing a data container.

   :param idx: \
        Optional. The index or indices of the observation(s) in
        `data` that the subset should represent. Can be of type
        ``Int`` or some subtype ``AbstractVector{Int}``.

   :param obsdim: \
        Optional. If it makes sense for the type of `data`, then
        `obsdim` can be used to specify which dimension of `data`
        denotes the observations. It can be specified in a
        typestable manner as a positional argument, or as a more
        convenient keyword parameter. See :ref:`obsdim` for more
        information.


Request Target(s)
------------------------------

.. _obsdim:

Observation Dimension
----------------------

Note that when implementing support for your custom type,
``obsdim`` must be dispatched on as a positional argument only.
In that case `obsdim` can take on any of the following values.
Their interpretation is completely up to the user.

+--------------------+-------------------+------------------------+
| ``ObsDim.First()`` | ``ObsDim.Last()`` | ``ObsDim.Constant(N)`` |
+--------------------+-------------------+------------------------+
