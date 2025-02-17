# How to Develop a new workflow for `atomate2`

## Anatomy of an `atomate2` computational workflow (i.e., what do I need to write?)

Every `atomate2` workflow is an instance of jobflow's `Flow ` class, which is a collection of Job and/or other `Flow` objects. So your end goal is to produce a `Flow `.

In the context of computational materials science, `Flow ` objects are most easily created by a `Maker`, which contains a factory method make() that produces a `Flow `, given certain inputs. Typically, the input to `Maker`.make() includes atomic coordinate information in the form of a `pymatgen` `Structure` or `Molecule` object. So the basic signature looks like this:

```python
class ExampleMaker(Maker):
    def make(self, coordinates: Structure) -> Flow:
        # take the input coordinates and return a `Flow `
        return Flow(...)
```

The `Maker` class usually contains most of the calculation parameters and other settings that are required to set up the calculation in the correct way. Much of this logic can be written like normal python functions and then turned into a `Job` via the `@job` decorator.

One common task encountered in almost any materials science calculation is writing calculation input files to disk so they can be executed by the underlying software (e.g., VASP, Q-Chem, CP2K, etc.). This is preferably done via a `pymatgen` `InputSet` class. `InputSet` is essentially a dict-like container that specifies the files that need to be written, and their contents. Similarly to the way that `Maker` classes generate `Flow`s, `InputSet`s are most easily created by `InputGenerator` classes. `InputGenerator`
have a method `get_input_set()` that typically takes atomic coordinates (e.g., a `Structure` or `Molecule` object) and produce an `InputSet`, e.g.,

```python
class ExampleInputGenerator(InputGenerator):
    def get_input_set(self, coordinates: Structure) -> InputSet:
        # take the input coordinates, determine appropriate
        # input file contents, and return an `InputSet`
        return InputSet(...)
```

`pymatgen` already contains `InputSet` for many common codes, so when developing a workflow `Maker` it is convenient to use the `InputGenerator` / `InputSet` to prepare your files. This is done in `atomate2` by making the `InputGenerator` a class parameter, e.g.,

**TODO - the code block below needs refinement. Not exactly sure how write_inputs() fits into a`Job`**

```python
class ExampleMaker(Maker):
    input_set_generator: ExampleInputGenerator = field(
        default_factory=ExampleInputGenerator
    )

    def make(self, coordinates: Structure) -> Flow:
        # create an`InputSet`
        input_set = self.input_set_generator.get_input_set(coordinates)
        # write the input files
        input_set.write_inputs()
        return Flow(...)
```

Finally, most `atomate2` workflows return structured output in the form of "Task Documents". Task documents are instances of `emmet`'s `BaseTaskDocument` class (similarly to a `python` `@dataclass`) that define schemas for storing calculation outputs. `emmet` already contains calculation schemas for codes utilized by the Materials Project (e.g., VASP, Q-Chem, FEFF) as well as a number of schemas for code-agnostic structural and molecular information (for example, the `MaterialsDoc` is a schema for solid material calculation data). `atomate2` can also interpret output generated by [`cclib`](https://cclib.github.io/), which is able to parse the output of many additional codes.

**TODO - extend code block above to illustrate TaskDoc usage**

In summary, a new `atomate2` workflow consists of the following components:
 - A `Maker` that actually generates the workflow
 - One or more `Job` and/or `Flow ` classes that define the discrete steps in the workflow
 - (optionally) an `InputGenerator` that produces a `pymatgen` `InputSet` for writing calculation input files
 - (optionally) a `TaskDocument` that defines a schema for storing the output data

## Where do I put my code?

Because of the distributed design of the MP Software Ecosystem, writing a complete new workflow may involve making contributions to more than one GitHub repository. The following guidelines should help you understand where to put your contribution.

 - All workflow code (`Job`, `Flow `, `Maker`) belongs in `atomate2`
 - `InputSet` and `InputGenerator` code belongs in `pymatgen`. However, if you need to create these classes from scratch (i.e., you are working with a code that is not already supported in`pymatgen`), then it is recommended to include them in `atomate2` at first to facilitate rapid iteration. Once mature, they can be moved to `pymatgen` or to a `pymatgen` [addon package](https://pymatgen.org/addons).
 - `TaskDocument` schemas should generally be developed in `atomate2` alongside the workflow code. We recommend that you first check emmet to see if there is an existing schema that matches what you need. If so, you can import it. If not, check [`cclib`](https://cclib.github.io/). `cclib` output can be imported via [`atomate2.common.schemas.TaskDocument`](https://github.com/materialsproject/atomate2/blob/main/src/atomate2/common/schemas/cclib.py). If neither code has what you need, then new schemas should be developed within `atomate2` (or `cclib`).
