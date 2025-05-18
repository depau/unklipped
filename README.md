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
PY var = 42
PY var
PY import sys
PY sys.version
```

> [!NOTE]
> The `PY` macro only works if the Python code does not start with a number. If it does, consider using
> `PYTHON PY="..."` instead.

### `PYTHON PY="..."`

The `PYTHON` macro allows you to run code that Klipper is not able to parse, such as code that starts with a number.

```gcode
PYTHON PY="print('Hello, world!')"
PYTHON PY="var = 42"
PYTHON PY="var"
PYTHON PY="42 + 42"  # This does not work with PY but works with PYTHON
```

## Running a Python script from a file

You can run a Python script from a file using the `PYTHON` macro.

```gcode
PYTHON RUN_SCRIPT="path/to/script.py"
```

## Creating Python macros

Although a bit hacky, you can create Python macros that can be called from G-code.

Here's an example:

```gcode
[gcode_macro PY_MACRO_EXAMPLE]
gcode:
	{% set scr = printer.make_script("py_macro_example", params=params) %}
	PYS S{scr} def test():
	PYS S{scr}     print("hello world!")
	PYS S{scr}     return 42
	PYS S{scr} test()
	PYS S{scr} print(params)
	PYTHON RUN_SCRIPT={scr}
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

```gcode
[gcode_macro PY_MACRO_EXAMPLE]
gcode:
  {% set scr = printer.make_script("py_macro_example", params=params) %}
  PYTHON SCRIPT={scr} ADD="def test():"
  PYTHON SCRIPT={scr} ADD="    print('hello world!')"
  PYTHON SCRIPT={scr} ADD="    return 42"
  PYTHON SCRIPT={scr} ADD="test()"
  PYTHON SCRIPT={scr} ADD="print(params)"
  PYTHON RUN_SCRIPT={scr}
```

### `PYTHON RUN_SCRIPT={scr}`

The `PYTHON` macro runs the script. The `RUN_SCRIPT` parameter is the script identifier returned by
`printer.make_script()`.

>[!IMPORTANT]
> Note that each script can only be run once. Script IDs are invalidated when RUN_SCRIPT is called.

## License

This project is licensed under the LGPL-2.1 License - see the [LICENSE](LICENSE) file for details.
