==================================
Changes in this fork of SkProgs
==================================

Fork: https://github.com/lucienwb/skprogs, branched from upstream
https://github.com/dftbplus/skprogs at commit 0d5edc7. One entry per change, newest first: what
changed, why, and how it was checked, so the changes can be reviewed without reading the diff.


2026-10-02 -- sktwocnt: faster two-centre integration
=====================================================

What changed
------------

Files: ``sktwocnt/lib/twocnt.F90``, ``sktwocnt/lib/gridorbital.f90``. Results are unchanged to
round-off; the integration grid and all input settings are untouched.

* ``getDivergence`` (GGA potential term): the cubic-spline derivative along each grid line is a
  linear map of the values on that line, and the radial and angular grid nodes do not depend on
  the dimer distance. The map is now built once as a matrix (``getSplineDerivMatrix``, from the
  existing ``spline3ders``) and applied to all lines with ``dgemm``. Before, a banded spline system
  was set up and solved for every line at every distance.
* GGA case of ``getskintegrals``: the exchange and correlation divergence terms are computed in one
  call on ``vxsigma + vcsigma``; the term is linear in vsigma.
* Radial interpolation: every ``TGridorb2`` function (basis functions and their derivatives,
  density, potential) lives on the same 10000-point grid. The interpolation stencil and its
  4-point Lagrange weights are therefore computed once per grid point and distance
  (``getStencil``) and reused for every function (``getValueStencil``). This is the same
  interpolating polynomial that ``polyinter`` evaluates.
* ``getStencil`` stops with an error for radii beyond the interpolation grid. In that range
  ``TGridorb2_getValue`` uses ``rmax`` uninitialised and does not return early for ``rr > rcut``;
  the range is not reached with the current grids (largest grid radius about 9.4e4 bohr, the
  branch starts at about 4.1e7 bohr). ``TGridorb2_getValue`` itself is left unchanged.

Why
---

A callgrind profile (PBE, density superposition, C-C) attributed 42 % of the instructions to the
spline set-up and banded solves in ``getDivergence`` and 44 % to the per-function interpolation.
libxc itself was 6.5 %.

How it was checked
------------------

Linux, gfortran 13, libxc 7.1.2, OpenBLAS, one core, ``examples/mio/skdef.hsd``. Wall times of
``sktwocnt`` and the largest absolute difference of the Hamiltonian (Ha) and overlap tables
against the unmodified code:

=========================================  ========  =======  ===========  ===========
case                                       before    after    max dH       max dS
=========================================  ========  =======  ===========  ===========
C-C, PBE, density superposition, 0.4-10    110 s     30.4 s   1.0e-13      0
C-H, PBE, density superposition, 0.4-10    96.5 s    26.7 s   1.0e-15      0
C-H, LDA, potential superposition, 0.4-2   5.4 s     1.6 s    0            0
C-H, PBE, potential superposition, 0.4-2   5.6 s     1.6 s    1.0e-13      0
C-H, LCY-PBE (omega 0.3), density, full    216.5 s   145.9 s  0            0
C-H, B3LYP, density, full                  148.6 s   98.4 s   1.0e-13      1.0e-19
=========================================  ========  =======  ===========  ===========

(0 means no difference at the precision written by sktwocnt.) The generated C-C table
reproduces mio-1-1: onsite energies to 1e-7 Ha, H to about 1e-5 Ha, S to 1e-8.

Test suite: ``ctest`` 11/11 passed (LDA-PW91 x4, GGA-PBE96, HYB-PBE0, HYB-B3LYP, LCY-BNL,
LCY-PBE96, CAMY-B3LYP, CAMY-PBEh).
