<!-- BEGIN COMMENT -->

[<< Previous Chapter](CMAQ_UG_ch05_compile_and_run.md) - [Home](README.md) - [Next Chapter >>](CMAQ_UG_ch07_model_outputs.md)

<!-- END COMMENT -->

# 6. Model Configuration Options

## 6.1 Introduction
As discussed in [Chapter 1](CMAQ_UG_ch01_overview.md), CMAQ is a multipollutant, multiscale air quality modeling system that estimates the transport and chemistry of ozone, PM, toxic airborne pollutants, and acidic and nutrient pollutant species, as well as visibility degradation and deposition totals. CMAQ includes state-of-the-art technical and computational techniques to simulate air quality from urban to global scales. It can model complex atmospheric processes affecting transformation, transport, and deposition of air pollutants using a system architecture that is designed for fast and efficient computing. (See [Appendix D](Appendix/CMAQ_UG_appendixD_parallel_implementation.md) for an introduction on how data-parallelism can be applied in the CMAQ system to increase computational efficiency.) This chapter presents a brief overview of the conceptual formulation of Eulerian air quality modeling and the science features in various components of the Chemistry-Transport Model (CTM) component of CMAQ, CCTM. 

## 6.2 Numerical Approach
The theoretical basis for CMAQ’s formulation is the conservation of mass for atmospheric trace species emissions, transport, chemistry, and removal in the atmosphere. The general form of a chemical species equation derives from this conservation, so that changes in atmospheric concentrations of a species, C<sub>i</sub>, can mathematically be represented as

![Equation 8-1](images/Figure8-1.JPG)  

where the terms on the right-hand side of the equation represent the rate of change in C<sub>i</sub> due to advection, turbulent mixing, cloud processes (mixing, scavenging, and aqueous-phase chemistry), dry deposition, and aerosol processes (phase partitioning, and aerosol dynamics). R<sub>gi</sub> represents the rate of change due to gas and heterogeneous chemical reactions, while E<sub>i</sub> is the emission rate for that species. The mass conservation for trace species and the moment dynamic equations for the various modes of the particulate size distribution in CMAQ are further formulated in generalized coordinates, where in the same formulation allows the model to accommodate the commonly used horizontal map projections (i.e., Lambert conformal, polar stereographic, and Mercator) as well as different vertical coordinates (see Chapters 5 and 6 in Byun and Ching, 1999). The governing equation for CMAQ is numerically solved using the time-splitting or process splitting approach wherein each process equation is solved sequentially, typically with the process with the largest time-scale solved first. 

## 6.3 Grid Configuration
CMAQ is a three-dimensional Eulerian air quality model. To solve the governing partial differential equations, the domain of a model run (the volume of the atmosphere over a geographic region) is discretized with three-dimensional cells. The grid cells and boundaries of domain must be rigorously and consistently defined for all functional components of the model (e.g., chemistry, emissions, meteorology). 
CMAQ’s generalized coordinate formulation maps the physical space to the computational space. The horizontal grid specification (setting the x and y dimensions) must be regular: the horizontal projection of each grid cell (sometimes referred to as a pixel) has the same resolution, and the boundaries of each pixel are time-invariant. By contrast, the vertical grid specification (setting the z dimension) need not be regular.

After determining the horizontal and vertical extent of the domain of interest, a meteorological model must be run for a horizontal domain slightly larger than the CMAQ domain. A larger meteorology domain is necessary for distinguishing the meteorological boundary conditions from the CMAQ boundary conditions.

Available horizontal grids for a given CMAQ run are defined at runtime by setting the GRIDDESC and GRID_NAME environment variables to point to an existing grid definition file and to one of the grids defined in the file, respectively. Horizontal grids are defined by the
grid definition file (GRIDDESC), which can be edited by the user.  Further details on grid configuration are available in the [README.md](../../PREP/mcip/README.md) file in the PREP/mcip folder.

## 6.4 Science Configurations
CCTM contains several science configurations for simulating transport, chemistry, and deposition. All of the science configuration options in CCTM, such as the chemical mechanism to be used, are set when compiling the executable. The model grid and vertical layer structure for CCTM are set at execution. The important distinction between selecting the science configuration and the model grid/layer configuration is that CCTM does not need to be recompiled when changing model grids/layers but does need to be recompiled when new science options are invoked.  The following sections describe how these science options can be utilized by configuring using the `bldit_cctm.csh` and `run_cctm.csh` scripts.  For the remainder of this chapter these files will be referred to as simply BuildScript and RunScript.

**>>COMMENT<<** Each of the following sections should provide a brief description of the science process followed by the specific configuration options that need to be set in both the build script (BuildScript) and run script (RunScript). The information on these environment variables should be consistent with what is provided in Appendix A.  When specifying environment variable options please use the following syntax:
```
setenv ModDepv depv/m3dry
```

## 6.5 Advection
In CCTM, the 3-dimensional transport by mean winds (or advection) is numerically represented by sequentially solving locally-one dimensional equations for the two horizontal and vertical components. CMAQ uses the piecewise parabolic method (PPM) (Colella and Woodward, 1984) for representing tracer advection in each of the three directions. This algorithm is based on the finite-volume sub-grid definition of the advected scalar. In PPM, the sub-grid distribution is described by a parabola in each grid interval. PPM is a monotonic and positive-definite scheme. Positive-definite schemes maintain the sign of input values, which in this case means that positive concentrations will remain positive and cannot become negative.

