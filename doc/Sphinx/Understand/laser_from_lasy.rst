Lasers imported from LASY
-------------------------

`LASY <https://lasydoc.readthedocs.io>`_ is an open-source Python library for modeling and
manipulating laser pulses. A pulse can be defined as a
Gaussian or speckled profile, as a combination of a longitudinal profile
(Gaussian, super-Gaussian, cosine or continuous wave) and a transverse profile
(Gaussian, super-Gaussian, flattened Gaussian, Laguerre-Gaussian,
Hermite-Gaussian, jinc or plane wave), or from measured data (a NumPy array,
an openPMD file or an INSIGHT measurement). For now LASY supports in :program:`Smilei`  ``"AMcylindrical"`` geometry (:doc:`azimuthal_modes_decomposition`)  and :program:`Smilei` can import a pulse from a LASY file in two ways:

* ``LaserFromLasyAM`` injects the full laser field from the ``xmin`` boundary;
* ``LaserEnvelopeFromLasyAM`` injects a laser envelope, to be used with the
  :doc:`laser_envelope`.

How to use these creators is described in :doc:`/Use/namelist`.
This page presents the physical model and the equations used to convert
the LASY laser into the fields injected by :program:`Smilei`.

----

The laser in LASY
^^^^^^^^^^^^^^^^^

LASY describes the laser by the complex envelope :math:`\tilde{E}` of its transverse
electric field. For cylindrical geometry, the envelope is expanded in azimuthal modes
on a grid of time :math:`t` and radius :math:`r`:

.. math::
  :label: LasyEnvelope

  \tilde{E}(\theta, t, r) = \tilde{E}_0(t, r) + \sum_{m=1}^{N_L-1}
  \left[ \tilde{C}_m(t, r)\cos(m\theta) + \tilde{S}_m(t, r)\sin(m\theta) \right]

Here, :math:`\tilde{C}_m` and :math:`\tilde{S}_m` are complex. Internally, LASY expands the
envelope as :math:`\tilde{E} = \sum_{m=-(N_L-1)}^{N_L-1} \mathcal{E}_m e^{-im\theta}`, where the
negative modes are independent of the positive ones since the envelope is complex.
Grouping the modes :math:`m` and :math:`-m` gives
:math:`\tilde{C}_m = \mathcal{E}_m + \mathcal{E}_{-m}` and
:math:`\tilde{S}_m = -i\mathcal{E}_m + i\mathcal{E}_{-m}`, which are the components stored in the
LASY file (openPMD ``thetaMode`` convention). Both representations are therefore equivalent,
and the negative modes are fully taken into account.
:math:`N_L` is the number of azimuthal modes of the LASY laser. 
The physical electric field is obtained by adding the carrier oscillation
at the laser angular frequency :math:`\omega_0` and the polarization vector :math:`(p_x, p_y)`,
which may be complex:

.. math::
  :label: LasyField

  \mathbf{E} = \textrm{Re}\left[ \tilde{E}(\theta, t, r)\, e^{-i\omega_0 t}
  \left( p_x \mathbf{e}_y + p_y \mathbf{e}_z \right) \right]


LASY propagates the laser along its ``z`` axis, while :program:`Smilei` propagates it along ``x``.
The transverse axes ``(x, y)`` of LASY therefore correspond to the axes ``(y, z)`` of :program:`Smilei`.
It supports a generalized eliptical polarization.

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



----

Normalization
^^^^^^^^^^^^^

LASY uses SI units, whereas :program:`Smilei` uses normalized :doc:`units`,
defined by the reference angular frequency :math:`\omega_r`
(``reference_angular_frequency_SI`` in the ``Main`` block):
times are normalized by :math:`1/\omega_r`, lengths by :math:`c/\omega_r`,
electric fields by :math:`m_e c\,\omega_r/e` and magnetic fields by :math:`m_e\omega_r/e`.
The laser frequency :math:`\omega_0` may differ from :math:`\omega_r`:
in normalized units, the laser oscillates at the frequency

.. math::

  \omega = \frac{\omega_0}{\omega_r}

which reduces to :math:`\omega = 1` when the two frequencies are equal.

----

Full laser field: ``LaserFromLasyAM``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:program:`Smilei` injects a laser by imposing its transverse
magnetic field, in cylindrical components :math:`B_r` and :math:`B_\theta`,
decomposed in the :math:`N_m` azimuthal modes of the simulation.
These components are obtained from the LASY envelope as follows

.. math::
  :label: LasyBrBt

  \tilde{B}_r = \tilde{B}_y\cos\theta + \tilde{B}_z\sin\theta \quad,\quad
  \tilde{B}_\theta = -\tilde{B}_y\sin\theta + \tilde{B}_z\cos\theta

For a linear polarization along ``y``, this reduces to
:math:`\tilde{B}_r = \tilde{B}\sin\theta` and :math:`\tilde{B}_\theta = \tilde{B}\cos\theta`.


