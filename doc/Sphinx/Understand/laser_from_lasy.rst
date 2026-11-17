Lasers imported from LASY
-------------------------

`LASY <https://lasydoc.readthedocs.io>`_ is an open-source Python library for modeling and
manipulating laser pulses. A pulse can be defined as a
Gaussian or speckled profile, as a combination of a longitudinal profile
(Gaussian, super-Gaussian, cosine or continuous wave) and a transverse profile
(Gaussian, super-Gaussian, flattened Gaussian, Laguerre-Gaussian,
Hermite-Gaussian, jinc or plane wave), or from measured data (a NumPy array,
an openPMD file or an INSIGHT measurement).

Currently, :program:`Smilei` can import a LASY laser in ``"AMcylindrical"`` geometry
(see :doc:`azimuthal_modes_decomposition`), in two ways:

* ``LaserFromLasyAM`` injects the full laser field from the ``xmin`` boundary;
* ``LaserEnvelopeFromLasyAM`` injects a laser envelope, to be used with the :doc:`laser_envelope`.

These are not two options of the same model: the first one injects the oscillating field
of a standard simulation, while the second one provides the envelope of the vector potential
to the envelope model, which does not resolve the laser oscillation.
How to use these creators is described in :doc:`/Use/namelist`, and test cases are given
in the benchmarks ``tstAM_20_laser_from_Lasy.py`` and ``tstAM_21_laser_envelope_from_Lasy.py``.

This page presents the physical model and the equations used to convert
the LASY laser into the fields injected by :program:`Smilei`.

----

The laser in LASY
^^^^^^^^^^^^^^^^^

LASY describes the laser by a complex envelope, which is either the envelope :math:`\tilde{E}`
of its transverse electric field, or the envelope :math:`\tilde{A}` of its transverse vector potential, normalized as in
:program:`Smilei` (in units of :math:`m_e c/e`). The attribute ``envelopeField`` of the LASY file indicates which one is stored
(``"electric_field"`` or ``"normalized_vector_potential"``).
For cylindrical geometry, the envelope is expanded in azimuthal modes
on a grid of time :math:`t` and radius :math:`r`:

.. math::
  :label: LasyEnvelope

  \tilde{E}(\theta, t, r) = \tilde{E}_0(t, r) + \sum_{m=1}^{N_L-1}
  \left[ \tilde{C}_m(t, r)\cos(m\theta) + \tilde{S}_m(t, r)\sin(m\theta) \right]

where :math:`N_L` is the number of azimuthal modes of the LASY laser
(the same expansion applies to :math:`\tilde{A}`).
Here, :math:`\tilde{C}_m` and :math:`\tilde{S}_m` are complex. Internally, LASY expands the
envelope as :math:`\tilde{E} = \sum_{m=-(N_L-1)}^{N_L-1} \mathcal{E}_m e^{-im\theta}`, where the
negative modes are independent of the positive ones since the envelope is complex.
Grouping the modes :math:`m` and :math:`-m` gives
:math:`\tilde{C}_m = \mathcal{E}_m + \mathcal{E}_{-m}` and
:math:`\tilde{S}_m = -i\mathcal{E}_m + i\mathcal{E}_{-m}`, which are the components stored in the
LASY file (openPMD ``thetaMode`` convention). Both representations are therefore equivalent,
and the negative modes are fully taken into account.

The physical electric field is obtained by adding the carrier oscillation
at the laser angular frequency :math:`\omega_0` and the polarization vector :math:`(p_x, p_y)`,
which may be complex (generalized elliptical polarization) and is normalized to
:math:`|p_x|^2 + |p_y|^2 = 1`:

.. math::
  :label: LasyField

  \mathbf{E} = \textrm{Re}\left[ \tilde{E}(\theta, t, r)\, e^{-i\omega_0 t}
  \left( p_x \mathbf{e}_y + p_y \mathbf{e}_z \right) \right]

**Conversion of the axes.**
LASY propagates the laser along its ``z`` axis, while :program:`Smilei` propagates it along ``x``.
The axes are converted by the cyclic permutation

.. math::

  (x, y, z)_\textrm{LASY} \;\rightarrow\; (y, z, x)_\textrm{Smilei}

which keeps the coordinate system right-handed, so the propagation direction
:math:`+z_\textrm{LASY}` becomes :math:`+x` and the rotation sense of the polarization is kept.
The polarization vector :math:`(p_x, p_y)` of LASY therefore gives the components of the
electric field along ``y`` and ``z`` in :program:`Smilei`:

.. math::

  E_y = \textrm{Re}\left[ \tilde{E}\, e^{-i\omega_0 t}\, p_x \right]
  \quad,\quad
  E_z = \textrm{Re}\left[ \tilde{E}\, e^{-i\omega_0 t}\, p_y \right]

For instance, a linear polarization along ``y`` corresponds to :math:`(p_x, p_y) = (1, 0)`,
and a circular polarization to :math:`(p_x, p_y) = (1, \pm i)/\sqrt{2}`.

