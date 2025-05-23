Data Preparation
================

Receptual's algorithms work with multi-dimensional time-series data. Due to the generality of linear methods, this data can come from a variety of sources, including:

**Activity**

- Calcium imaging traces
- Averaged firing rates
- Spike trains
- RNN activations

**Stimuli**

- Image/video data
- Kinematics
- Sound

Any data loaded from disk must be a multi-dimensional NumPy array corresponding to one of *activity*, *stimulus*, *receptive field*, or *decoder*.

Array shapes
------------

There are no requirements on scaling your data. A strict requirement is that data is sampled at a fixed sample rate, and that your activity and stimlus recordings share common clocks and are of the same length.

We also require that you shape your arrays into the following shapes:

- **Activity**: `(n_timepoints, n_neurons)`
- **Stimulus**: `(n_timepoints, *spatial_dims)`
- **Receptive field**: `(kernel_size, n_neurons, *spatial_dims)`
- **Decoder**: `(kernel_size, n_neurons, *spatial_dims)`

Where `kernel_size` is the number of timepoints in the receptive field or decoder kernel, and `*spatial_dims` are any spatial dimensions (e.g. height and width for images). Note that we strictly require at least one spatial dimension and one neuron - if your stimulus data is a scalar value, or if you have activity from only one neuron, you must add a singleton dimension in the correct place. This is to avoid any ambiguity as to what the different dimensions correspond to - without it, your computation could silently fail, returning a result that doesn't mean what you intended.

Technically, our receptive field here actually corresponds to multiple independent receptive fields stacked for each neuron for simplicity. No correlations between neuron activities are modelled in standard receptive field regression. Similarly, for a decoder, different stimulus components are typically also modelled independetly.

Data size
---------

In order for receptive field estimation to be well-defined, it is necessary that the total size of the receptive field, given by the product of the kernel size and the number of spatial dimensions, is less than the number of timepoints in your stimulus. For example, if you have a 128x128 image stimulus, and a kernel size of 10, then you need at least 128x128x10 timepoints in your stimulus. Typically you should aim for much more than this. Similarly for a linear decoder, the total decoder size, given by the product of the kernel size and the number of neurons, should be less that the number of timepoints in your sample.

By default, we use zero padding at the edges of the your data. This means that predictions within the kernel size near the start of the recording for encoding, or within half the kernel size of the start and end of the recording for decoding, will have edge effects. This makes sense as there simply isn't enough data there to make a good estimate.

Note on whitened stimuli
------------------------

Often, you will hear it said that stimuli should be whitened to compute optimal receptive fields - that is, they your stimuli should have no temporal or spatial correlations prior to presentation to the animal/model. This is not technically necessary, and in fact by default our algorithm implements a decorrelation step that accounts for non-whitened stimuli.

However, there are some caveats here. What is necessary for receptive field estimation to be well-defined is that all frequencies resolvable within the kernel are present in the power spectrum of your stimulus. In full, the frequencies resolvable by the kernel are given by:

.. math::

	f_{res} = \frac{n}{K\Delta T}: n=1:\frac{K}{2}

Where :math:`K` is the kernel size, :math:`\Delta T` is your stimulus sample rate. This goes from the kernel frequency up to the Nyquist frequency, :math:`\frac{1}{2\Delta T}`, in units of the kernel frequency, and there are two independent phase components for each frequency.

These components must be present in the power spectrum of your stimulus. It doesn't require these frequency components to have equal power as in the case of white noise, but if any are missing, your receptive field is ill-defined. This can be solved by adding a small ridge-regression term, resolving the ambiguity by forcing receptive field terms to be closer to zero. It can also be improved by decreasing the kernel width K, decreasing the bandwidth of frequencies that need to be represented.

There are complications given by how components higher than the Nyquist frequency alias into the lower frequencies, and how non-integer frequencies get aliased, but a good rule of thumb is to ensure you have good coverage up to the Nyquist frequency.

**TL;DR:** the closer your stimuli are to white noise, the better your receptive field estimates will be. You need to ensure your stimulus has good frequency coverage, meaning monotone signals will fail. If your stimuli are lacking crucial frequency components, the situation can be improved by decreasing the kernel width K.

Some tricks
-----------

There are a few useful NumPy tools that can help you get your data into the right shape.

First of all, your data might need transposing. Say, for example, your stimulus consists of some stacked images of size 64x64, across 1000 timepoints, and your NumPy array has shape `(64, 64, 1000)`. You can use the `np.transpose` function to get it into the right shape:

.. code-block:: pycon

	>>> data.shape
	(64, 64, 1000)

	>>> data = np.transpose(data, (2, 0, 1))
	>>> data.shape
	(1000, 64, 64)

You might also need to add a singleton dimension. Say you have activity from just a single neuron across 1000 timepoints, and your NumPy array has shape `(1000,)`. You can use `np.newaxis` to add a singleton dimension:

.. code-block:: pycon

	>>> data.shape
	(1000,)

	>>> data = data[:, np.newaxis]
	>>> data.shape
	(1000, 1)

If your activity and stimulus are recorded at different sample rates, then you will need to resample them to match. There are a number of useful tools for this in `scipy.signal` and `scipy.interpolate`. We will demonstrate a particularly useful one here, using `scipy.signal.make_interp_spline`.

Let's say you recorded calcium imaging activity at 10Hz for 60s, and recorded 20 neurons. Then we expect your data to look like:

.. code-block:: pycon

	>>> activity.shape
	(600, 20)

Let's also say your stimulus was a 128x128 wideo recorded at 30Hz for 60s. Then we expect your data to look like:

.. code-block:: pycon

	>>> stimulus.shape
	(1800, 128, 128)

To resample your data to match, you have a few options. Let's say you choose to upsample your activity recording to 30Hz. This could be done as follows:

.. code-block:: pycon

	>>> from scipy.signal import make_interp_spline
	>>> import numpy as np

	>>> # Create a time vector for your activity data
	>>> t_act = np.linspace(0, 60, 600)

	>>> # Create a time vector for your stimulus data
	>>> t_stim = np.linspace(0, 60, 1800)

	>>> # Create a spline interpolation of your activity data
	>>> spline = make_interp_spline(t_act, activity, k=3)

	>>> # Resample your activity data to match the stimulus data
	>>> resampled_activity = spline(t_stim)

	>>> resampled_activity.shape
	(1800, 20)

Where `k=3` means we are using a cubic spline, which is often a good choice as it guarantees smoothness of the interpolation, while avoiding unnecessary oscillations from higher order polynomials.

Alternatively, you could have chosen to downsample your stimulus data. Additionally, this method is powerful enough to resample data from non-constant sample rates, as long as you know the underlying timestamps each sample was taken at.