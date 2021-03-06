MEXPLUS
=======

C++ Matlab MEX development kit.

The kit contains a couple of C++ classes and macros to make MEX development
easy in Matlab. There are 3 major components in the development kit.

 * `mexplus/dispatch.h` Macros to dispatch function calls within a MEX binary.
 * `mexplus/arguments.h` MEX function argument wrappers.
 * `mexplus/mxarray.h` MxArray data conversion and access class.

All classes are located in `mexplus` namespace, and you can use all of them by
including the `mexplus.h` header file.

The library depends on a few C++11 features, and might not be compatible with
older compilers. For older `g++`, make sure to add `-std=c++0x` flag in the
`CXXFLAGS` in the MEX options located at `$HOME/.matlab/$VERSION/mexopts.sh`.

Example
-------

Suppose we have a Database class in C++ and will design a MEX interface to it.
Create the following files.

`Database.cc`

C++ implementation of the MEX interface. It provides MEX entry points by
`MEX_DEFINE` macros and `MEX_DISPATCH` macro at the end. Notice how inputs and
outputs are wrapped by mexplus `InputArguments` and `OutputArguments` class.
They automatically convert majority of C++ types to/from `mxArray`, using C++
template. The `Session` class keeps instances of `Database` objects between MEX
calls, allowing the MEX binary to be stateful.

```c++
// Demonstration of MEX interface to a hypothetical database.
#include <mexplus.h>

using namespace std;
using namespace mexplus;

// Hypothetical database class to be wrapped.
class Database {
public:
  Database(const string& filename);
  virtual ~Database();
  string query(const string& key) const;
  // Other methods...
};

// This initializes the instance session storage for Database.
template class mexplus::Session<Database>;

// Create a new instance of Database and return its session id.
MEX_DEFINE(new) (int nlhs, mxArray* plhs[],
                 int nrhs, const mxArray* prhs[]) {
  InputArguments input(nrhs, prhs, 1);
  OutputArguments output(nlhs, plhs, 1);
  output.set(0, Session<Database>::create(
      new Database(input.get<string>(0))));
}

// Delete the Database instance specified by its id.
MEX_DEFINE(delete) (int nlhs, mxArray* plhs[],
                    int nrhs, const mxArray* prhs[]) {
  InputArguments input(nrhs, prhs, 1);
  OutputArguments output(nlhs, plhs, 0);
  Session<Database>::destroy(input.get(0));
}

// Query to the Database instance specified by its id with a string argument.
MEX_DEFINE(query) (int nlhs, mxArray* plhs[],
                   int nrhs, const mxArray* prhs[]) {
  InputArguments input(nrhs, prhs, 2);
  OutputArguments output(nlhs, plhs, 1);
  const Database& database = Session<Database>::getConst(input.get(0));
  output.set(0, database.query(input.get<string>(1)));
}

// Other entry points...

MEX_DISPATCH
```

`Database.m`

Matlab class interface file. The `id_` property keeps the session ID in the MEX
binary. Each method is a wrapper around corresponding MEX entry points defined
in the C++ file. The first argument of `Database_()` MEX function is the name
defined using `MEX_DEFINE()` macro in the above file.

```matlab
classdef Database < handle
%DATABASE Hypothetical Matlab database API.
properties (Access = private)
  id_ % ID of the session instance.
end

methods
  function this = Database(filename)
  %DATABASE Create a new database.
    assert(ischar(filename));
    this.id_ = Database_('new', filename);
  end

  function delete(this)
  %DELETE Destructor.
    Database_('delete', this.id_);
  end

  function result = query(this, key)
  %QUERY Query to the database.
    assert(isscalar(this));
    result = Database_('query', this.id_, key);
  end

  % Other methods...
end
```

_Build_

The above C++ can be compiled by `mex` command. The output name `Database_` is
the MEX function name used in `Database.m`.

```matlab
mex -Iinclude Database.cc -output Database_
```

After this, the Database class is available in Matlab.

```matlab
database = Database('mydatabase.db');
result = database.query('something');
clear database;
```

The development kit also contains `make.m` build function to make a build
process easier. Customize this file to build your own MEX interface. The kit
depends on some of the C++11 features. In Linux, you might need to add
`CFLAGS="$CFLAGS -std=c++01x"` to `mex` command.