----

Normalization
^^^^^^^^^^^^^

LASY uses SI units, whereas :program:`Smilei` uses normalized :doc:`units`,
defined by the reference angular frequency :math:`\omega_r`
(``reference_angular_frequency_SI`` in the ``Main`` block):
times are normalized by :math:`1/\omega_r`, lengths by :math:`c/\omega_r`,
electric fields by :math:`m_e c\,\omega_r/e` and magnetic fields by :math:`m_e\omega_r/e`.
The vector potential :math:`\tilde{A}` stored by LASY is already normalized as in :program:`Smilei`
and needs no conversion.
The laser frequency :math:`\omega_0` may differ from :math:`\omega_r`:
in normalized units, the laser oscillates at the frequency

.. math::

  \omega = \frac{\omega_0}{\omega_r}

which reduces to :math:`\omega = 1` when the two frequencies are equal.

----

Relation between the two envelopes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

At a fixed position, :math:`\mathbf{E} = -\partial_t\mathbf{A}`. Writing both fields as an envelope
multiplied by :math:`e^{-i\omega t}`, the two envelopes are related, in normalized units, by

.. math::
  :label: LasyEnvelopeEA

  \tilde{E} = i\omega\tilde{A} - \frac{\partial\tilde{A}}{\partial t}

For an envelope varying slowly compared with the laser period, the time derivative is negligible
and :math:`\tilde{E} \simeq i\omega\tilde{A}`, i.e. :math:`|\tilde{A}| \simeq |\tilde{E}|/\omega`.
The two creators use the full relation :eq:`LasyEnvelopeEA` whenever the file does not contain
the quantity they need: ``LaserFromLasyAM`` computes :math:`\tilde{E}` from a file containing
:math:`\tilde{A}`, and ``LaserEnvelopeFromLasyAM`` computes :math:`\tilde{A}` from a file containing :math:`\tilde{E}`.

.. note::

  When LASY saves the vector potential, it computes it with the slowly varying approximation
  :math:`\tilde{A} = \tilde{E}/(i\omega)`. Converting it back with the full relation :eq:`LasyEnvelopeEA`
  therefore gives a field that differs slightly, by a small phase shift of relative order
  :math:`1/(\omega\tau)` for a pulse of duration :math:`\tau`, from the one of a file
  containing :math:`\tilde{E}` for the same laser.

----

Full laser field: ``LaserFromLasyAM``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

At the ``xmin`` boundary, :program:`Smilei` injects a laser by imposing its transverse
magnetic field, in cylindrical components :math:`B_r` and :math:`B_\theta`,
decomposed in the :math:`N_m` azimuthal modes of the simulation (``number_of_AM`` in the ``Main`` block).
These components are obtained from the LASY envelope as follows.

**Electric field from the vector potential.**
If the file contains :math:`\tilde{A}`, the envelope :math:`\tilde{E}` is first computed
with Eq. :eq:`LasyEnvelopeEA`, the time derivative being evaluated with centered,
second-order finite differences on the time grid of the file.

**Magnetic field.**
For a laser propagating along ``x`` in vacuum, :math:`\mathbf{B} = \mathbf{e}_x \times \mathbf{E}/c`.
From Eq. :eq:`LasyField`, with :math:`\tilde{B} = \tilde{E}/c`, the envelopes of its transverse
components are :math:`\tilde{B}_y = -p_y\,\tilde{B}` and :math:`\tilde{B}_z = p_x\,\tilde{B}`,
and, with :math:`y = r\cos\theta` and :math:`z = r\sin\theta`, its cylindrical components are

.. math::
  :label: LasyBrBt

  \tilde{B}_r = \tilde{B}_y\cos\theta + \tilde{B}_z\sin\theta \quad,\quad
  \tilde{B}_\theta = -\tilde{B}_y\sin\theta + \tilde{B}_z\cos\theta

For a linear polarization along ``y``, this reduces to
:math:`\tilde{B}_r = \tilde{B}\sin\theta` and :math:`\tilde{B}_\theta = \tilde{B}\cos\theta`.

**Azimuthal decomposition.**
As in :doc:`azimuthal_modes_decomposition`, a real field is written

.. math::

  F(\theta) = \textrm{Re}\left[\sum_{m\geq 0} F^m e^{-im\theta}\right]
  \quad,\quad
  F^0 = \frac{1}{2\pi}\int_0^{2\pi} F\, d\theta
  \quad,\quad
  F^m = \frac{1}{\pi}\int_0^{2\pi} F\, e^{im\theta}\, d\theta \quad (m \geq 1)

