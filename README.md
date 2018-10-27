# CASINO workflow
This workflow performs a quantum monte carlo (QMC) calculation using "trial" wave function (WFN)
generated by QCHEM and ORCA.

## Authors

* Vladimir Konjkov

## Usage

### Step 1: Install workflow and additional utilities

Download [CASINO QMC](https://vallico.net/casinoqmc/what-is-casino/) program and install it.

Download CASINO workflow from [repository](https://github.com/Konjkov/QMC-snakemake-workflow)

Download [molden2qmc.py](https://github.com/Konjkov/molden2qmc) utility required to convert "trial" WFN from MOLDEN output format to CASINO input format.

Download [multideterminant.py](https://github.com/Konjkov/molden2qmc) utility required to convert multi determinant information available in CASSCF and post-HF methods to CASINO format.

Choose program to generate "trial" WFN (QCHEM > 4.0 or ORCA > 4.0.1) and install it.

Make sure all these programs are accessible through the PATH variable.


### Step 2: Configure workflow

Configure the workflow according to your needs via editing the file `ORCA/Snakefile` or `QCHEM/Snakefile` depending on what program you will use.

The basic information necessary for calculations is contained in `config.yaml` file:

* STD_ERR: desireable accuracy of DMC energy

* VMC_NCONFIG is a number of VMC config in VMC optimization step

* MOLECULES is a list of molecular geometry file names in xyz-format (without extension) located in the `chem_database` directory for which the calculation will be performed.

* METHODS is a list of quantum chemistry methods to obtain a "trial" WFN.

* BASES is a list of bases in which the "trial" function will be represented.

* JASTROWS is a filename of template (without extension) defining JASTROW factor

Depending on the program used to generate "trial" WFN, the following rules are available:

#### ORCA rules:

* generation ORCA output file.
```
rule ALL_ORCA:
    input: '{method}/{basis}/{molecule}/mol.out'
```
where:
* __method__ - method available in ORCA to calculate "trial" WFN like HF, any DFT methods (i.e. B3LYP, CAM-BLYP, PBE0), OO-RI-MP2, OO-RI-SCS-MP2, CASSCF(N.M) for multideterminant extension.
* __basis__ - any basis available in ORCA (i.e. cc-pVDZ, aug-cc-pVQZ, def2-SVP).
* __molecule__ - molecular geometry file name in xyz-format (without extension) located in the `chem_database` directory.

#### QCHEM rules

* generation QCHEM output file.
```
rule ALL_QCHEM:
    input: '{method}/{basis}/{molecule}/mol.out'
```
where:
* __method__ - method available in QCHEM to calculate "trial" WFN including HF, any DFT methods (i.e. B3LYP, CAM-BLYP, PBE0), any orbital optimized methods from the list (OD, OD(2), VOD, VOD(2), QCCD, QCCD(2), VQCCD).
in case of OO-method T2-amplitudes subsequently used as determinant's weights but some type of active space truncation should be specified.\
It should be done with two ways:
    - __OD__ all available T2-amplitudes were taken.
    - __OD_10__ first 10 active orbitals were taken.
    - __OD\_0.01__ all orbitals with T2-amplitudes greater than 0.01 were taken.
* __basis__ - any basis available in QCHEM (i.e. cc-pVDZ, aug-cc-pVQZ, pc-1).
* __molecule__ - molecular geometry file names in xyz-format (without extension) located in the `chem_database` directory.

#### QMC rules

* pure VMC calculation (without JASTROW)
    ```
    rule ALL_VMC:
        input: '{method}/{basis}/{molecule}/VMC/{nstep}/out'
    ```
    where:
    * __nstep__ - number of VMC statistic accumulation steps.

    This rule intended to check whether the conversion of the "trial" WFN to the CASINO format is correct and HF energy is equal to pure VMC one.
    It is not required to calculate the DMC energy.
* JASTROW coefficients optimization using some optimization plan
    ```
    rule ALL_VMC_OPT:
        input: '{method}/{basis}/{molecule}/VMC_OPT/{opt_plan}/{jastrow}/out'
    ```
    where:
    * __opt_plan__ - name (without extension) of CASINO input file in `opt_plan` directory which define JASTROW optimization plan.
    * __jastrow__ - name (without extension) of template for `parameters.casl` file in `casl` directory.
* VMC energy calculation with optimized JASTROW coefficients.
    ```
    rule ALL_VMC_OPT_ENERGY:
        input: '{method}/{basis}/{molecule}/VMC_OPT_ENERGY/{opt_plan}/{jastrow}/{nstep}/out'
    ```
    where:
    * __nstep__ - number of VMC statistic accumulation steps.

    This rule is not required to calculate the DMC energy, but this may be necessary if you need the VMC energy with high accuracy.
* FN-DMC energy calculation to achieve desired accuracy (specified in the variable STD_ERR).
    ```
    rule ALL_VMC_DMC:
        input: '{method}/{basis}/{molecule}/VMC_DMC/{opt_plan}/{jastrow}/tmax_{dt}_{nconfig}_{i}/out'
    ```
    where:
    * __dt__ - part of denominator (integer) to calculate the DMC time step using the formula 1.0/(max_Z<sup>2</sup> * 3.0 * __dt__).
    * __nconfig__ - number of configuration in DMC calculation (1024 is recommended)
    * __i__ - stage of DMC calculation (1 - only DMC equilibration and fixed step (50000) of DMC accumulation run, 2 - additional DMC accumulation run to achieve desired accuracy)
* JASTROW coefficients optimization using some optimization plan with BACKFLOW transformed WFN.
    ```
    rule ALL_VMC_OPT_BF:
        input: '{method}/{basis}/{molecule}/VMC_OPT_BF/{opt_plan}/{jastrow}__{backflow}/out'
    ```
    where:
    * __backflow__ - combination of {ETA-term}\_{MU-term}\_{PHI-term-electron-nucleus}{PHI-term-electron-electron} expansion orders:
        - 3_3_00 - ETA-term expansion of order 3 and MU-term of order 3, no PHI-term
        - 9_9_33 - ETA-term expansion of order 9 and MU-term of order 9, PHI-term with electron-nucleus and electron-electron expansion of order 3 both.

    It should be noted that the BACKFLOW transformation in CASINO is implemented only up to f-orbitals.
* VMC energy calculation with optimized JASTROW coefficients with BACKFLOW transformation.
    ```
    rule ALL_VMC_OPT_ENERGY_BF:
        input: '{method}/{basis}/{molecule}/VMC_OPT_ENERGY_BF/{opt_plan}/{jastrow}__{backflow}/{nstep}/out'
    ```
    where:
    * __nstep__ - number of VMC statistic accumulation steps.

    This rule is not required to calculate the DMC energy.
* FN-DMC energy calculation to achieve desired accuracy (specified in the variable STD_ERR) with BACKFLOW transformed WFN.
    ```
    rule ALL_VMC_DMC_BF:
        input: '{method}/{basis}/{molecule}/VMC_DMC_BF/{opt_plan}/{jastrow}__{backflow}/tmax_{dt}_{nconfig}_{i}/out'
    ```

### Step 3: Execute workflow

All you need to execute this workflow is to install Snakemake via the [Conda package manager](http://snakemake.readthedocs.io/en/stable/getting_started/installation.html#installation-via-conda). Software needed by this workflow is automatically deployed into isolated environments by Snakemake.

Test your configuration by performing a dry-run via

    snakemake <rule> -n

Execute the workflow locally via

    snakemake <rule>

## Examples

To demonstrate the possibilities of this workflow, examples of calculations are given.

### ORCA

#### (U)HF/def2-QZVP "trial" WFN for H-Ar atoms

Since the DMC calculation time for achieving a given accuracy increases ~ Z<sup>5.5</sup>, I used several configuration files to perform calculations:
[config_00001.yaml](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/config_00001.yaml) for He-Li,
[config_0001.yaml](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/config_0001.yaml) for Be-N,
[config_003.yaml](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/config_0003.yaml) for O-Si,
[config_001.yaml](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/config_001.yaml) for P-Ar.

The [VMC optimization plan](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/opt_plan/emin.tmpl) was used in the first step performs
__varmin__ optimization and then __emin__ optimization for the remaining eight steps. This is a fairly effective optimization plan, as shown in the
[graphs](https://github.com/Konjkov/QMC-snakemake-workflow/blob/master/ORCA/HF/def2-QZVP/vmc_opt.ipynb), in all cases, it would be enough only 4-5 steps.

Other useful graphs summarizing the results of this calculations are given.

* [__H__ (<sup>2</sup>S<sub>1/2</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/H)

  The Hartree–Fock (HF) "trial" WFN for the ground state of H-atom is nodeless, so DMC energy is exact, also as no electron correlations are present VMC energy is also exact.

  &Psi;<sub>HF</sub>(R) = &psi;<sub>1s</sub>(r<sub>1</sub>)

* [__He__ (<sup>1</sup>S<sub>0</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/He)

  Spacial part of The Hartree–Fock (HF) "trial" WFN for the ground state of He-atom is symmetric so has no nodal surface, thus DMC energy is exact.

  &Psi;<sub>HF</sub>(R) = &psi;<sub>1s</sub>(r<sub>1</sub>) &bull; &psi;<sub>1s</sub>(r<sub>2</sub>)

* [__Li__ (<sup>2</sup>S<sub>1/2</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/Li)

  For The Hartree–Fock (HF) "trial" WFN electrons nodal surface is determined by equation r<sub>1</sub> = r<sub>2</sub> 
  when 1 and 2 label the electrons in the same spin channel [[1](https://doi.org/10.1103/PhysRevB.92.045122)].

  &Psi;<sub>HF</sub>(R) = det[&psi;<sub>1s</sub>(r<sub>1</sub>), &psi;<sub>2s</sub>(r<sub>2</sub>)] &bull; &psi;<sub>1s</sub>(r<sub>3</sub>)

  The electron 1 therefore “sees” the node as a sphere which passes through the position of electron 2 and is centered around the nucleus.
  The wave function will be equal to zero if electron 1 occupies any point on the spherical nodal surface.
  For the correlated electrons this is not strictly exact, as the correlation with the electron in the spin-down channel will cause deformations away from a perfect sphere.
  For example, the excitation 1s<sup>2</sup>2s<sup>1</sup>2p<sup>2</sup> will have a contribution to the exact ground state and would in principle lead to a departure from the single particle node
  (i.e., the sphere will slightly deform to ellipsoid or perhaps a more complicated surface that would depend on the position of the minority spin electron).
  It is therefore quite remarkable that the HF nodal surface seems to be so accurate: the total energy with the HF nodes, is accurate to ~ 0.05 mHa.

* [__Be__ (<sup>1</sup>S<sub>0</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/Be)

  The Hartree–Fock (HF) "trial" WFN for Be is given by a Slater determinant which is block diagonal in spin so it can be broken into a product of the spin channels [[2](https://doi.org/10.1016/j.cplett.2012.01.016)].

  &Psi;<sub>HF</sub>(R) = det[&psi;<sub>1s</sub>(r<sub>1</sub>), &psi;<sub>2s</sub>(r<sub>2</sub>)] &bull; det[&psi;<sub>1s</sub>(r<sub>3</sub>), &psi;<sub>2s</sub>(r<sub>4</sub>)]

  nodal surface is determined by equation (r<sub>1</sub> - r<sub>2</sub>)(r<sub>3</sub> - r<sub>4</sub>) = 0 which clearly shows that there are 2 * 2 = 4 nodal pockets.
  However, it has been found that for ground state the correct number of nodal domains is two.
  The accurate nodal surface for this system is actually remarkably well described by a two configuration wave function where HF WFN is augmented by adding [1s<sup>2</sup>2p<sup>2</sup>](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/CASSCF(2.4)/def2-QZVP/Be) double excitation which corresponds to a near-degeneracy effect.

* [__B__ (<sup>2</sup>P<sub>1/2</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/B)

    The Hartree–Fock (HF) "trial" WFN for B also should be improved by adding [1s<sup>2</sup>2p<sup>3</sup>](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/CASSCF(3.4)/def2-QZVP/B) configuration, however,
    even after this improvement fixed node error is ~ 5 mHa and additional configurations is required to achieve chemical accuracy.
    The article [[3](https://doi.org/10.1142/3357)] __chapter 5.__ _Quantum Monte Carlo Calculations with Multi-Reference Trial Wave Functions (H-J Flad et al.)_
    points out that next important configuration is [1s<sup>2</sup>2s<sup>1</sup>2p<sup>1</sup>3d<sup>1</sup>](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/CASSCF(3.8)/def2-QZVP/B)
    which reduces the error to ~ 1.7 mHa.

* [__C__ (<sup>3</sup>P<sub>0</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/C)

    The Hartree–Fock (HF) "trial" WFN for C also should be improved by adding [1s<sup>2</sup>2p<sup>4</sup>](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/CASSCF(4.4)/def2-QZVP/C) configuration, but this again does not give us an exact nodal surface.

* [__N__ (<sup>4</sup>S<sub>3/2</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/N)

    Only single determinant approximation was used.

* [__O__ (<sup>3</sup>P<sub>2</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/O)

    Only single determinant approximation was used.

* [__F__ (<sup>2</sup>P<sub>3/2</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/F)

    Only single determinant approximation was used.

* [__Ne__ (<sup>1</sup>S<sub>0</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/Ne)

    Only single determinant approximation was used.

* [__Na__ (<sup>1</sup>S<sub>0</sub>)](https://github.com/Konjkov/QMC-snakemake-workflow/tree/master/ORCA/HF/def2-QZVP/Na)

    Only single determinant approximation was used.

### QCHEM
