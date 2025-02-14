(type-translations)=
# Type translations
In order to communicate between Python and Javascript, we "translate" objects
between the two languages. Depending on the type of the object we either
translate the object by implicitly converting it or by proxying it. By
"converting" an object we mean producing a new object in the target language
which is the equivalent of the object from the source language, for example
converting a Python string to the equivalent a Javascript string. By "proxying"
an object we mean producing a special object in the target language that
forwards requests to the source language. When we proxy a Javascript object into
Python, the result is a `JsProxy` object. When we proxy a Python object into
Javascript, the result is a `PyProxy` object. A proxied object can be explicitly
converted using the explicit conversion methods `JsProxy.to_py` and
`PyProxy.toJs`.

Python to Javascript translations occur:

- when returning the final expression from a {any}`pyodide.runPython` call,
- when using {any}`pyodide.pyimport`,
- when passing arguments to a Javascript function called from Python,
- when returning the results of a Python function called from Javascript,
- when accessing an attribute of a `PyProxy`

Javascript to Python translations occur:

- when using the `from js import ...` syntax
- passing arguments to a Python function called from Javascript
- returning the result of a Javascript function called from Python
- when accessing an attribute of a `JsProxy`

`````{admonition} Memory Leaks and Python to Javascript translations
:class: warning

Any time a Python to Javascript translation occurs, it may create a `PyProxy`.
To avoid memory leaks, you must store the result and destroy it when you are
done with it. Unfortunately, we currently provide no convenient way to do this,
particularly when calling Javascript functions from Python.
`````

## Round trip conversions 
Translating an object from Python to Javascript and then back to
Python is guaranteed to give an object that is equal to the original object
(with the exception of `nan` because `nan != nan`). Furthermore, if the object
is proxied into Javascript, then translation back unwraps the proxy, and the
result of the round trip conversion `is` the original object (in the sense that
they live at the same memory address).

Translating an object from Javascript to Python and then back to
Javascript gives an object that is `===` to the original object (with the
exception of `NaN` because `NaN !== NaN`, and of `null` which after a round trip
is converted to `undefined`). Furthermore, if the object is proxied into Python,
then translation back unwraps the proxy, and the result of the round trip
conversion is the original object (in the sense that they live at the same
memory address).

## Implicit conversions

We only implicitly convert immutable types. This is to ensure that a mutable
type in Python can be modified in Javascript and vice-versa. Python has
immutable types such as `tuple` and `bytes` that have no equivalent in
Javascript. In order to ensure that round trip translations yield an object of
the same type as the original object, we proxy `tuple` and `bytes` objects.
Proxying tuples also has the benefit of ensuring that implicit conversions take
a constant amount of time.

### Python to Javascript
The following immutable types are implicitly converted from Javascript to
Python:

| Python          | Javascript          |
|-----------------|---------------------|
| `int`           | `Number`            |
| `float`         | `Number`            |
| `str`           | `String`            |
| `bool`          | `Boolean`           |
| `None`          | `undefined`         |

### Javascript to Python
The following immutable types are implicitly converted from Python to
Javascript:

| Javascript      | Python                          |
|-----------------|---------------------------------|
| `Number`        | `int` or `float` as appropriate |
| `String`        | `str`                           |
| `Boolean`       | `bool`                          |
| `undefined`     | `None`                          |
| `null`          | `None`                          |


## Proxying

Any of the types not listed above are shared between languages using proxies
that allow methods and some operations to be called on the object from the other
language.

### Proxying from Javascript into Python

When most Javascript objects are translated into Python a `JsProxy` is returned.
The following operations are currently supported on a `JsProxy`. (More should be
possible in the future -- work is ongoing to make this more complete):

| Python                    | Javascript             |
|---------------------------|------------------------|
| `str(proxy)`              | `x.toString()`         |
| `proxy.foo`               | `x.foo`                |
| `proxy.foo = bar`         | `x.foo = bar`          |
| `del proxy.foo`           | `delete x.foo`         |
| `hasattr(proxy, "foo")`   | `"foo" in x`           |
| `proxy(...)`              | `x(...)`               |
| `proxy.foo(...)`          | `x.foo(...)`           |
| `proxy.new(...)`          | `new X(...)`           |
| `len(proxy)`              | `x.length` or `x.size` |
| `foo in proxy`            | `x.has(foo)`           |
| `proxy[foo]`              | `x.get(foo)`           |
| `proxy[foo] = bar`        | `x.set(foo, bar)`      |
| `del proxy[foo]`          | `x.delete(foo)`        |
| `proxy1 == proxy2`        | `x === y`              |
| `proxy.typeof`            | `typeof x`             |
| `iter(proxy)`             | `x[Symbol.iterator]()` |
| `next(proxy)`             | `x.next()`             |
| `await proxy`             | `await x`              |
| `proxy.object_entries()`  | `Object.entries(x)`    |