This decomposition is applied separately to the real and imaginary parts of
:math:`\tilde{B}_r` and :math:`\tilde{B}_\theta`, which are real fields.
The multiplication by :math:`\cos\theta` or :math:`\sin\theta` in Eq. :eq:`LasyBrBt`
shifts each mode by :math:`\pm 1`: the mode :math:`m` of the LASY envelope contributes
to the modes :math:`|m \pm 1|` of the simulation. For instance, a cylindrically symmetric
envelope with linear polarization gives only the mode 1, and a LASY file with modes
:math:`0` to :math:`N_L-1` gives modes :math:`0` to :math:`N_L`.
Only the modes :math:`m < N_m` are injected.

**Carrier oscillation.**
Since :math:`\textrm{Re}\left[\tilde{B}\,e^{-i\omega t}\right] =
\textrm{Re}(\tilde{B})\cos(\omega t) + \textrm{Im}(\tilde{B})\sin(\omega t)`,
the injected field of each mode is

.. math::

  B^m_{r}(t, r) = B^m_{r,\textrm{Re}}(t, r)\cos(\omega t) + B^m_{r,\textrm{Im}}(t, r)\sin(\omega t)

  B^m_{\theta}(t, r) = B^m_{\theta,\textrm{Re}}(t, r)\cos(\omega t) + B^m_{\theta,\textrm{Im}}(t, r)\sin(\omega t)

where the subscripts Re and Im denote the modes of the real and imaginary parts of the envelope.
The envelope varies slowly and is linearly interpolated in :math:`t` and :math:`r`
from the LASY grid to the times and radii of the simulation,
while the fast carrier oscillation is computed analytically.

**Time origin and duration.**
The first time of the LASY grid corresponds to :math:`t=0` in the simulation
(a time offset stored in the file is not used).
By default, the laser is injected during the whole time axis of the file;
a shorter injection can be set with ``N_time``.
The field is zero outside the time and radial ranges of the LASY grid.

----

Laser envelope: ``LaserEnvelopeFromLasyAM``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :doc:`laser_envelope` describes the laser by the complex envelope :math:`\tilde{A}`
of its transverse vector potential, :math:`\mathbf{A} = \textrm{Re}\left[\tilde{A}\,e^{i\omega(x-t)}\right]`.

**Azimuthal modes.**
The envelope model assumes a cylindrically symmetric envelope.
Only the mode :math:`m=0` of the LASY laser is used.
This mode is the one of the scalar envelope, whereas the polarization is described separately
(see below): it is not the same as the mode :math:`0` of :math:`B_r` and :math:`B_\theta`
in ``LaserFromLasyAM``, in which a linearly polarized laser appears in the mode 1.
Non-axisymmetric features of the LASY laser, such as astigmatism or coma, are not represented.

**Polarization.**
The envelope model supports linear and circular polarizations only.
The global phase of the normalized polarization vector,
:math:`\alpha = \frac{1}{2}\arg(p_x^2 + p_y^2)`, is first extracted.

* If :math:`(p_x, p_y)\,e^{-i\alpha}` is real, the polarization is linear,
  along the direction of this real vector. The phase :math:`e^{i\alpha}` is included in the envelope.
* If :math:`|p_x| = |p_y|` and :math:`\textrm{Re}(p_x p_y^*) = 0`, the polarization is circular,
  and the envelope is the one of the ``y`` component.
* Other (elliptical) polarizations are not supported.

**Envelope of the vector potential.**
If the file contains :math:`\tilde{A}`, it is used directly.
If it contains :math:`\tilde{E}`, Eq. :eq:`LasyEnvelopeEA`, written as
:math:`\partial_t\tilde{A} = i\omega\tilde{A} - \tilde{E}`, is integrated along the LASY time grid,
of step :math:`\Delta t`, with a centered (trapezoidal) scheme:

.. math::

  \tilde{A}_{n+1} = \frac{\left(1 + i\omega\Delta t/2\right)\tilde{A}_n
  - \Delta t\left(\tilde{E}_n + \tilde{E}_{n+1}\right)/2}{1 - i\omega\Delta t/2}

This scheme is second-order accurate, and its amplification factor has a modulus equal to 1
for any :math:`\omega\Delta t`, so the integration is stable, even when the time step of the file
is longer than the laser period.
The initial value is the slowly varying solution of Eq. :eq:`LasyEnvelopeEA`,
to first order in the time derivative of the envelope:

.. math::

  \tilde{A}_0 = \frac{\tilde{E}_0}{i\omega} + \frac{\partial_t\tilde{E}_0}{(i\omega)^2}

The envelope :math:`\tilde{A}` is then linearly interpolated in :math:`t` and :math:`r`
on the simulation grid, with the same time origin and duration as for ``LaserFromLasyAM``.

----

Limitations
^^^^^^^^^^^

* Only the ``"AMcylindrical"`` geometry is supported.
* The LASY laser must be described in azimuthal modes (``thetaMode`` geometry in LASY).
* The laser can only be injected from ``box_side = "xmin"``.
* ``LaserEnvelopeFromLasyAM`` uses only the mode :math:`m=0`,
  and supports only linear or circular polarizations.