As in :doc:`azimuthal_modes_decomposition`, a real field is written

.. math::

  F(\theta) = \textrm{Re}\left[\sum_{m\geq 0} F^m e^{-im\theta}\right]
  \quad,\quad
  F^0 = \frac{1}{2\pi}\int_0^{2\pi} F\, d\theta
  \quad,\quad
  F^m = \frac{1}{\pi}\int_0^{2\pi} F\, e^{im\theta}\, d\theta \quad (m \geq 1)

This decomposition is applied separately to the real and imaginary parts of
:math:`\tilde{B}_r` and :math:`\tilde{B}_\theta`, which are real fields.
Since :math:`\textrm{Re}\left[\tilde{B}\,e^{-i\omega t}\right] =
\textrm{Re}(\tilde{B})\cos(\omega t) + \textrm{Im}(\tilde{B})\sin(\omega t)`,
the injected field of each mode is

.. math::

  B^m_{r}(t, r) = B^m_{r,\textrm{Re}}(t, r)\cos(\omega t) + B^m_{r,\textrm{Im}}(t, r)\sin(\omega t)

  B^m_{\theta}(t, r) = B^m_{\theta,\textrm{Re}}(t, r)\cos(\omega t) + B^m_{\theta,\textrm{Im}}(t, r)\sin(\omega t)

where the subscripts Re and Im denote the modes of the real and imaginary parts of the envelope.

----

Laser envelope: ``LaserEnvelopeFromLasyAM``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :doc:`laser_envelope` describes the laser by the complex envelope :math:`\tilde{A}`
of its transverse vector potential, :math:`\mathbf{A} = \textrm{Re}\left[\tilde{A}\,e^{i\omega(x-t)}\right]`,
whereas LASY provides the envelope :math:`\tilde{E}` of the electric field.

**Azimuthal modes.**
The envelope model assumes a cylindrically symmetric envelope.
Only the mode :math:`m=0` of the LASY laser, :math:`\tilde{E}_0`, is used.

**Polarization.**
The envelope model supports linear and circular polarizations only.
After normalizing the polarization vector, its global phase
:math:`\alpha = \frac{1}{2}\arg(p_x^2 + p_y^2)` is extracted.

* If :math:`(p_x, p_y)\,e^{-i\alpha}` is real, the polarization is linear,
  along the direction of this real vector. The phase :math:`e^{i\alpha}` is included in the envelope.
* If :math:`|p_x| = |p_y|` and :math:`\textrm{Re}(p_x p_y^*) = 0`, the polarization is circular,
  and the envelope is the one of the ``y`` component.
* Other (elliptical) polarizations are not supported.

**From** :math:`\tilde{E}` **to** :math:`\tilde{A}`.
At a fixed position, :math:`\mathbf{E} = -\partial_t\mathbf{A}`. Writing both fields as an envelope
multiplied by :math:`e^{-i\omega t}`, the two envelopes are related by

.. math::
  :label: LasyEnvelopeEA

  \frac{\partial\tilde{A}}{\partial t} = i\omega\tilde{A} - \tilde{E}

(see also Eq. A5 of `F. Massimo et al., Physical Review E (2020) <https://doi.org/10.1103/PhysRevE.102.033204>`_).
For an envelope varying slowly compared with the laser period, :math:`\partial_t\tilde{A}` is negligible
and :math:`\tilde{A} \simeq \tilde{E}/(i\omega)`, i.e. :math:`|\tilde{A}| \simeq |\tilde{E}|/\omega`.
Instead of this approximation, Eq. :eq:`LasyEnvelopeEA` is integrated along the LASY time grid,
of step :math:`\Delta t`, with a centered (trapezoidal) scheme:

.. math::

  \tilde{A}_{n+1} = \frac{\left(1 + i\omega\Delta t/2\right)\tilde{A}_n
  - \Delta t\left(\tilde{E}_n + \tilde{E}_{n+1}\right)/2}{1 - i\omega\Delta t/2}

This scheme is second-order accurate, and its amplification factor has a modulus equal to 1
for any :math:`\omega\Delta t`, so the integration is stable.
The initial value is the slowly varying solution of Eq. :eq:`LasyEnvelopeEA`,
to first order in the time derivative of the envelope:

.. math::

  \tilde{A}_0 = \frac{\tilde{E}_0}{i\omega} + \frac{\partial_t\tilde{E}_0}{(i\omega)^2}

The envelope :math:`\tilde{A}` is then linearly interpolated in :math:`t` and :math:`r`
on the simulation grid, with the same time origin as for ``LaserFromLasyAM``.

----

Limitations
^^^^^^^^^^^

* Only the ``"AMcylindrical"`` geometry is supported.
* The LASY laser must be described in azimuthal modes (``thetaMode`` geometry in LASY).
* The laser can only be injected from ``box_side = "xmin"``.
* ``LaserEnvelopeFromLasyAM`` uses only the mode :math:`m=0`,
  and supports only linear or circular polarizations.