Some other code snippets:
```py
for v in proxy:
    # do something
```
is equivalent to:
```js
for(let v of x){
    // do something
}
```
The `dir` method has been overloaded to return all keys on the prototype chain
of `x`, so `dir(x)` roughly translates to:
```js
function dir(x){
    let result = [];
    do {
        result.push(...Object.getOwnPropertyNames(x));
    } while (x = Object.getPrototypeOf(x));
    return result;
}
```

As a special case, Javascript `Array`, `HTMLCollection`, and `NodeList` are
container types, but instead of using `array.get(7)` to get the 7th element,
Javascript uses `array["7"]`. For these cases, we translate:

| Python                    | Javascript             |
|---------------------------|------------------------|
| `proxy[idx]`              | `x.toString()`         |
| `proxy[idx] = val`        | `x.foo`                |
| `idx in proxy`            | `idx in array`         |
| `del proxy[idx]`          | `proxy.splice(idx)`    |


### Proxying from Python into Javascript

When most Python objects are translated to Javascript a `PyProxy` is produced.
Fewer operations can be overloaded in Javascript than in Python so some
operations are more cumbersome on a `PyProxy` than on a `JsProxy`. The following
operations are currently supported:

| Javascript                            | Python                   |
|---------------------------------------|--------------------------|
| `foo in proxy`                        | `hasattr(x, 'foo')`      |
| `proxy.foo`                           | `x.foo`                  |
| `proxy.foo = bar`                     | `x.foo = bar`            |
| `delete proxy.foo`                    | `del x.foo`              |
| `Object.getOwnPropertyNames(proxy)`   | `dir(x)`                 |
| `proxy(...)`                          | `x(...)`                 |
| `proxy.foo(...)`                      | `x.foo(...)`             |
| `proxy.length` or `x.size`            | `len(x)`                 |
| `proxy.has(foo)`                      | `foo in x`               |
| `proxy.get(foo)`                      | `x[foo]`                 |
| `proxy.set(foo, bar)`                 | `x[foo] = bar`           |
| `proxy.delete(foo)`                   | `del x[foo]`             |
| `x.type`                              | `type(x)`                |
| `x[Symbol.iterator]()`                | `iter(x)`                |
| `x.next()`                            | `next(x)`                |
| `await x`                             | `await x`                |
| `Object.entries(x)`                   |  `repr(x)`               |

`````{admonition} Memory Leaks and PyProxy
:class: warning
When proxying a Python object into Javascript, there is no way for Javascript to
automatically garbage collect the Proxy. The `PyProxy` must be manually
destroyed when passed to Javascript, or the proxied Python object will leak. To
do this, call `PyProxy.destroy()` on the `PyProxy`, after which Javascript will
no longer have access to the Python object. If no references to the Python
object exist in Python either, then the Python garbage collector can eventually
collect it.

```javascript
let foo = pyodide.pyimport('foo');
foo();
foo.destroy();
foo(); // throws Error: Object has already been destroyed
```
`````

`````{admonition} Memory Leaks and PyProxy method calls
:class: warning

Every time you access a Python method on a `PyProxy`, it creates a new temporary
`PyProxy` of a Python bound method. If you do not capture this temporary and
destroy it, you will leak the Python object. 
`````
Here's an example:

```pyodide
pyodide.runPython(`
    class Test(dict):
        def __del__(self):
            print("destructed!")
    d = Test(a=2, b=3)
    import sys
    print(sys.getrefcount(d)) # prints 2
`);
let d = pyodide.pyimport("d");
// Leak three temporary bound "get" methods!
let l = [d.get("a", 0), d.get("b", 0), d.get("c", 0)];
d.destroy(); // Try to free dict
// l is [2, 3, 0].
pyodide.runPython(`
    print(sys.getrefcount(d)) # prints 5 = original 2 + leaked 3
    del d # Destructor isn't run because of leaks
`);
```
Here is how we can do this without leaking:
```pyodide
let d = pyodide.pyimport("d");
let d_get = d.get; // this time avoid the leak
let l = [d_get("a", 0), d_get("b", 0), d_get("c", 0)];
d.destroy();
d_get.destroy();
// l is [2, 3, 0].
pyodide.runPython(`
    print(sys.getrefcount(d)) # prints 2
    del d # runs destructor and prints "destructed!".
`);
```
Another exciting inconsistency is that `d.set` is a __Javascript__ method not a
PyProxy of a bound method, so using it has no effect on refcounts or memory
reclamation and it cannot be destroyed.
```pyodide
let d = pyodide.pyimport("d");
let d_set = d.set;
d_set("x", 7);
pyodide.runPython(`
    print(sys.getrefcount(d)) # prints 2, d_set doesn't hold an extra reference to d
`);
d_set.destroy(); // TypeError: d_set.destroy is not a function
```

## Explicit Conversion of Proxies

