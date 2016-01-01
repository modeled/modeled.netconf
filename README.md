

# MODELED.netconf



> [github.com/modeled/modeled.netconf](https://github.com/modeled/modeled.netconf)



[![](http://www.gnu.org/graphics/lgplv3-88x31.png)](
  https://gnu.org/licenses/lgpl.html)
[![](https://img.shields.io/pypi/pyversions/modeled.netconf.svg)](
  https://python.org)
[![](https://img.shields.io/pypi/v/modeled.netconf.svg)](
  https://pypi.python.org/pypi/modeled.netconf)
[![](https://img.shields.io/pypi/dd/modeled.netconf.svg)](
  https://pypi.python.org/pypi/modeled.netconf)



[![](https://travis-ci.org/modeled/modeled.netconf.svg?branch=master)](
  https://travis-ci.org/modeled/modeled.netconf)
[![](https://ci.appveyor.com/api/projects/status/nqymmsa76qo90gdi?svg=true)](
  https://ci.appveyor.com/project/userzimmermann/python-modeled-netconf)



## Highly Pythonized NETCONF and YANG



[\\/\\/\\>](#from-a-modeled-class-to-a-yang-module)
  Automagically create [YANG](http://www.yang-central.org)
  modules and data containers from
  [MODELED](https://pypi.python.org/pypi/modeled) Python classes

[\\/\\/\\>](#adding-rpc-methods)
  Simply turn Python methods into
  [NETCONF](http://www.netconfcentral.org)/YANG RPC methods
  using decorators

[\\/\\/\\>](#from-modeled-yang-modules-to-a-netconf-service)
  Directly start NETCONF servers from modeled YANG modules

**TODO**:

* Proper RPC namespace handling
* Handle default NETCONF RPC methods like `<get>` or `<get-config>`
* Create NETCONF servers with multiple modeled YANG modules
* Automagically create Pythonic NETCONF clients from YANG definitions



**WARNING**: This project is in early alpha state
and therefore not production ready.



**ABOUT** this README:

* It contains [setup instructions](#setup)
  and a [tutorial](#from-a-modeled-class-to-a-yang-module).
* It was automatically created from
  [IPython](http://ipython.org) notebook **README.ipynb**.
  You can [view the notebook](
    http://nbviewer.ipython.org/github/modeled/modeled.netconf/blob/master/README.ipynb)
  online.
* The internal links don't work on Bitbucket.



## Setup



Just use [pip](http://www.pip-installer.org/)
to install the latest release from [PyPI](https://pypi.python.org):

> `pip install modeled.netconf`

It automatically installs all runtime requirements:




```python
>>> import modeled.netconf
>>> modeled.netconf.__requires__
zetup>=0.2.27
backports.socketpair>=3.5.0
moretools>=0.1.7
modeled>=0.1.8
netconf>=0.3.1
pyang>=1.6
```



To install in development mode:

> `pip install -e /path/to/repository/`



## From a MODELED class to a YANG module



**MODELED.netconf** is based on my
[MODELED](https://pypi.python.org/pypi/modeled) framework,
which provides tools for defining Python classes
with typed members and methods,
quite similar to [Django](https://djangoproject.com) database models,
but with a more general approach.
Those modeled classes can then automagically be mapped
to data serialization formats, databases,
GUI frameworks, web frameworks, or whatever,
using the integrated `modeled.Adapter` system.
The **MODELED** framework is still in a late alpha stage,
needs some internal refactoring, and lacks documentation,
but I am actively working on this.
The basic principles should nevertheless become visible during the following example.



### A MODELED Turing Machine



Since **MODELED.netconf** uses [pyang](https://pypi.python.org/pypi/pyang)
for auto-generating YANG definitions from modeled classes,
I decided to resemble the Turing Machine example from the
**pyang** [tutorial](https://github.com/mbj4668/pyang/wiki/Tutorial)...
a bit more simplified and with some little structural and naming changes...
however... below is a modeled Turing Machine implementation:




```python
import modeled
from modeled import member
```



```python
class Input(modeled.object):
    """The input part of a Turing Machine program rule.
    """
    state = member[int]()
    symbol = member[str]()
```



```python
class Output(modeled.object):
    """The output part of a Turing Machine program rule.
    """
    state = member[int]()
    symbol = member[str]()
    head_move = member[str]['L', 'R']()
```



```python
class Rule(modeled.object):
    """A Turing Machine program rule.
    """
    input = member[Input]()
    output = member[Output]()

    def __init__(self, input, output):
        """Expects both `input` and `output` as mappings.
        """
        self.input = Input(
            # modeled.object.__init__ supports **kwargs
            # for initializing modeled.member values
            **dict(input))
        self.output = Output(**dict(output))
```



```python
class TuringMachine(modeled.object):
    state = member[int]()
    head_position = member[int]()

    # the list of symbols on the input/output tape
    tape = member.list[str](indexname='cell', itemname='symbol')

    # the machine program as named rules
    program = member.dict[str, Rule](keyname='name')

    def __init__(self, program):
        """Create a Turing Machine with the given `program`.
        """
        program = dict(program)
        for name, (input, output) in program.items():
            self.program[name] = Rule(input, output)

    def run(self):
        """Start the Turing Machine.
        
        - Runs until no matching input part for current state and tape symbol
          can be found in the program rules.
        """
        self.log = " %s  %d\n" % (''.join(self.tape), self.state)
        while True:
            pos = self.head_position
            if 0 <= pos < len(self.tape):
                symbol = self.tape[pos]
            else:
                symbol = None
            for name, rule in self.program.items():
                if (self.state, symbol) == (rule.input.state, rule.input.symbol):
                    self.log += "%s^%s --> %s\n" % (
                        ' ' * (pos + 1),
                        ' ' * (len(self.tape) - pos),
                        name)
                    if rule.output.state is not None:
                        self.state = rule.output.state
                    if rule.output.symbol is not None:
                        self.tape[pos] = rule.output.symbol
                    self.head_position += {'L': -1, 'R': 1}[rule.output.head_move]
                    self.log += " %s  %d\n" % (''.join(self.tape), self.state)
                    break
            else:
                break
```


To check if the Turing Machine works, it needs an actual program.
I took it from the **pyang** tutorial again.
It's a very simple program for adding to numbers in unary notation,
separated by a **0**.

The program is defined in [YAML](http://yaml.org) below.
If you haven't installed [pyyaml](http://pyyaml.org/wiki/PyYAML) yet:

> `pip install pyyaml`

(`%%...` are IPython magic functions):




```yaml
%%file turing-machine-program.yaml

left summand:
  - {state:    0, symbol:    1}
  - {state: null, symbol: null, head_move: R}
separator:
  - {state:    0, symbol:    0}
  - {state:    1, symbol:    1, head_move: R}
right summand:
  - {state:    1, symbol:    1}
  - {state: null, symbol: null, head_move: R}
right end:
  - {state:    1, symbol: null}
  - {state:    2, symbol: null, head_move: L}
write separator:
  - {state:    2, symbol:    1}
  - {state:    3, symbol:    0, head_move: L}
go home:
  - {state:    3, symbol:    1}
  - {state: null, symbol: null, head_move: L}
final step:
  - {state:    3, symbol: null}
  - {state:    4, symbol: null, head_move: R}
```

```
Writing turing-machine-program.yaml
```


```python
import yaml
with open('turing-machine-program.yaml') as f:
    TM_PROGRAM = yaml.load(f)
```


Instantiate the Turing Machine with the loaded program:




```python
tm = TuringMachine(TM_PROGRAM)
```


And set the initial state for computing unary **1 + 2**:




```python
tm.state = 0
tm.head_position = 0
tm.tape = '1011'
```


The tape string gets automatically converted to a list,
because `TuringMachine.tape` is defined as a `list` member:




```python
>>> tm.tape
['1', '0', '1', '1']
```



Ready for turning on the Turing Machine:




```python
tm.run()
```



```python
>>> print(tm.log)
```

```
 1011  0
 ^     --> left summand
 1011  0
  ^    --> separator
 1111  1
   ^   --> right summand
 1111  1
    ^  --> right summand
 1111  1
     ^ --> right end
 1111  2
    ^  --> write separator
 1110  3
   ^   --> go home
 1110  3
  ^    --> go home
 1110  3
 ^     --> go home
 1110  3
^      --> final step
 1110  4
```

Final state is reached. Result is unary **3**. Seems to work!



### YANGifying the Turing Machine



Creating a YANG module from the modeled `TuringMachine` class
is now quite simple. Just import the modeled `YANG` module adapter class:




```python
from modeled.netconf import YANG
```


And plug it to the `TuringMachine`.
This will create a new class which will be derived
from both the `YANG` module adapter and the `TuringMachine` class:




```python
>>> YANG[TuringMachine].mro()
[modeled.netconf.yang.YANG[TuringMachine],
 modeled.netconf.yang.YANG,
 modeled.netconf.yang.container.YANGContainer,
 modeled.Adapter,
 __main__.TuringMachine,
 modeled.object,
 modeled.base.base,
 zetup.object.object,
 object]
```



It also has a class attribute referencing the original modeled class:




```python
>>> YANG[TuringMachine].mclass
__main__.TuringMachine
```



BTW: the class adaption will be cached,
so every `YANG[TuringMachine]` operation
will return the same class object:




```python
>>> YANG[TuringMachine] is YANG[TuringMachine]
True
```



But let's take look at the really useful features now.
The adapted class dynamically provides `.to_...()` methods
for every **pyang** output format plugin
which you could pass to the **pyang** command's **-f** flag.
Calling such a method will programmatically
create a `pyang.statement.Statement` tree
(which **pyang** does internally on loading an input file)
according to the typed members of the adapted modeled class.

Every `.to_...()` method takes optional
`revision` date and XML `prefix` and `namespace` arguments.
If no `revision` is given,
the current date will be used.

The adapted class will be mapped to a YANG module
and its main data container definition.
Module and container name will be generated from the name
of the adapted modeled class
by decapitalizing and joining its name parts with hyphens.
YANG leaf names will be generated from modeled member names
by replacing underscores with hyphens.
`list` and `dict` members will be mapped to YANG list definitions.
If members have other modeled classes as types,
sub-containers will be defined.

Type mapping is very simple in this early project stage.
Only `int` and `str` are supported
and no YANG typedefs are used.
All containers and their contents are defined configurable
(with write permissions).
That will change soon...

The result is a complete module definition text in the given format,
like default YANG:




```python
>>> print(YANG[TuringMachine].to_yang(
>>>     prefix='tm', namespace='http://modeled.netconf/turing-machine'))
```

```
module turing-machine {
  namespace "http://modeled.netconf/turing-machine";
  prefix tm;

  revision 2015-10-29;

  container turing-machine {
    leaf state {
      type int64;
    }
    leaf head-position {
      type int64;
    }
    list tape {
      key "cell";
      leaf cell {
        type int64;
      }
      leaf symbol {
        type string;
      }
    }
    list program {
      key "name";
      leaf name {
        type string;
      }
      container rule {
        container input {
          leaf state {
            type int64;
          }
          leaf symbol {
            type string;
          }
        }
        container output {
          leaf state {
            type int64;
          }
          leaf symbol {
            type string;
          }
          leaf head-move {
            type string;
          }
        }
      }
    }
  }
}
```

Or XMLified YIN:




```python
>>> print(YANG[TuringMachine].to_yin(
>>>     prefix='tm', namespace='http://modeled.netconf/turing-machine'))
```

```
<?xml version="1.0" encoding="UTF-8"?>
<module name="turing-machine"
        xmlns="urn:ietf:params:xml:ns:yang:yin:1"
        xmlns:tm="http://modeled.netconf/turing-machine">
  <namespace uri="http://modeled.netconf/turing-machine"/>
  <prefix value="tm"/>
  <revision date="2015-10-29"/>
  <container name="turing-machine">
    <leaf name="state">
      <type name="int64"/>
    </leaf>
    <leaf name="head-position">
      <type name="int64"/>
    </leaf>
    <list name="tape">
      <key value="cell"/>
      <leaf name="cell">
        <type name="int64"/>
      </leaf>
      <leaf name="symbol">
        <type name="string"/>
      </leaf>
    </list>
    <list name="program">
      <key value="name"/>
      <leaf name="name">
        <type name="string"/>
      </leaf>
      <container name="rule">
        <container name="input">
          <leaf name="state">
            <type name="int64"/>
          </leaf>
          <leaf name="symbol">
            <type name="string"/>
          </leaf>
        </container>
        <container name="output">
          <leaf name="state">
            <type name="int64"/>
          </leaf>
          <leaf name="symbol">
            <type name="string"/>
          </leaf>
          <leaf name="head-move">
            <type name="string"/>
          </leaf>
        </container>
      </container>
    </list>
  </container>
</module>
```

Since the modeled YANG module
is derived from the adapted `TuringMachine` class,
it can still be instantiated and used in the same way:




```python
tm = YANG[TuringMachine](TM_PROGRAM)
```



```python
tm.state = 0
tm.head_position = 0
tm.tape = '1011'
```



```python
tm.run()
```



```python
>>> tm.state, tm.tape
(4, ['1', '1', '1', '0'])
```



### Adding RPC methods



The above modeled YANG module is not very useful
without some RPC methods for controlling the Turing Machine via NETCONF.
**MODELED.netconf** offers a simple `@rpc` decorator
for defining them:




```python
from modeled.netconf import rpc
```


The following RPC definitions are again designed
according to the **pyang** tutorial.

Since those RPC methods are NETCONF/YANG specific,
they are defined after the modeled YANG adaption.
The simplest way is to derive a new class for that purpose:




```python
class TM(YANG[TuringMachine]):

    @rpc(argtypes={'tape_content': str})
    # in Python 3 you can also use function annotations
    # and write (self, tape_content: str) below
    # instead of argtypes= above
    def initialize(self, tape_content):
        """Initialize the Turing Machine.
        """
        self.state = 0
        self.head_position = 0
        self.tape = tape_content

    @rpc(argtypes={})
    def run(self):
        """Start the Turing Machine operation.
        """
        TuringMachine.run(self)
```


Now the `.to_yang()` conversion also includes the **rpc** definitions,
with descriptions taken from the Python methods' `__doc__` strings,
and **rpc** and **input** leaf names automatically
created from the Python method and argument names
by replacing underscores with hyphens again:




```python
>>> TM_YANG = TM.to_yang(
>>>     prefix='tm', namespace='http://modeled.netconf/turing-machine')
>>> print(TM_YANG)
```

```
module turing-machine {
  namespace "http://modeled.netconf/turing-machine";
  prefix tm;

  revision 2015-10-29;

  container turing-machine {
    leaf state {
      type int64;
    }
    leaf head-position {
      type int64;
    }
    list tape {
      key "cell";
      leaf cell {
        type int64;
      }
      leaf symbol {
        type string;
      }
    }
    list program {
      key "name";
      leaf name {
        type string;
      }
      container rule {
        container input {
          leaf state {
            type int64;
          }
          leaf symbol {
            type string;
          }
        }
        container output {
          leaf state {
            type int64;
          }
          leaf symbol {
            type string;
          }
          leaf head-move {
            type string;
          }
        }
      }
    }
  }
  rpc initialize {
    description
      "Initialize the Turing Machine.";
    input {
      leaf tape-content {
        type string;
      }
    }
  }
  rpc run {
    description
      "Start the Turing Machine operation.";
  }
}
```

Now is a good time to verify if that's really correct YANG.
Just write it to a file:




```python
with open('turing-machine.yang', 'w') as f:
    f.write(TM_YANG)
```


And feed it to the **pyang** command.
Since the **pyang** turorial also produces
a tree format output from its YANG Turing Machine,
I also do it here for comparison
(`!...` runs external programs in IPython):
```
!pyang -f tree turing-machine.yang
```

```
module: turing-machine
   +--rw turing-machine
      +--rw state?           int64
      +--rw head-position?   int64
      +--rw tape* [cell]
      |  +--rw cell      int64
      |  +--rw symbol?   string
      +--rw program* [name]
         +--rw name    string
         +--rw rule
            +--rw input
            |  +--rw state?    int64
            |  +--rw symbol?   string
            +--rw output
               +--rw state?       int64
               +--rw symbol?      string
               +--rw head-move?   string
rpcs:
   +---x initialize
   |  +---w input
   |     +---w tape-content?   string
   +---x run
```

No errors. Great!



## From modeled YANG modules to a NETCONF service



Finally! Time to run a Turing Machine NETCONF server...

First create an instance of the final Turing Machine class
with RPC method definitions:




```python
tm = TM(TM_PROGRAM)
```


Currently only serving NETCONF over
[SSH](https://en.wikipedia.org/wiki/Secure_Shell) is supported.
An SSH service needs a network port and user authentication credentials:




```python
PORT = 12345
USERNAME = 'user'
PASSWORD = 'password'
```


And it needs an SSH key.
If you don't have any key lying around,
the UNIX tool **ssh-keygen** from
[OpenSSH](http://www.openssh.com)
(or Windows tools like
[PuTTY](http://www.chiark.greenend.org.uk/~sgtatham/putty))
can generate one for you.
Just name the file **key**:

> `ssh-keygen -f key`




```python
server = tm.serve_netconf_ssh(
    port=PORT, host_key='key', username=USERNAME, password=PASSWORD)
```


And that's it! The created `server` is an instance of Python
[netconf](https://pypi.python.org/pypi/netconf) project's
`NetconfSSHServer` class.
The server's internals run in a separate thread,
so it doesn't block the Python script.
We can just continue with creating a NETCONF client
which talks to the server.
Let's directly use `NetconfSSHSession`
from the **netconf** project for now.
The Pythonic client features of **MODELED.netconf** are not implemented yet,
but they will also be based on **netconf**.




```python
from netconf.client import NetconfSSHSession
```



```python
client = NetconfSSHSession(
    'localhost', port=PORT, username=USERNAME, password=PASSWORD)
```


Now the Turing Machine can be remotely initialized
with a NETCONF RPC call.
Let's compute unary **2 + 3** this time.
Normally this would also need the Turing Machine's XML namespace,
but namspace handling is not properly supported yet
by **MODELED.netconf**:




```python
reply = client.send_rpc(
    '<initialize><tape-content>110111</tape-content></initialize>')
```


The tape will be set accordingly:




```python
>>> tm.tape
['1', '1', '0', '1', '1', '1']
```



Now run the Turing Machine via RPC:




```python
reply = client.send_rpc('<run/>')
```



```python
>>> tm.state, tm.tape
(4, ['1', '1', '1', '1', '1', '0'])
```



As expected!