Mass consistency is a key desired attribute in tracer advection. Data consistency is maintained for air quality simulations by using dynamically and thermodynamically consistent meteorology data from WRF/MCIP. Mass inconsistencies can nevertheless arise either through the use of different grid configurations (horizontal or vertical) or due to differing numerical advection schemes between the driving meteorological model and the CCTM. While inconsistencies due to the former can be eliminated through use of the same grid configurations (thus, layer collapsing is not recommended), some inconsistencies can still remain due to differing numerical representations for satisfying the mass-continuity equation between the driving meteorological model and the CCTM. These mass-inconsistencies manifest as first order terms (whose magnitude can often be comparable to tracer lifetimes if continuity is not satisfied with high accuracy) that can artificially produce or destroy mass during 3D tracer advection (e.g., Mathur and Peters, 1990).

CMAQ has two options that minimize mass consistency errors in tracer advection. In one scheme, that was first implemented in CMAQv4.5 and later improved for CMAQv4.7.1, designated “yamo” in the BLD  script, CMAQ  advects air density and re-diagnoses the vertical velocity field according to the layer-by-layer mass continuity equation which guarantees that the CCTM advected density matches that derived from the driving meteorological inputs (e.g., Odman and Russell, 2000). Briefly, x- and y-advection are first performed (the order of these is reversed every step to minimize aliasing errors) to yield intermediate tracer and density fields. The intermediate density field is then subject to vertical advection with the PPM scheme such that it yields the WRF derived density field at the end of the advection time-step. This scheme results in an estimated vertical velocity field that is minimally adjusted relative to the WRF derived field in the lower model layers but yields strict mass-consistent tracer advection in CMAQ.  A drawback to this approach is that erroneous noise in the diagnosed vertical velocity field accumulates toward the top of the model with non-zero velocity and mass flux across the top boundary.  The noise in the vertical velocity field causes excessive diffusion in upper layers.  Therefore, starting in CMAQv5.0, a new scheme was implemented, designated “wrf”, that closely follows the vertical velocity calculation in WRF.  This scheme solves the vertically integrated mass continuity equation such that the collunm integrated horizontal mass divergence is balanced by the net change in column mass (Skamarock et al, 2019).  An advantage of this scheme is that the diagnosed vertical velocity agrees more closely with the WRF vertical velocity field with zero velocity and mass flux across the upper model boundary.  Thus, the spurious velocity noise and excessive diffusion in the upper layer are eliminated.  The main drawback of this scheme is that mass conservation is not guaranteed so density must be updated from the meteorology inputs every timestep.  

The **“WRF”** option is the recommended configuration for CMAQv5.3.

To invoke the "WRF" option in 3-D advection, the build script has to set the following within the CCTM Science Modules section:

```
set ModDriver = driver/wrf
set ModCpl    = couple/gencoor_wrf
set ModHadv   = hadv/ppm  
set ModVadv   = vadv/wrf
```
To invoke the "YAMO" option in 3-D advection, the build script has to set the following within the CCTM Science Modules section: 
```
set ModDriver = driver/yamo
set ModCpl    = couple/gencoor
set ModHadv   = hadv/ppm  
set ModVadv   = vadv/yamo
```
## 6.6 Horizontal Diffusion
Horizontal diffusion is implemented with a single eddy diffusion algorithm that is based on local wind deformation and is scaled to the grid cell size. The horizontal eddy diffusivity is assumed to be uniform but dependent on the grid resolution of the model. This diffusivity is larger for a higher-resolution run where the numerical diffusion due to the advection process is smaller.

## 6.7 Vertical Diffusion
The vertical diffusion model in CMAQ is the Asymmetrical Convective Model Version 2 (ACM2) (Pleim 2007a,b).  The ACM2 is a combined local and non-local closure PBL scheme that is implemented in CMAQ, WRF, and MPAS for consistent PBL transport of meteorology and chemistry.  Thus, it is recommended that the ACM2 option in WRF or MPAS also be used when preparing meteorology for CMAQ.  

There are two options for the ACM2 model in the build script that are compatible with either the M3Dry or STAGE dry deposition options.  

When running m3dry dry deposition:

```
Set ModVdiff   = acm2_m3dry
```

When running STAGE dry deposition:

```
Set ModVdiff   = acm2_stage
```
  
## 6.8 Dry Deposition
The dry deposition model in CMAQ was originally based on the dry deposition model developed for the Acid Deposition and Oxidant Model (ADOM) (Pleim et al., 1984).  Dry deposition is computed by electrical resistance analogy where concentration gradients are analogous to voltage, flux is analogous to current, and deposition resistance is analogous to electrical resistance (Pleim and Ran, 2011).  In CMAQ, several key resistances, such as aerodynamic resistance and bulk stomatal resistance, and other related parameters, such as LAI, vegetation fraction, roughness length, friction velocity etc., are expected to be provided from the meteorological inputs.  Use of common model elements and parameters with the land surface model in the meteorology model ensures consistency between chemical surface fluxes and meteorological surface fluxes (moisture, heat, momentum).  While the dry deposition model was designed to be used with the PX LSM option in WRF, any LSM can be used if the necessary parameters are output and then provided for input into CMAQ.  

