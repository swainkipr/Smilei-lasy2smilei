Laser profiles from LASY
------------------------

The open-source library `LASY <https://github.com/LASY-org/lasy>`_ can model
realistic laser profiles, for instance profiles measured in an experiment,
in both the transverse and longitudinal directions.
The function ``LaserFromLasy`` reads the output file of :program:`LASY` and
injects the corresponding laser in a :program:`Smilei` simulation in
``"AMcylindrical"`` geometry (see :doc:`azimuthal_modes_decomposition`).

----

Structure of the LASY output
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:program:`LASY` writes its output in the openPMD standard, as an HDF5 file.
The laser envelope is stored in the dataset ``/data/0/meshes/laserEnvelope``.
In the ``thetaMode`` representation, this dataset has the shape ``(m, t, r)``, where:

* ``m`` is the index of the azimuthal mode component,
* ``t`` is the longitudinal (temporal) coordinate,
* ``r`` is the transverse (radial) coordinate.

For :math:`M` azimuthal modes, only :math:`2M-1` real components are stored:
the mode :math:`m=0`, then a :math:`\cos(m\theta)` and a :math:`\sin(m\theta)`
component for each mode :math:`m \geq 1`.
For example, with 3 modes, there are 5 components:
:math:`1`, :math:`\cos\theta`, :math:`\sin\theta`, :math:`\cos 2\theta` and :math:`\sin 2\theta`.
The full angular dependence of the field is therefore

.. math::
  :label: LasyFieldTheta

  E(\theta, t, r) = E_0(t, r) + \sum_{m=1}^{M}
  \left[ C_m(t, r)\cos(m\theta) + S_m(t, r)\sin(m\theta) \right]

Note that :program:`LASY` stores the envelope as the :math:`E_x` component,
whereas the corresponding component in :program:`Smilei` is :math:`E_y`.

----

Reconstruction of the magnetic field
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

At the boundaries, :program:`Smilei` imposes the magnetic field rather than the
electric field. The relevant component is :math:`B_z = E_y/c`.
The field :math:`B_z(\theta, t, r)` is first reconstructed from the stored
components, following Eq. :eq:`LasyFieldTheta`:

.. math::

  B_z(\theta, t, r) = B_0(t, r) + \sum_{m=1}^{M}
  \left[ C_m(t, r)\cos(m\theta) + S_m(t, r)\sin(m\theta) \right]

In ``"AMcylindrical"`` geometry, :program:`Smilei` does not use :math:`B_z`
directly, but its cylindrical components :math:`B_r` and :math:`B_\theta`.
These are decomposed into azimuthal modes, as in
:doc:`azimuthal_modes_decomposition`:

.. math::

  B_{r,\theta}(\theta, t, r) = \textrm{Re}\left[
  \sum_{m=0}^{+\infty} B^m_{r,\theta}(t, r)\, e^{-im\theta} \right]

where the coefficients of each mode are given by

.. math::

  B^m_r(t, r) = \frac{1}{2\pi}\int_0^{2\pi} B_r(\theta, t, r)\, e^{im\theta}\, d\theta
  \quad,\quad
  B^m_\theta(t, r) = \frac{1}{2\pi}\int_0^{2\pi} B_\theta(\theta, t, r)\, e^{im\theta}\, d\theta

The real and imaginary parts of the complex envelope are decomposed separately,
and are then combined with the laser carrier:

.. math::

  B^m_r(t, r) = B^m_{r,\textrm{real}}(t, r)\cos(\omega_r t) + B^m_{r,\textrm{imag}}(t, r)\sin(\omega_r t)

  B^m_\theta(t, r) = B^m_{\theta,\textrm{real}}(t, r)\cos(\omega_r t) + B^m_{\theta,\textrm{imag}}(t, r)\sin(\omega_r t)

where :math:`\omega_r` is the ratio between the laser carrier frequency and the
reference frequency of the simulation (see :doc:`units`).
When the two frequencies are equal, :math:`\omega_r = 1`.

The resulting laser boundary condition is a list of functions, two for each mode:

.. math::

  \left[ B^0_r,\, B^0_\theta,\, B^1_r,\, B^1_\theta,\, \dots \right]

----

Usage in the namelist
^^^^^^^^^^^^^^^^^^^^^

The function ``LaserFromLasy`` performs all the steps above and creates the laser::

  LaserFromLasy(
      filename            = "laser_lasy.h5",
      dt                  = Main.timestep,
      dtrans              = Main.cell_length[1],
      Ltrans              = Main.grid_length[1],
      ntrans              = Main.number_of_cells[1],
      N_time              = 4000,
      Nm                  = Main.number_of_AM,
      lambda_0            = 0.8e-6,
      omega_0             = 2.*math.pi*299792458./0.8e-6,
      reference_frequency = Main.reference_angular_frequency_SI,
      box_side            = "xmin",
  )

Internally, it passes the list of mode functions to a ``Laser`` block::

  Laser(
      box_side              = "xmin",
      space_time_profile_AM = mode_functions,
  )

The arguments are:

* ``filename`` (string): path to the output ``.h5`` file of :program:`LASY`.
* ``dt`` (float): simulation timestep.
* ``dtrans`` (float): cell size in the transverse direction.
* ``Ltrans`` (float): size of the simulation box in the transverse direction.
* ``ntrans`` (int): number of cells in the transverse direction.
* ``N_time`` (int): number of timesteps during which the laser is injected.
  The laser is zero for :math:`t \geq N_\textrm{time}\, dt`.
  If a moving window is used, choose ``N_time`` so that the window starts
  moving after the laser has been injected, i.e.
  ``time_start`` of the ``MovingWindow`` block is smaller than :math:`N_\textrm{time}\, dt`.
* ``Nm`` (int): number of azimuthal modes. It must be equal to ``number_of_AM``
  in the ``Main`` block.
* ``lambda_0`` (float): laser wavelength in SI units (m).
* ``omega_0`` (float): laser carrier angular frequency in SI units (rad/s).
* ``reference_frequency`` (float): reference angular frequency of the simulation
  in SI units (rad/s), i.e. ``reference_angular_frequency_SI`` in the ``Main`` block.
  It may differ from the laser frequency ``omega_0``.
* ``box_side`` (string): side of the box from which the laser is injected.
  Default: ``"xmin"``.

.. note::

  The current implementation has the following limitations:

  * only the ``"AMcylindrical"`` geometry is supported;
  * the :program:`LASY` file must use the ``thetaMode`` (azimuthal modes) representation;
  * only linearly polarized lasers are supported.