### Python to Javascript
Explicit conversion of a `PyProxy` into a native Javascript object is done with
the `toJs` method. By default, the `toJs` method does a recursive "deep"
conversion, to do a shallow conversion use `proxy.toJs(1)`. The `toJs` method
performs the following explicit conversions:

| Python           | Javascript          |
|------------------|---------------------|
| `list`, `tuple`  | `Array`             |
| `dict`           | `Map`               |
| `set`            | `Set`               |

In Javascript, `Map` and `Set` keys are compared using object identity unless
the key is an immutable type (meaning a string, a number, a bigint, a boolean,
`undefined`, or `null`). On the other hand, in Python, `dict` and `set` keys are
compared using deep equality. If a key is encountered in a `dict` or `set` that
would have different semantics in Javascript than in Python, then a
`ConversionError` will be thrown.

`````{admonition} Memory Leaks and toJs
:class: warning

The `toJs` method can create many proxies at arbitrary depth. It is your
responsibility to manually `destroy` these proxies if you wish to avoid memory
leaks, but we provide no way to manage this. 
`````

To ensure that no `PyProxy` is leaked, the following code suffices:
```js
function destroyToJsResult(x){
    if(!x){
        return;
    }
    if(x.destroy){
        x.destroy();
        return;
    }
    if(x[Symbol.iterator]){
        for(let k of x){
            freeToJsResult(k);
        }
    }
}
```


### Javascript to Python
Explicit conversion of a `JsProxy` into a native Python object is done with the
`to_py` method. By default, the `to_py` method does a recursive "deep"
conversion, to do a shallow conversion use `proxy.to_py(1)` The `to_py` method
performs the following explicit conversions:

| Javascript       | Python              |
|------------------|---------------------|
| `Array`          | `list`              |
| `Object`**       | `dict`              |
| `Map`            | `dict`              |
| `Set`            | `set`               |

** `to_py` will only convert an object into a dictionary if its constructor
is `Object`, otherwise the object will be left alone. Example:
```pyodide
class Test {};
window.x = { "a" : 7, "b" : 2};
window.y = { "a" : 7, "b" : 2};
Object.setPrototypeOf(y, Test.prototype);
pyodide.runPython(`
    from js import x, y
    # x is converted to a dictionary
    assert x.to_py() == { "a" : 7, "b" : 2} 
    # y is not a "Plain Old JavaScript Object", it's an instance of type Test so it's not converted
    assert y.to_py() == y
`);
```

In Javascript, `Map` and `Set` keys are compared using object identity unless
the key is an immutable type (meaning a string, a number, a bigint, a boolean,
`undefined`, or `null`). On the other hand, in Python, `dict` and `set` keys are
compared using deep equality. If a key is encountered in a `Map` or `Set` that
would have different semantics in Python than in Javascript, then a
`ConversionError` will be thrown. Also, in Javascript, `true !== 1` and `false
!== 0`, but in Python, `True == 1` and `False == 0`. This has the result that a
Javascript map can use `true` and `1` as distinct keys but a Python `dict`
cannot. If the Javascript map contains both `true` and `1` a `ConversionError`
will be thrown.

## Buffers

### Converting Javascript Typed Arrays to Python

Javascript typed arrays (`Int8Array` and friends) are translated to Python
`memoryviews`. This happens with a single binary memory copy (since Python can't
directly access arrays if they are outside of the wasm heap), and the data type
is preserved. This makes it easy to correctly convert the array to a Numpy array
using `numpy.asarray`:

```js
let array = new Float32Array([1, 2, 3]);
```

```py
from js import array
import numpy as np
numpy_array = np.asarray(array)
```

### Converting Python Buffer objects to Javascript

Python `bytes` and `buffer` objects are translated to Javascript as
`TypedArray`s without any memory copy at all. This conversion is thus very
efficient, but be aware that any changes to the buffer will be reflected in both
places.

Numpy arrays are currently converted to Javascript as nested (regular) Arrays. A
more efficient method will probably emerge as we decide on an ndarray
implementation for Javascript.


## Importing Python objects into Javascript

A Python object in the `__main__` global scope can imported into Javascript
using the {any}`pyodide.pyimport` function. Given the name of the Python object
to import, `pyimport` returns the object translated to Javascript.

```js
let sys = pyodide.pyimport('sys');
```

As always, if the result is a `PyProxy` and you care about not leaking the Python 
object, you must destroy it when you are done.

(type-translations_using-js-obj-from-py)=
## Importing Javascript objects into Python

Javascript objects in the
[`globalThis`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/globalThis)
global scope can be imported into Python using the `js` module.

When importing a name from the `js` module, the `js` module looks up Javascript
attributes of the `globalThis` scope and translates the Javascript objects into
Python. You can create your own custom Javascript modules using
{any}`pyodide.registerJsModule`.

```py
import js
js.document.title = 'New window title'
from js.document.location import reload as reload_page
reload_page()
```