Since ammonia can be both emitted from the surface and deposited, ammonia surface flux is modeled as bidirectional.  In North America, the upward flux of ammonia is greatest in agricultural areas during the growing season.  Fertilizer adds large amounts of nitrogen compounds to the soil in croplands some of which volatilizes as ammonia.  Estimates of the soil and stomatal compensation concentrations are needed to compute the bidirectional ammonia flux in CMAQ.  While stomatal concentrations are estimated according to landuse type, daily estimates of soil concentration are derived from input proved by the Environmental Policy Integrated Climate (EPIC) agricultural ecosystem model that is executed using the Fertilizer Emission Scenario Tool for CMAQ (FEST-C, https://www.cmascenter.org/fest-c ) (Ran et al., 2011; Cooter et al., 2012).

The two options for dry deposition are invoked as:

```
Set ModDepv   = m3dry
```

or:

```
Set Set ModDepv   = stage
```

### 6.8.1 Dry Deposition - m3dry
The m3dry option for dry deposition and ammonia bidirectional surface flux is the continuing evolution of the default dry deposition model in CMAQ since its initial release.  M3dry is designed for compatibility with land surface models, particularly the PX LSM, used in WRF and MPAS to simulate surface processes and fluxes.  It features consideration of subgrid land-use fractions through aggregation of key model parameters, such as LAI, veg fraction, roughness length and minimum stomatal conductance, to the grid cell level.  Land use specific dry deposition fluxes can be calculated using a postprocessing program.

Upgrades for version 5.3 includes much greater surface resistances to snow and ice and reduced resistance to bare ground for ozone with dependence on surface soil moisture content.  The aerosol deposition has also been revised including a new dependence on LAI.  The ammonia bidirectional surface flux from croplands has been substantially revised from earlier versions.  The new version has close linkages with the EPIC agricultural ecosystem model.  Daily values of all soil parameters needed to compute the available soil ammonia concentrations (soil ammonia content, soil moisture, soil texture parameters, soil pH, and Cation Exchange Capacity (CEC)) for each of 21 agricultural production types that are either rainfed or irrigated (42 types total) are input to CMAQ.  Soil ammonia concentrations and soil pH are combined to derive the soil compensation concentration for the bidirectional flux calculation (Pleim et al, 2019).

### 6.8.2 Dry Depostion - STAGE

## 6.9 Emissions

CMAQ introduces emissions of trace gases and aerosols from a variety of important sources (e.g. electric generating utilities, vehicles, fires, trees, dust storms, farms, etc.). Some emissions are applied in the surface layer of the model grid, while others are applied at higher altitudes if, for example, they originate from point source like an elevated stack, or a large forest fire. Many sources that are related to local meteorology may be calculated online in CMAQ. However, most sources, especially anthropogenic ones, are preprocessed using software like the Sparse Matrix Operator Kerner Emissions (SMOKE) Modeling System. Once these external tools have calculated the offline emissions, they may merge them into larger aggregated files. We refer to emissions that are either calculated online or read into CMAQ from a file as emission "streams".

Because CMAQ represents both primary and secondary pollutants, emissions are processed for a subset of the species CMAQ treats. The emissions chemical speciation must be compatible with the chemical mechanism chosen for CMAQ (e.g. cb6r3_ae7_aq) because different mechanisms represent large compounds like functionalized hydrocarbons with different formulae. CMAQv5.3 has introduced new features that make the process of mapping emissions species to CMAQ species more transparent and flexible (see Emission Control with DESID[link]). In fact, users can now toggle, modify, and augment emissions from all available streams in order to better tailor their simulations to the questions they are asking CMAQ to help answer. For tutorials covering specific tasks, please see the DESID tutorial page [link].

### 6.9.1 Emission Streams
Depending on the nature of any stream and the information used to quantify its emissions, it may be treated as one of three types:

#### Online Stream:
CMAQ will calculate the emission rates from this source using information about local meteorology, land characteristics, etc. The streams available for running Online in CMAQ are: biogenics (BEIS) [link], marine gas[link], lightning NO [link], wind-blown dust [link], and sea-spray[ link].

#### Gridded Stream (offline):
CMAQ will read emission rates from an input file, which is organized into an array that is identical in shape to the grid CMAQ is running. Typically these rates are stored at hourly time points and are then interpolated within CMAQ to each time step. Some common examples of Gridded emissions include:

- Mobile sources such as passenger vehicles, trains, ships, scooters, etc.
- Low-level point source emissions that are not large enough to be treated individually
- Residential heating
- Consumer product use (e.g. adhesives, personal care products, pesticides, etc.)
- Agricultural (e.g. burning, dust, animal waste, etc.)
- Road, Construction and mechanically generated dust
- Biogenic VOCs (if not calculated online with BEIS)

Users add Gridded emissions to a simulation via the RunScript. First the variable N_EMIS_GR must be set to the number of Gridded Streams to be used:

```
setenv N_EMIS_GR 3
```

The RunScript must also specify the location of the input files using three-digit suffixes for the stream number:

```
setenv GR_EMIS_001 /home/user/path-to-file/emiss_stream_1_${DATE}.nc
```

and the short-name label to be used to refer to the Stream in logfiles:

```
setenv GR_EMIS_LAB_001 MOBILE
```
If N_EMIS_GR is set 0, then CMAQ will run with no Gridded emissions even if the values for GR_EMIS_XXX and GR_EMIS_LAB_XXX are all set.

#### Inline Stream (offline):
For these streams, emission rates and stack characteristics are provided for many individual sources on the same file. CMAQ uses the stack information to calculate important quantities like the injection height online taking into account local meteorology. A specific latitude/longitude pair is given for each source to locate it in the CMAQ grid.  Some common examples of Inline emissions include:

- Stacks (electric generation units, industrial sources, manufacturing, etc.)
- Forest fires
- Large prescribed fire events  

Users add Inline emissions to a simulation via the RunScript. First the variable N_EMIS_PT must be set to the number of Inline Streams to be used:

```
setenv N_EMIS_PT 3
```
The RunScript must also specify the location of the input files using three-digit suffixes for the stream number:

```
setenv STK_EMIS_002 /home/user/path-to-file/inline_emiss_stream_2_${DATE}.nc
```

The location to the "stack file" with static information about the properties of each source on the stream:
```
setenv STK_GRPS_002 /home/user/path-to-file/inline_stack_groups_2.nc
```
and the short-name label to be used to refer to the Stream in logfiles:
```
setenv STK_EMIS_LAB_002 POINT_FIRES
```
If N_EMIS_PT is set 0, then CMAQ will run with no Inline emissions even if the values for STK_EMIS_XXX, STK_GRPS_XXX and STK_EMIS_LAB_XXX are all set.

### 6.9.2 Online Emission Streams
#### BEIS
#### Plume Rise 
#### Wind-Blown Dust
#### Sea Spray
#### Lightning NO

## 6.10 Gas Phase Chemistry
### 6.10.1 Gas Phase Chemical Mechanisms
The CMAQ modeling system accounts for chemistry in three phases: a gas phase, aerosols (solid or liquid), and an aqueous phase. Refer to the release notes to find the gas‑phase chemistry mechanisms available in each version of CMAQ. Several variations of the base gas-phase mechanisms, with and without chlorine, mercury, and toxic species chemistry, are distributed with CMAQ. The modularity of CMAQ makes it possible to create or modify the gas-phase chemical mechanism.

Gas-phase chemical mechanisms are defined in CMAQ through Fortran source files. Located in subdirectories of the CCTM/src/MECHS directory (each corresponding to a mechanism name), these files define the source, reaction parameters, and atmospheric processes (e.g., diffusion, deposition, advection) of the various mechanism species. The species definitions for each mechanism are contained in namelist files that are read in during execution of the CMAQ programs. The CMAQ mechanism configuration is more similar to the science module configuration than to the horizontal grid or vertical layer configuration in that the mechanism is defined at compilation, resulting in executables that are hard-wired to a specific gas-phase mechanism. To change chemical mechanisms between simulations, a new executable that includes the desired mechanism configuration must be compiled.

#### Using predefined chemical mechanisms
To select a predefined mechanism configuration in CMAQ, set the *Mechanism* variable in the build scripts to one of the mechanism names listed in Table 6.1. 

```
 set Mechanism = MECHANISM_NAME
```

Refer to the [README.md](../../CCTM/src/MECHS/README.md) under CCTM/src/MECHS for detailed information reactions and on model species names for each mechanism. 

**Table 6.1**  Chemical Mechanisms available with CMAQv5.3.  Atmospheric chemistry mechanisms of varying complexity are available to support diverse applications across scales and explore extensions for emerging problems and contaminants.

|**Mechanism Name** | **Comment** |
| ----------------- | ---------------------------------------------------- |
| cb6r3_ae7_aq      | Carbon Bond 6 version r3 with aero7 treatment of SOA set up for standard cloud chemistry |
| cb6r3_ae7_aqkmt2    | Carbon Bond 6 version r3 with aero7 treatment of SOA set up for expanded organic cloud chemistry version 2  |
| cb6r3m_ae7_kmtbr  | Carbon Bond 6 version r3 with aero7 treatment of SOA and DMS and marine halogen chemistry set up for expanded organic and halogen cloud chemistry  | 
| cb6r3_ae6_aq      | Carbon Bond 6 version r3 with aero6 treatment of SOA set up for with standard cloud chemistry | 
| cb6mp_ae6_aq      | Carbon Bond 6 version r3 with air toxics and aero6 treatment of SOA set up for standard cloud chemistry | 
| racm2_ae6_aq      | Regional Atmospheric Chemistry Mechanism version 2 with aero6 treatment of SOA set up for with standard cloud chemistry |
| saprc07tic_ae7i_aq | State Air Pollution Research Center version 07tc with extended isoprene chemistry and aero7i treatment of SOA set up for with standard cloud chemistry | 
| saprc07tic_ae7i_aqkmt2 | State Air Pollution Research Center version 07tc with extended isoprene chemistry and aero7i treatment of SOA for expanded organic cloud chemistry version 2  |
| saprc07tic_ae6i_aq | State Air Pollution Research Center version 07tc with extended isoprene chemistry and aero6i treatment of SOA set up for standard cloud chemistry | 
| saprc07tic_ae6i_aqkmti | State Air Pollution Research Center version 07tc with extended isoprene chemistry and aero6i treatment of SOA for expanded organic cloud chemistry for isoprene  | 
| saprc07tc_ae6_aq | State Air Pollution Research Center version 07tc with aero6 treatment of SOA set up for with standard cloud chemistry  | 


### 6.10.2 Solvers
To solve the photochemistry, the model uses one of three numerical methods or solvers. They differ by accuracy, generalization, and computational efficiency, i.e. model run times. Options include Euler Backward Iterative (EBI) solver (Hertel et al., 1993),  Rosenbrock (ROS3) solver (Sandu et al., 1997), and Sparse Matrix Vectorized GEAR (SMVGEAR) solver (Jacobson and Turco, 1994). The EBI solver is default method because it is the fastest but is less accurate and must be _tailored_ for each mechanism. The build script defines which EBI solver to use as below.   

```
 set ModGas    = gas/ebi_${Mechanism} 
``` 
 
If a user creates new FORTRAN modules representing the photochemical mechanism or modifies the existing modules, they have to create a new EBI solver by the using the create_ebi utility.  Documentation on compiling and running create_ebi is available under the [UTIL/create_ebi](../../UTIL/create_ebi/README.md) folder. The remaining two solvers, SMVGEAR and ROS3, are more accurate and less prone to convergence errors. Both methods are labeled as “generalized” because they only require the mechanism’s namelist and FORTRAN modules representing the photochemical mechanism. Rosenbrock is preferred over SMVGEAR because it several times faster. To use either SMVGEAR and ROS3, the build script defines ModGas as below. 

```
 set ModGas    = gas/smvgear
```   

or

```
 set ModGas    = gas/ros3
``` 

 
### 6.10.3 Photolysis

All the mechanism include photolysis rates. The build script has two options for calculating the rates.

```
 set ModPhot    = phot/inline
```   

or

```
 set ModPhot    = phot/table
``` 

The in-line method (Binkowski et al., 2007) is the preferred option because it includes feedbacks from meteorology, predicted ozone and aerosol concentrations. Three ASCII files support the in-line method. **PHOT_OPTICS** describes the optical properties of clouds, aerosols, and the earth’s surface. The **OMI** file is used to determine how much light is absorbed by ozone above the model domain. Both files are included in the released version of CMAQ. Calculating photolysis rates uses one more file, the **CSQY_DATA_${Mechanism}** file, that depends the Mechanism used. It contains the cross sections and quantum yields of photolysis rates used by the Mechanism. The files are provided for each mechanisms in a released version of CMAQ. If a user creates a Mechanism using new or additional photolysis rates, they have to create a new **CSQY_DATA_${Mechanism}** file. The [inline_phot_preproc utility](../../UTIL/inline_phot_preproc/README.md) produces this file based on the Fortran modules describing the Mechanism and data files describing the absorption cross-section and quantum yields described for each photolysis reaction. The CCTM run script set values for each file's path through the environment variables, OPTICS_DATA, OMI, and CSQY_DATA.

The other option uses look-up tables that contain photolysis rates under cloud free conditions based on a fixed meridional cross-sections of atmospheric composition, temperature, density and aerosols. The values represents rates as a function of altitude, latitude and the hour angle of the sun on a specified Julian date. In model simulations, the method interpolates rates in the table for the date and corrects them to account for clouds described by the meteorology. Tables are dependent on the photochemical mechanism used. The [jproc utility](../../UTIL/jproc/README.md) creates them based on the photochemical mechanism's FORTRAN modules. The CCTM run script set value for a table's path with the environment variable, XJ_DATA.


## 6.11 Aerosol Dynamics and Chemistry

Particulate Matter (PM) can be either primary (directly emitted) or secondary (formed in the atmosphere) and from natural or anthropogenic (man-made) sources. Secondary sources include gas-phase oxidation of SO<sub>2</sub> to sulfate, condensation of ammonia and nitrate, and oxidation of gas-phase VOCs such as isoprene, monoterpenes, aromatics, and alkanes. Cloud processes also contribute to the formation of PM; for example, aqueous oxidation of sulfur dioxide in cloud droplets is a significant pathway for production of particulate sulfate. CCTM represents PM using three interacting lognormal distributions, or modes. Two modes (Aitken and accumulation) are generally less than 2.5 microns in diameter while the coarse mode contains significant amounts of mass above 2.5 microns. PM<sub>2.5</sub> and PM<sub>10</sub>, species aggregate metrics within the NAAQS, can be obtained from the model mass concentration and size distribution information.

The 6th generation CMAQ aerosol module (AERO6) was introduced in CMAQv5.0.2 and expanded the chemical speciation of PM. Eight new PM species were added to CMAQ in AERO6: Al, Ca, Fe, Si, Ti, Mg, K, and Mn. Four species that were explicitly treated in previous versions of CMAQ but were not modeled can now be treated as primary anthropogenic species: H2O, Na, Cl, and NH4. The PM emissions mass that remains after speciation into the new components is now input to the model as PMOTHER. AERO6 requires 18 PM emissions species: OC, EC, sulfate, nitrate, H<sub>2</sub>O, Na, Cl, NH<sub>4</sub>, NCOM, Al, Ca, Fe, Si, Ti, Mg, K, Mn, and Other (Reff et al., 2009).

Aero6 mechanisms available in CMAQv5.3 are compatible with semivolatile primary organic aerosol (POA). For the nonvolatile POA configuration, mass is tracked separately in terms of its carbon (OC) and non-carbon (NCOM) content. With this approach, mass can be added to the non-carbon species to simulate the aging of POA in response to atmospheric oxidants. Simon and Bhave (2011) document the implementation of the second-order reaction between primary organic carbon and OH radicals. The semivolatile POA configuration segregates POA into several model species based on a combination of volatility and oxidation state (Murphy et al., 2017). There are five POA species at low oxidation state representing low volatility, semivolatile and intermediate volatility compounds (LVPO1, SVPO1, SVPO2, SVPO3, IVPO1). As the gas-phase species (e.g. VLVPO1) oxidize with OH they form species with higher oxidation state (i.e. LVOO1, LVOO2, SVOO1, SVOO2, SVOO3). The multigenerational aging chemistry for the semivolatile POA configuration is derived from the approach of Donahue et al. (2012) which takes into account the functionalization and fragmentation of organic vapors upon oxidation. The semivolatile POA configuration also includes the option (on by default) of potential secondary organic aerosol from combustion sources (pcSOA). This species is emitted as a VOC (pcVOC) and forms SOA after reaction with OH. The emissions of pcVOC may be zeroed out by the user.

AERO6 uses ISORROPIA in the “reverse mode” to calculate the condensation/evaporation of volatile inorganic gases to/from the gas-phase concentrations of known coarse particle surfaces. It also uses ISORROPIA in the “forward mode” to calculate instantaneous thermodynamic equilibrium between the gas and fine-particle modes. The mass transfer of all semivolatile organic species is calculated assuming equilibrium absorptive partitioning, although some nonvolatile species do exist (e.g. cloud-processed organic aerosol, oligomers, nonvolatile POA (if selected)).

CMAQ can output the reduction in visual range caused by the presence of PM, perceived as haze. CCTM integrates Mie scattering (a generalized particulate light-scattering mechanism that follows from the laws of electromagnetism applied to particulate matter) over the entire range of particle sizes to obtain a single visibility value for each model grid cell at each time step. More detailed descriptions of the PM calculation techniques used in CCTM can be found in Binkowski and Shankar (1995),Binkowski and Roselle (2003), and Byun and Schere (2006).

For easier comparison of CMAQ’s output PM values with measurements, time-dependent cutoff fractions may be output by the model (e.g. Jiang et al., 2006). These include quantities for describing the fraction of each mode that would be categorized as PM2.5 (i.e. PM25AT, PM25AC, and PM25CO) and PM1.0 (i.e. PM1AT, PM1AC, and PM1CO) as well as the fraction of particles from each mode that would be detected by an AMS (i.e AMSAT, AMSAC, and AMSCO). . There is also a surface interaction module in the multipollutant version of CMAQ that calculates the flux of mercury to and from the surface (rather than just depositing mercury).

Further discussion on the scientific improvements to the CMAQ PM treatment is available in the release notes for each version of the model.

## 6.12 Aqueous Chemistry and Scavenging

Clouds are an important component of air quality modeling and play a key role in aqueous chemical reactions, vertical mixing of pollutants, and removal of pollutants by wet deposition. Clouds also indirectly affect pollutant concentrations by altering the solar radiation, which in turn affects photochemical pollutants (such as ozone) and the flux of biogenic emissions. The cloud module in CMAQ performs several functions related to cloud physics and chemistry. Three types of clouds are modeled in CMAQ: sub-grid convective precipitating clouds, sub-grid nonprecipitating clouds, and grid-resolved clouds. The meteorological model provides information about grid-resolved clouds, with no additional cloud dynamics considered in CMAQ. For the two types of sub-grid clouds, the cloud module in CCTM vertically redistributes pollutants, calculates in-cloud and precipitation scavenging, performs aqueous chemistry calculations, and accumulates wet deposition amounts. Aqueous chemistry and scavenging is calculated for resolved clouds as well, using the cell liquid water content and precipitation from the meteorological model.

CMAQ’s standard cloud chemistry treatment (AQCHEM) estimates sulfate production from five sulfur oxidation pathways, as well as includes a simple parameterization to estimate secondary organic aerosol formation from the reactions of glyoxal and methylglyoxal with the hydroxyl radical. The distribution between gas and aqueous phases is determined by instantaneous Henry’s law equilibrium, and the bisection method is used to estimate pH (and the distribution of ionic species) assuming electroneutrality.  Beginning with CMAQv5.1 a new set of options for cloud chemistry was introduced that relies on the Kinetic PreProcessor (KPP), version 2.2.3 (Damian et al., 2002) to generate a Rosenbrock integrator to solve the chemical kinetics, ionic dissociation, wet deposition, and kinetic mass transfer between the gas and aqueous phases in CMAQ clouds.  These options can be collectively referred to as the AQCHEM "KMT” cloud chemistry treatments (Fahey et al., 2017).  

There are several KMT chemistry options currently available in CMAQv5.3.  AQCHEM-KMT treats the standard cloud chemistry mechanism and only differs from AQCHEM with the treatment of kinetic mass transfer between the phases (Schwartz, 1986) and Rosenbrock solver.  AQCHEM-KMTI also includes an expanded aqueous-phase chemical mechanism that treats SOA formation from biogenic-derived epoxides (Pye et al., 2013) in cloud, in addition to the standard sulfur and alpha-dicarbonyl oxidation reactions. With CMAQv5.3 we introduce two additional cloud chemistry options: AQCHEM-KMT2 and AQCHEM-KMTBR.  AQCHEM-KMT2 replaces the simple yield parameterization of SOA from glyoxal and methylglyoxal with a more mechanistic representation of the multi-step formation of oxalic acid/oxalate and other organic acids (assumed here to remain in the aerosol phase after cloud droplet evaporation) from the reactions of hydroxyl radical with glyoxal, methylglyoxal, glycolaldehyde, and acetic acid (Lim et al., 2005; Tan et al., 2009).  AQCHEM-KMT2 expands upon the reactions in AQCHEM-KMTI with additional chemistry for S, N, O-H, and C species (Leriche et al., 2013; Warneck, 1999; Lee and Schwartz, 1983). AQCHEM-KMTBR is the companion aqueous chemistry routine to the gas-phase cb6r3m_ae7_kmtbr mechanism and contains the standard 5 S(IV) oxidation reactions, SOA parameterization from glyoxal and methylglyoxal, as well as additional reactions involving Bromine species in cloud water (Sarwar et al., 2019).  

The AQCHEM KMT family of cloud chemistry options can be significantly more computationally demanding than standard AQCHEM and may be thus better suited for research applications, particularly those investigating cloud/fog events or the evolution of species whose concentrations are potentially heavily influenced by cloud processing and not explicitly represented in the standard AQCHEM mechanism (e.g., oxalate – AQCHEM-KMT2).  Note that when using the gas-phase mechanism with marine chemistry (CB6R3M_AE7_KMTBR), one is required to also run the companion aqueous chemistry routine, AQCHEM-KMTBR.  For limited-area simulations where the primary focus is simulating ozone or total PM2.5 concentrations, especially for longer-term averages, standard AQCHEM would likely capture the most important cloud chemistry impacts (i.e., sulfate formation from the main aqueous oxidation pathways) and is significantly more computationally efficient.

To invoke the default AQCHEM cloud chemistry option, the build script under the CCTM Science Modules section should be set as follows: 

```
set ModCloud  = cloud/acm_ae7
or
set ModCloud  = cloud/acm_ae6
```
For the AQCHEM-KMT cloud chemistry option, use the following option in the build script: 

```
set ModCloud  = cloud/acm_ae6_kmt
```

AQCHEM and AQCHEM-KMT can be used with any of the cb6r3_ae6, cb6r3_ae7, racm, or saprc07 gas phase chemistry mechanisms.

For the AQCHEM-KMTI cloud chemistry option, use the following option in the build script: 
```
set ModCloud  = cloud/acm_ae6i_kmti
```
AQCHEM-KMTI is meant to be used with the saprc07tic_ae6i gas phase chemistry option; i.e., in the build script:

```
set Mechanism = saprc07tic_ae6i_aqkmti
```

For the AQCHEM-KMT2 cloud chemistry option, use the following option in the build script: 
```
set ModCloud  = cloud/acm_ae7_kmt2
```
AQCHEM-KMT2 should only be used in conjunction with the cb6r3_ae7 or saprc07tic_ae7i gas phase chemical mechanisms; i.e., in the build script:

```
set Mechanism = cb6r3_ae7_aqkmt2
```
OR

```
set Mechanism = saprc07tic_ae7i_aqkmt2
```

For the AQCHEM-KMTBR cloud chemistry option, use the following option in the build script: 
```
set ModCloud  = cloud/acm_ae7_kmtbr
```
Note that this cloud chemistry option will be used automatically (and should be used only) when the gas phase chemistry option “cb6r3m_ae7_kmtbr” is chosen.

For toxics/Hg simulations (using the gas phase “cb6mp_ae6_aq” mechanism), one may also invoke the complementary cloud chemistry routine that includes aqueous phase chemistry for some toxic species in addition to the default chemistry:

```
set ModCloud  = cloud/acm_ae6_mp
```

## 6.13 Potential Vorticity Scaling


## References
Colella, P., and P. L. Woodward, 1984: The piecewise parabolic method (PPM) for gas-dynamical simulations. J. Comput. Phys., 54, 174–201.

Cooter, E. J., Bash, J. O., Walker, J. T., Jones, M. R., & Robarge, W. (2010). Estimation of NH3 bi-directional flux from managed agricultural soils. Atmospheric Environment, 44(17), 2107-2115.

Damian, V., A. Sandu, M. Damian, F. Potra, and G.R. Carmichael, 2002: The Kinetic PreProcessor KPP -- A Software Environment for Solving Chemical Kinetics, *Computers and Chemical Engineering*, **26**, 1567-1579.

Fahey, K.M., A.G. Carlton, H.O.T. Pye, J. Baek, W.T. Hutzell, C.O. Stanier, K.R. Baker, K.W. Appel, M. Jaoui, J.H. Offenberg, 2017: A framework for expanding aqueous chemistry in the Community Multiscale Air Quality (CMAQ) model version 5.1, *Geosci. Model Dev.*, **10**, 1587-1605.

Lee, Y.N. and S.E. Schwartz, 1983, Kinetics of oxidation of aqueous sulfur(IV) by nitrogen dioxide. In Precipitation Scavenging, Dry Deposition, and Resuspension, v1 , H.R. Pruppacher et al. (eds.), Elsevier, New York.

Leriche, M., Pinty, J.-P., Mari, C., and D. Gazen, 2013, A cloud chemistry module for the 3-D cloud-resolving mesoscale model Meso-NH with application to idealized cases. Geosci. Mod. Dev., 6, 1275-1298.

Lim, H., Carlton, A. G., and B.J. Turpin, 2005, Isoprene forms secondary
organic aerosol through cloud processing: model simulations. Environ. Sci. Technol., 39, 4441–4446.

Mathur, R. and L.K. Peters, 1990: Adjustment of wind fields for application in air pollution modeling, Atmos. Environ., 24(5), 1095-1106.

Odman, M. T., and A. G. Russell, 2000: Mass conservative coupling of non-hydrostatic meteorological models with air quality models, in Air Pollution Modelling and Its Application XIII, edited by S.-E. Gryning and E. Batchvarova, pp. 651-660, Kluwer Academic/Plenum Publishers, New York.

Pleim J.; Venkatram, A.; Yamartino, R. ADOM/TADAP Model Development Program: The Dry
Deposition Module; Ontario Ministry of the Environment: Rexdale, Canada, 1984; Volume 4.

Pleim, J. E. (2007a). A combined local and nonlocal closure model for the atmospheric boundary layer. Part I: Model description and testing. Journal of Applied Meteorology and Climatology, 46(9), 1383-1395.

Pleim, J. E. (2007b). A combined local and nonlocal closure model for the atmospheric boundary layer. Part II: Application and evaluation in a mesoscale meteorological model. Journal of Applied Meteorology and Climatology, 46(9), 1396-1409.

Pleim, J., & Ran, L. (2011). Surface flux modeling for air quality applications. Atmosphere, 2(3), 271-302.

Pleim, J. E., L. Ran, W. Appel, M. W. Shephard, and K. Cady-Pereira, 2019,  New bidirectional ammonia flux model in an air quality model coupled with an agricultural model.  JAMES in review.

Pye, H.O.T., R.W. Pinder, I.R. Piletic, Y. Xie, S.L. Capps, Y.H. Lin, J.D. Surratt, Z.F. Zhang, A. Gold, D.J. Luecken, W.T. Hutzell, M. Jaoui, J.H. Offenberg, T.E. Kleindienst, M. Lewandowski, E.O. Edney, 2013: Epoxide pathways improve model predictions of isoprene markers and reveal key role of acidity in aerosol formation, *Environ. Sci. Technol.*, **47(19)**, 11056-11064.\

Ran, L., Cooter, E., Benson, V., & He, Q. (2011). Development of an Agricultural Fertilizer Modeling System for Bi-directional Ammonia Fluxes in the CMAQ Model. In D. G. Steyn, & S. Trini Castelli (Eds.), Air Pollution Modeling and its Application XXI (Chapter 36, pp. 213-219). Springer, Dordrecht. 

Sarwar, G., Gantt, B., Foley, K., Fahey, K., Spero, T.L., Kang, D., Mathur, R., Foroutan, H., Xing, J., Sherwen, T, and A. Saiz-Lopez, 2019, Influence of bromine and iodine chemistry on annual, seasonal, diurnal, and background ozone: CMAQ simulations over the Northern Hemisphere.  Atmos. Env., Accepted.

Schwartz, S.E., 1986: Mass transport considerations pertinent to aqueous-phase reactions of gases in liquid water clouds. In Chemistry of multiphase atmospheric systems, NATO ASI Series, *G6*, 415-471.

Skamarock, W.C., Klemp, J.B., Dudhia, J., Gill, D.O., Liu, Z., Berner, J., Wang, W., Powers, J.G., Duda, M.G., Barker, D.M., Huang, X.Y., A description of the advanced research WRF version 4. NCAR TECHNICAL NOTE, NCAR/TN–556+STR, March, 2019.

Tan, Y., Perri, M.J., Seitzinger, S.P., and B.J. Turpin, 2009, Effects of Precursor Concentration and Acidic Sulfate in Aqueous Glyoxal-OH Radical Oxidation and Implications for Secondary Organic Aerosol. Env. Sci. Technol., 43, 8105–8112.

Warneck, P., 1999, The relative importance of various pathways for the oxidation of sulfur dioxide and nitrogen dioxide in sunlit continental fair weather clouds.  Phys. Chem. Chem. Phys., 1, 5471-5483.


<!-- BEGIN COMMENT -->

[<< Previous Chapter](CMAQ_UG_ch05_compile_and_run.md) - [Home](README.md) - [Next Chapter >>](CMAQ_UG_ch07_model_outputs.md)
<br>
CMAQ User's Guide (c) 2019<br>

<!-- END COMMENT -->