See `example` directory for a complete demonstration.

Dispatching calls
-----------------

MEXPLUS defines a few macros in `mexplus/dispatch.h` that help to create a
single MEX binary with multiple function entries. Create a C++ file that looks
like this:

```c++
//mylibrary.cc
#include <mexplus/dispatch.h>

MEX_DEFINE(myfunc) (int nlhs, mxArray* plhs[],
                    int nrhs, const mxArray* prhs[]) {
  // Do something.
}

MEX_DEFINE(myfunc2) (int nlhs, mxArray* plhs[],
                     int nrhs, const mxArray* prhs[]) {
  // Do another thing.
}

MEX_DISPATCH
```

Notice how `MEX_DEFINE` and `MEX_DISPATCH` macros are used. Then build this
file in Matlab.

```matlab
mex -Iinclude mylibrary.cc
```

The build MEX binaries can now calls two entry points by the first command
argument.

```matlab
mylibrary('myfunc', varargin{:})  % myfunc is called.
mylibrary('myfunc2', varargin{:}) % myfunc2 is called.
```

To prevent from unexpected use, it is a good practice to wrap these MEX calls
in a Matlab class or namescoped functions and place the MEX binary in a private
directory:

    @MyClass/MyClass.m
    @MyClass/myfunc.m
    @MyClass/myfunc2.m
    @MyClass/private/mylibrary.mex*

or,

    +mylibrary/myfunc.m
    +mylibrary/myfunc2.m
    +mylibrary/private/mylibrary.mex*

Inside of `myfunc.m` and `myfunc2.m`, call the `mylibrary` MEX function. This
design pattern is very useful to wrap a C++ class in Matlab.

Parsing function arguments
--------------------------

MEXPLUS provides `InputArguments` and `OutputArguments` classes to ease
parsing, validation, and data conversion of input and output arguments to MEX
functions.

### InputArguments

The class provides a wrapper around input arguments to validate and convert
Matlab variables. You can define a single or multiple input formats to accept.
The `get()` method automatically converts Matlab's `mxArray` to most of the
basic C++ types using a template parameter. Also it can convert to a custom
data type when a template specialization to `MxArray::to()` method is provided.
(See the next section.)

__Example__: The MEX function takes a single numeric input argument.

```c++
// C++
InputArguments input(nrhs, prhs, 1);
myFunction(input.get<double>(0));
```

```matlab
% Matlab
myFunction(1.0);
```

__Example__: The MEX function takes two numeric arguments, and two optional
arguments specified by name-value pairs. When optional arguments are not given,
the function uses a default value.

```c++
// C++
InputArguments input(nrhs, prhs, 2, 2, "option1", "option2");
myFunction(input.get<double>(0),
           input.get<int>(1),
           input.get<string>("option1", "foo"), // default: "foo".
           input.get<int>("option2", 10)); // default: 10.
```

```matlab
% Matlab
myFunction(1.0, 2);
myFunction(1.0, 2, 'option2', 11);
myFunction(1.0, 2, 'option1', 'bar');
myFunction(1.0, 2, 'option1', 'baz', 'option2', 12);
myFunction(1.0, 2, 'option2', 12, 'option1', 'baz');
```

__Example__: The MEX function has two input formats: 1 + 2 arguments or 2 + 2
arguments.

```c++
// C++
InputArguments input;
input.define("format1", 1, 2, "option1", "option2");
input.define("format2", 2, 2, "option1", "option2");
input.parse(nrhs, prhs);
if (input.is("format1"))
    myFunction(input.get<int>(0),
               input.get<string>("option1", "foo"),
               input.get<int>("option2", 10));
else if (input.is("format2"))
    myFunction(input.get<int>(0),
               input.get<vector<double> >(1),
               input.get<string>("option1", "foo"),
               input.get<int>("option2", 10));
```

```matlab
% Matlab
myFunction(1.0);
myFunction(1.0, 'option1', 'foo', 'option2', 10);
myFunction(1.0, [1,2,3,4]);
myFunction(1.0, [1,2,3,4], 'option1', 'foo', 'option2', 10);
```

### OutputArguments

