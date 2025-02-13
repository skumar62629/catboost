# Traitlets

[![Tests](https://github.com/ipython/traitlets/actions/workflows/tests.yml/badge.svg)](https://github.com/ipython/traitlets/actions/workflows/tests.yml)
[![Documentation Status](https://readthedocs.org/projects/traitlets/badge/?version=latest)](https://traitlets.readthedocs.io/en/latest/?badge=latest)
[![codecov](https://codecov.io/gh/ipython/traitlets/branch/main/graph/badge.svg?token=HcsbLGEmI1)](https://codecov.io/gh/ipython/traitlets)
[![Tidelift](https://tidelift.com/subscription/pkg/pypi-traitlets)](https://tidelift.com/badges/package/pypi/traitlets)

|               |                                      |
| ------------- | ------------------------------------ |
| **home**      | https://github.com/ipython/traitlets |
| **pypi-repo** | https://pypi.org/project/traitlets/  |
| **docs**      | https://traitlets.readthedocs.io/    |
| **license**   | Modified BSD License                 |

Traitlets is a pure Python library enabling:

- the enforcement of strong typing for attributes of Python objects
  (typed attributes are called _"traits"_);
- dynamically calculated default values;
- automatic validation and coercion of trait attributes when attempting a
  change;
- registering for receiving notifications when trait values change;
- reading configuring values from files or from command line
  arguments - a distinct layer on top of traitlets, so you may use
  traitlets without the configuration machinery.

Its implementation relies on the [descriptor](https://docs.python.org/howto/descriptor.html)
pattern, and it is a lightweight pure-python alternative of the
[_traits_ library](https://docs.enthought.com/traits/).

Traitlets powers the configuration system of IPython and Jupyter
and the declarative API of IPython interactive widgets.

## Installation

For a local installation, make sure you have
[pip installed](https://pip.pypa.io/en/stable/installing/) and run:

```bash
pip install traitlets
```

For a **development installation**, clone this repository, change into the
`traitlets` root directory, and run pip:

```bash
git clone https://github.com/ipython/traitlets.git
cd traitlets
pip install -e .
```

## Running the tests

```bash
pip install "traitlets[test]"
py.test traitlets
```

## Code Styling

`traitlets` has adopted automatic code formatting so you shouldn't
need to worry too much about your code style.
As long as your code is valid,
the pre-commit hook should take care of how it should look.

To install `pre-commit` locally, run the following::

```
pip install pre-commit
pre-commit install
```

You can invoke the pre-commit hook by hand at any time with::

```
pre-commit run
```

which should run any autoformatting on your code
and tell you about any errors it couldn't fix automatically.
You may also install [black integration](https://github.com/psf/black#editor-integration)
into your text editor to format code automatically.

If you have already committed files before setting up the pre-commit
hook with `pre-commit install`, you can fix everything up using
`pre-commit run --all-files`. You need to make the fixing commit
yourself after that.

Some of the hooks only run on CI by default, but you can invoke them by
running with the `--hook-stage manual` argument.

## Usage

Any class with trait attributes must inherit from `HasTraits`.
For the list of available trait types and their properties, see the
[Trait Types](https://traitlets.readthedocs.io/en/latest/trait_types.html)
section of the documentation.

### Dynamic default values

To calculate a default value dynamically, decorate a method of your class with
`@default({traitname})`. This method will be called on the instance, and
should return the default value. In this example, the `_username_default`
method is decorated with `@default('username')`:

```Python
import getpass
from traitlets import HasTraits, Unicode, default

class Identity(HasTraits):
    username = Unicode()

    @default('username')
    def _username_default(self):
        return getpass.getuser()
```

### Callbacks when a trait attribute changes

When a trait changes, an application can follow this trait change with
additional actions.

To do something when a trait attribute is changed, decorate a method with
[`traitlets.observe()`](https://traitlets.readthedocs.io/en/latest/api.html?highlight=observe#traitlets.observe).
The method will be called with a single argument, a dictionary which contains
an owner, new value, old value, name of the changed trait, and the event type.

In this example, the `_num_changed` method is decorated with `` @observe(`num`) ``:

```Python
from traitlets import HasTraits, Integer, observe

class TraitletsExample(HasTraits):
    num = Integer(5, help="a number").tag(config=True)

    @observe('num')
    def _num_changed(self, change):
        print("{name} changed from {old} to {new}".format(**change))
```

and is passed the following dictionary when called:

```Python
{
  'owner': object,  # The HasTraits instance
  'new': 6,         # The new value
  'old': 5,         # The old value
  'name': "foo",    # The name of the changed trait
  'type': 'change', # The event type of the notification, usually 'change'
}
```

### Validation and coercion

Each trait type (`Int`, `Unicode`, `Dict` etc.) may have its own validation or
coercion logic. In addition, we can register custom cross-validators
that may depend on the state of other attributes. For example:

```Python
from traitlets import HasTraits, TraitError, Int, Bool, validate

class Parity(HasTraits):
    value = Int()
    parity = Int()

    @validate('value')
    def _valid_value(self, proposal):
        if proposal['value'] % 2 != self.parity:
            raise TraitError('value and parity should be consistent')
        return proposal['value']

    @validate('parity')
    def _valid_parity(self, proposal):
        parity = proposal['value']
        if parity not in [0, 1]:
            raise TraitError('parity should be 0 or 1')
        if self.value % 2 != parity:
            raise TraitError('value and parity should be consistent')
        return proposal['value']

parity_check = Parity(value=2)

# Changing required parity and value together while holding cross validation
with parity_check.hold_trait_notifications():
    parity_check.value = 1
    parity_check.parity = 1
```

However, we **recommend** that custom cross-validators don't modify the state
of the HasTraits instance.
