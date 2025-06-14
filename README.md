# Unklipped

Uncircumsize your Klipper and make it run Python code in macros, as it should.

No additional modules/extensions are needed, just a single config file to include in your `printer.cfg`.

Inspired by [Kalico](https://docs.kalico.gg/).

### Why not just use Kalico?

Because [RatOS](https://os.ratrig.com/) comes with Klipper and it's currently not compatible with Kalico. Also some
functionality provided by these macros is not available in Kalico.

## Installation

1. Copy the `unklipped.cfg` to your Klipper config directory.
2. Add the following line to your `printer.cfg`:
    ```ini
    [include unklipped.cfg]
    ```

## Running Python code directly

### `PY ...`

Runs Python code directly, as if it were in a Python REPL.

```gcode
PY print("Hello, world!")
; Hello, world!
PY var = 42
PY var
; 42
PY import sys
PY sys.version
; '3.9.2 (default, Mar 20 2025, 22:21:41) \n[GCC 10.2.1 20210110]'
```

> [!NOTE]
> The `PY` macro only works if the Python code does not start with a number. If it does, consider using
> `PYTHON PY="..."` instead.

### `PYTHON PY="..."`

The `PYTHON` macro allows you to run code that Klipper is not able to parse, such as code that starts with a number.

```gcode
PYTHON PY="print('Hello, world!')"
; Hello, world!
PYTHON PY="var = 42"
PYTHON PY="var"
; 42
PYTHON PY="42 + 42"  # This does not work with PY but works with PYTHON
; 84
```

The `SCOPE` optional parameter allows you to specify which `gcode_macro` object stores the local variables (by default
the `PYTHON` macro itself). This is useful if you want to have different scopes for different macros.

```ini
[gcode_macro MY_MACRO]
gcode:
    PYTHON SCOPE=MY_MACRO PY="var = 42"
    PYTHON SCOPE=MY_MACRO PY="print(var)"
```

Any additional parameters passed to the macro are readable from the script in the `params` dict:

```gcode
PYTHON SCOPE=MY_MACRO VAR="Hello, world!" PY="print(params['VAR'])"
; Hello, world!
```

## Running a Python script from a file

You can run a Python script from a file using the `PYTHON` macro.

```gcode
PYTHON RUN_SCRIPT="path/to/script.py"
```

The `SCOPE` optional parameter allows you to specify which `gcode_macro` object stores the local variables (by default
the `PYTHON` macro itself).

```ini
[gcode_macro MY_MACRO]
gcode:
    PYTHON SCOPE=MY_MACRO RUN_SCRIPT="path/to/script.py"
```

Any additional parameters passed to the macro are readable from the script in the `params` dict. You can also use
[`params.forward()`](#paramsforwardexclude).

```ini
[gcode_macro MY_MACRO]
gcode:
    PYTHON SCOPE=MY_MACRO RUN_SCRIPT="/path/to/script.py" DESTINATION="/tmp/hello.txt" {params.forward()}
```

```python
# /path/to/script.py
with open(params["DESTINATION"], "w") as f:
    f.write("Hello, world!")
```

## Running a G-code script template from a file

There are two ways to run a G-code script template from a file depending on your needs. Either way the file is loaded
every time the macro is called, so you can change it without restarting Klipper.

### `printer.load_template()`

The `printer.load_template()` function loads a G-code script template from a file and renders it into the current macro.
This is helpful to have auto-reload while developing a G-code macro.

Note that any parameters, including `params`, must be forwarded to the template explicitly using the keyword arguments.
Variables are loaded automatically from the specified macro name.

```ini
[gcode_macro DISK_MACRO]
gcode:
   {printer.load_template("/home/pi/template.gcode", "DISK_MACRO", params=params)}
```

### `RUN_GCODE_FILE PATH="..."`

The `RUN_GCODE_FILE` macro simply runs a G-code script template from a file. Any additional parameters passed to the
macro are forwarded automatically to the template.

The `VARIABLES_FROM` optional parameter allows you to specify which `gcode_macro` object stores the G-code variables
the template will see. By default, the `RUN_GCODE_FILE` macro itself is used.

>[!NOTE]
> You can use `params.forward()` to forward all parameters of the current macro to the called macro.

```ini
[gcode_macro DISK_MACRO]
variable_test: 42
gcode:
    RUN_GCODE_FILE VARIABLES_FROM=DISK_MACRO PATH="/home/pi/template.gcode" {params.forward()}
```


## Creating Python macros

Although a bit hacky, you can create Python macros that can be called from G-code.

Here's an example:

```ini
[gcode_macro PY_MACRO_EXAMPLE]
gcode:
	{% set scr = printer.make_script("py_macro_example", params=params) %}
	PYS S{scr} def test():
	PYS S{scr}     print("hello world!")
	PYS S{scr}     return 42
	PYS S{scr} test()
	PYS S{scr} print(params)
	PYTHON SCOPE=PY_MACRO_EXAMPLE RUN_SCRIPT={scr}
```

### `printer.make_script(name=None, **params)`

The `printer.make_script()` function starts a new script and returns its identifier. It must be called before providing
any code to the script.

The optional `name` parameter is the name of the script, and it's printed in exception stack traces. If not provided, a
random name is generated.

The keyword arguments are made available to the script as local variables. For example, if you want to access your
macros parameters from the script, just pass them as keyword arguments as demonstrated in the example above.

### `PYS S{scr}`

The `PYS` macro appends a line of code to the script. It must be called after `printer.make_script()`.
The `S` parameter is the script identifier returned by `printer.make_script()`. The code is executed in the context of
the script, so you can use local variables defined in the script.

### `PYTHON SCRIPT={scr} ADD="..."`

As an alternative to `PYS` should Klipper be fussy about its syntax, you can use the `PYTHON` macro as a worse-looking
but otherwise equivalent alternative.

```ini
[gcode_macro PY_MACRO_EXAMPLE]
gcode:
  {% set scr = printer.make_script("py_macro_example", params=params) %}
  PYTHON SCRIPT={scr} ADD="def test():"
  PYTHON SCRIPT={scr} ADD="    print('hello world!')"
  PYTHON SCRIPT={scr} ADD="    return 42"
  PYTHON SCRIPT={scr} ADD="test()"
  PYTHON SCRIPT={scr} ADD="print(params)"
  PYTHON SCOPE=PY_MACRO_EXAMPLE RUN_SCRIPT={scr}
```

### `PYTHON RUN_SCRIPT={scr}`

The `PYTHON` macro runs the script. The `RUN_SCRIPT` parameter is the script identifier returned by
`printer.make_script()`.

>[!IMPORTANT]
> Note that each script can only be run once. Script IDs are invalidated when RUN_SCRIPT is called.

The `SCOPE` optional parameter allows you to specify which `gcode_macro` object stores the local variables (by default
the `PYTHON` macro itself). This is useful if you want to have different scopes for different macros.

```ini
[gcode_macro MY_MACRO]
gcode:
    {% set scr = printer.make_script("py_macro_example", params=params) %}
    PYS S{scr} def test():
    PYS S{scr}     print("hello world!")
    PYS S{scr}     return 42
    PYS S{scr} test()
    PYS S{scr} print(params)
    PYTHON SCOPE=MY_MACRO RUN_SCRIPT={scr}
```

## Other macro helpers

### `params.forward(*exclude)`

The `params.forward()` function formats the parameters of the current macro into a string that can be passed to another
macro. You can pass a list of parameter names to exclude from forwarding.

```ini
[gcode_macro MY_MACRO]
gcode:
    MY_MACRO2 {params.forward("EXCLUDE")}
```

```gcode
MY_MACRO EXCLUDE="secret param" PARAM1="Hello, world!" PARAM2=42
; Child macro called with:
; MY_MACRO2 PARAM1="Hello, world!" PARAM2=42
```

## License

This project is licensed under the LGPL-2.1 License - see the [LICENSE](LICENSE) file for details.
