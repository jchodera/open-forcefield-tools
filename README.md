# Open Forcefield Tools

Tools for open forcefield development.

## API documentation

### Physical property measurements

[**Physical property measurements**](https://en.wikipedia.org/wiki/Physical_property) are measured properties of a substance that provide some information about the physical parameters that define the interactions within the substance.
A physical property is defined by a combination of:
* A `Mixture` specifying the substance that the measurement was performed on
* A `ThermodynamicState` specifying the thermodynamic conditions under which the measurement was performed
* A `PhysicalProperty` is the physical property that was measured
* A `MeasurementMethod` specifying the kind of measurement that was performed

An example of each:
* `Mixture`: 0.8 mole fraction mixture of ethanol and water
* `ThermodynamicState`: 298 kelvin, 1 atmosphere
* `PhysicalProperty`: mass density
* `MeasurementMethod`: vibrating tube method

#### Physical substances

We use the concept of a liquid or gas `Mixture` throughout.

A simple liquid has only one component:
```python
liquid = Mixture()
liquid.addComponent('water')
```

A binary mixture has two components:
```python
binary_mixture = Mixture()
binary_mixture.addComponent('water', mole_fraction=0.2)
binary_mixture.addComponent('methanol') # assumed to be rest of mixture if no mole_fraction specified
```

A ternary mixture has three components:
```python
ternary_mixture = Mixture()
binary_mixture.addComponent('ethanol', mole_fraction=0.2)
binary_mixture.addComponent('methanol', mole_fraction=0.2)
ternary_mixture.addComponent('water')
```

The infinite dilution of one solute within a solvent or mixture is also specified as a `Mixture`, where the solute has zero mole fraction:
```python
infinite_dilution = Mixture()
infinite_dilution.addComponent('phenol', mole_fraction=0.0) # infinite dilution
infinite_dilution.addComponent('water')
```

**TODO**:
* Should we instead allow the user to specify one molecule of the solute, as in
```python
infinite_dilution = Mixture()
infinite_dilution.addComponent('phenol', number=1) # one molecule
infinite_dilution.addComponent('water')
```

**QUESTIONS**:
* Is the concept of `Mixture` sufficiently general that we don't need further types of substances?
* Is it OK that we don't specify the total number of molecules in the substance, and only specify the fractional composition? We would have to specify the total number of molecules in the `PropertyCalculator` instead.

#### Thermodynamic states

A `ThermodynamicState` specifies a combination of thermodynamic parameters (e.g. temperature, pressure) at which a measurement is performed.
```python
from simtk import unit
thermodynamic_state = ThermodynamicState(pressure=500*unit.kilopascals, temperature=298.15*unit.kelvin)
```
We use the `simtk.unit` unit system from [OpenMM](http://openmm.org) for units.

**QUESTIONS:**
* Is it OK to use `simtk.unit` from OpenMM for now, or should we switch to [`pint`](https://pint.readthedocs.io/en/0.7.2/) to make this more portable?

#### Measurement methods

A `MeasurementMethod` subclass has information specific to the particular method used to measure a property (such as experimental uncertainty guidance).

Some examples:
* `FlowCalorimetry` for `HeatCapacity` or `ExcessMolarEnthalpy`
* `VibratingTubeMethod` for `MassDensity`

#### Physical property measurements

A `MeasuredPhysicalProperty` is a combination of `Mixture`, `ThermodynamicState`, and a unit-bearing measured property `value` and `uncertainty`:
```python
# Define mixture
mixture = Mixture()
mixture.addComponent('water', mole_fraction=0.2)
mixture.addComponent('methanol')
# Define thermodynamic state
thermodynamic_state = ThermodynamicState(pressure=500*unit.kilopascals, temperature=298.15*unit.kelvin)
# Define measurement
measurement = ExcessMolarEnthalpy(substance, thermodynamic_state, value=83.3863244*unit.kilojoules_per_mole, uncertainty=0.1220794866*unit.kilojoules_per_mole)
```
The various properties are all subclasses of `MeasuredPhysicalProperty` and generally follow the `<ePropName/>` ThermoML tag names.
Some examples:
* `MassDensity` - mass density
* `ExcessMolarEnthalpy` - excess partial apparent molar enthalpy
* `HeatCapacity` - molar heat capacity at constant pressure

A [roadmap of physical properties to be implemented](https://github.com/open-forcefield-group/open-forcefield-tools/wiki/Physical-Properties-for-Calculation) is available.
Please raise an issue if your physical property of interest is not listed!

Each `MeasuredPhysicalProperty` has several properties:
* `.substance` - the `Mixture` for which the measurement was made
* `.thermodynamic_state` - the `ThermodynamicState`
* `.value` - the unit-bearing measurement value
* `.uncertainty` - the standard uncertainty of the measurement
* `.reference` - the literature reference (if present) for the measurement
* `.DOI` - the literature reference DOI (if available) for the measurement

The value, uncertainty, reference, and DOI do not necessarily need to be defined for a dataset in order for property calculations to be performed.

### Physical property datasets

A `PhysicalPropertyDataset` is a collection of `MeasuredPhysicalProperty` objects that are related in some way.
```python
dataset = PhysicalPropertyDataset([measurement1, measurement2])
```
The dataset is iterable:
```python
dataset = PhysicalPropertyDataset([measurement1, measurement2])
for measurement in dataset:
    print measurement.value
```
and has accessors to retrieve DOIs and references associated with measurements in the dataset:
```python
# Print the DOIs associated with this dataset
print(dataset.DOIs)
# Print the references associated with this dataset
print(dataset.references)
```

### ThermoML datasets

Direct access to the [NIST ThermoML Archive](http://trc.nist.gov/ThermoML.html) is supported for defining physical property measurements in the IUPAC-standard [ThermoML](http://trc.nist.gov/ThermoMLRecommendations.pdf) format, a format for specifying thermodynamic properties in XML format.

For example, to retrieve [this ThermoML dataset](http://trc.boulder.nist.gov/ThermoML/10.1016/j.jct.2005.03.012) that accompanies [this paper](http://www.sciencedirect.com/science/article/pii/S0021961405000741), we can simply use the DOI `10.1016/j.jct.2005.03.012` as a key for creating a `PhysicalPropertyDataset` subclassed object from the ThermoML Archive:
```python
dataset = ThermoMLDataset('10.1016/j.jct.2005.03.012')
```
You can also specify multiple ThermoML Archive keys to create a dataset from multiple ThermoML files:
```python
thermoml_keys = ['10.1021/acs.jced.5b00365', '10.1021/acs.jced.5b00474']
dataset = ThermoMLDataset(thermoml_keys)
```
You can see which DOIs contribute to the current `ThermoMLDataset` with the convenience functions:
```python
thermoml_keys = ['10.1021/acs.jced.5b00365', '10.1021/acs.jced.5b00474']
dataset = ThermoMLDataset(thermoml_keys)
```

### Estimating properties

The `PropertyEstimator` class creates objects that handle property estimation of all of the properties in a dataset, given a set or sets of parameters.
The implementation will isolate the user from whatever backend (local machine, HPC cluster, XSEDE resources, Amazon EC2) is being used to compute the properties, as well as whether new simulations are being launched and analyzed or existing simulation data is being reweighted.
Different backends will take different optional arguments, but here is an example that will launch and use 10 worker processes on a cluster:
```python
estimator = PropertyEstimator(nworkers=10) # NOTE: multiple backends will be supported in the future
computed_properties = estimator.computeProperties(dataset, parameter_sets)
```
Here, `dataset` is a `PhysicalPropertyDataset` or subclass, and `parameter_sets` is a list containing `SMIRFFParameterSet` objects used to parameterize the physical systems in the dataset. This can be a single parameter set or multiple (usually related) parameter sets.

`PropertyEstimator.computeProperties(...)` returns a list of `ComputedPhysicalProperty` objects that provide access to several pieces of information:
* `property.value` - the computed property value, with appropriate units
* `property.uncertainty` - the statistical uncertainty in the computed property
* `property.parameters` - a reference to the parameter set used to compute this property
* `property.property` - a reference to the corresponding `MeasuredPhysicalProperty` this property was computed for

This API can be extended in the future to provide access to the simulation data used to estimate the property, such as
```python
# Attach to my compute and storage resources
estimator = PropertyEstimator(...)
# Estimate some properties
computed_properties = estimator.computeProperties(dataset, parameters)
# Get statistics about simulation data that went into each property
for property in computed_properties:
   # Get statistics about simulation data that was reweighted to estimate this property
   for simulation in property.simulations:
      print('The simulation was %.3f ns long' % (simulation.length / unit.nanoseconds))
      print('The simulation was run at %.1f K and %.1f atm' % (simulation.thermodynamic_state.temperature / unit.kelvin, simulation.thermodynamic_state.pressure / unit.atmospheres))
      # Get the ParameterSet that was used for this simulation
      parameters = simulation.parameters
      # what else do you want...?
```

In future, we will want to use a parallel key/value database like [cassandra](http://cassandra.apache.org) to store simulations, along with a distributed task management system like [celery](http://www.celeryproject.org) with [redis](https://www.google.com/search?client=safari&rls=en&q=redis&ie=UTF-8&oe=UTF-8).

## API Usage Examples

### Using the high-level API

In this example, datasets are retrieved from the ThermoML and filtered to retain certain properties.
The corresponding properties for a given parameter set filename are then computed for a SMIRFF parameter set and printed.
```python
# Define the input datasets from ThermoML
thermoml_keys = ['10.1016/j.jct.2005.03.012', ...]
dataset = ThermoMLDataset(thermoml_keys)
# Filter the dataset to include only molar heat capacities measured between 280-350 K
dataset.filter(ePropName='Excess molar enthalpy (molar enthalpy of mixing), kJ/mol') # filter to retain only this property name
dataset.filter(VariableType='eTemperature', min=280*unit.kelvin, max=350*kelvin) # retain only measurements with `eTemperature` in specified range
# Load an initial parameter set
parameter_set = [ SMIRFFParameterSet('smarty-initial.xml') ]
# Compute physical properties for these measurements
estimator = PropertyEstimator(nworkers=10) # NOTE: multiple backends will be supported in the future
computed_properties = estimator.computeProperties(dataset, parameter_set)
# Write out statistics about errors in computed properties
for (computed, measured) in (computed_properties, dataset):
   property_unit = measured.value.unit
   print('%24s : experiment %8.3f (%.3f) | calculated %8.3f (%.3f) %s' % (measured.value / property_unit, measured.uncertainty / property_unit, computed.value / property_unit, computed.uncertainty / property_unit, str(property_unit))
```

### Using the low-level API

TBD