The class provides a wrapper around output arguments to validate and convert
Matlab variables. The `set()` method automatically converts most of the
basic C++ types to Matlab's `mxArray` using a template parameter.

__Example__: The MEX function returns at most 3 arguments. The wrapper doesn't
assign to the output when the number of outputs are less than 3.

```c++
OutputArguments output(nlhs, plhs, 3);
output.set(0, 1);
output.set(1, "foo");
MxArray cell = MxArray::Cell(1, 2);
cell.set(0, 0);
cell.set(1, "value");
output.set(2, cell.release());
```

Data conversion
---------------

Data conversion in MEXPLUS is provided by `MxArray` class.

### MxArray

The MxArray class provides common data conversion methods between `mxArray`
and C++ types, as well as serving itself as a unique_ptr to manage memory.

Two static methods: `MxArray::to()` and `MxArray::from()` are the core of the
high-level conversions. Give a desired type in the template parameter. The
`MxArray::to()` method has two function signatures. The one with a second
pointer argument is to avoid extra copy assignment in the return value.

```c++
int value = MxArray::to<int>(prhs[0]);
string value = MxArray::to<string>(prhs[0]);
vector<double> value = MxArray::to<vector<double> >(prhs[0]);
vector<double> value2;
MxArray::to<vector<double> >(prhs[0], &value2); // No extra copy.

plhs[0] = MxArray::from(20);
plhs[0] = MxArray::from("text value.");
plhs[0] = MxArray::from(vector<double>(20, 0));
```

Additionally, the following object API's are to wrap around a complicated data
construction with automatic memory management. Use `MxArray::release()` to
get a mutable `mxArray` pointer after construction.

```c++
// Read access.
MxArray cell(prhs[0]);   // {x, y, ...}
int x = cell.at<int>(0);
vector<double> y = cell.at<vector<double> >(1);

MxArray struct_array(prhs[0]);   // struct('field1', x, ...)
int x = struct_array.at<int>("field1");
vector<double> y = struct_array.at<vector<double> >("field2");

MxArray numeric(prhs[0]);   // [x, y, ...]
double x = numeric.at<double>(0);
int y = numeric.at<int>(1);
```

```c++
// Write access.
MxArray cell(MxArray::Cell(1, 3));
cell.set(0, 12);
cell.set(1, "text value.");
cell.set(2, vector<double>(4, 0));
plhs[0] = cell.release(); // {12, 'text value.', [0, 0, 0, 0]}

MxArray struct_array(MxArray::Struct());
struct_array.set("field1", 12);
struct_array.set("field2", "text value.");
struct_array.set("field3", vector<double>(4, 0));
plhs[0] = struct_array.release(); // struct('field1', 12, ...)

MxArray numeric(MxArray::Numeric<double>(2, 2));
numeric.set(0, 0, 1);
numeric.set(0, 1, 2);
numeric.set(1, 0, 3);
numeric.set(1, 1, 4);
plhs[0] = numeric.release(); // [1, 2; 3, 4]
```

To add your own data conversion, define in `namespace mexplus` a template
specialization of `MxArray::from()` and `MxArray::to()` with a pointer
argument. This will also enable automatic conversion in `InputArguments` and
`OutputArguments` class.

```c++
class MyObject; // This is your custom data class.

namespace mexplus {
// Define two template specializations.
template <>
mxArray* MxArray::from(const MyObject& value) {
  // Write your conversion code.
}

template <>
void MxArray::to(const mxArray* array, MyObject* value) {
  // Write your conversion code.
}
} // namespace mexplus

// Then you can use any of the following.
MyObject object;
std::vector<MyObject> object_vector;
MxArray::to<MyObject>(prhs[0], &object);
MxArray::to<std::vector<MyObject> >(prhs[1], &object_vector);
plhs[0] = MxArray::from(object);
plhs[1] = MxArray::from(object_vector);
```

Test
----

Run the following to test MEXPLUS.

```matlab
make test
```

TODO
----

_General_

 * Add a script to generate wrapper templates.
  * Maybe, use a compiler front-end to automatically generate a wrapper?
 * Runtime dependency checker.

_MxArray_

 * N-D array composition and decomposition. See
   [this](https://github.com/kyamagu/matlab-bson/blob/master/src/bsonmex.c).
 * Sparse arrays.
