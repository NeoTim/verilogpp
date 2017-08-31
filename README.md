# Verilogpp User Manual

<!-- This documentation is automatically generated by verilogpp.
  -- Changes should be made to verilogpp itself. -->

## Usage

    verilogpp [options] [files]

Options:

| Option | Description |
|--------|-------------|
| -v --verbose | Turn on verbose reporting. |
| -q --quieter | Make verilogpp quieter. |
| -k --check | Does a dry run, checks if changes would be made. |
| -c --config | Specify a configuration file. |
| -t --trim | Trim all macro expansions out, returning file to an un-pre-processed state. |
| -i --incdir dir | Appends dir to the list of paths to search for included files |
| -r --recursive | Causes the preprocessor to recursively reprocess all dependencies for the current file as well. |
| -p --prettyprint | Set to enable extra formatting and annotations in generated code. |
| -f --fsm_state_suffix | Suffix string added to FSM state decode wires. |
| -h --help | This message. |
| -l --longhelp | Dumps macro documentation as well. |
| --mdhelp | Dumps user manual as markdown. |

## Introduction

`verilogpp` is a preprocessor that operates on a set of
verilog files.  It provides functionality roughly equivalent to
that of the verilog mode for emacs, except that the results are
maintainable and repeatable.

Verilogpp macros are always embedded in code as specially formatted
comments that begin with `/**` rather than the usual `/*`.  See the
[MACROS](#macros) section below for details.

It operates in two modes:

* Process: When passed a file that ends with the "pp" extension (for example,
  "foo.svpp"), it creates a preprocessed output file using a filename with the
  final "pp" token stripped away (for example, "foo.sv").

* In-Place: Otherwise, verilogpp operates in-place on the files it is provided.

##### Generated code

The code generated by verilogpp is always placed between a pair of
comments that indicate that the code between them is machine-generated
and should never be hand-editted.  Instead, the generating macro should
be changed instead, and the code reprocessed by the preprocessor.

The comments that denote machine generated code always look like this:

    /*PPSTART*/
    ... machine generated code lives here ...
    /*PPSTOP*/

##### Deprecated macros

The [PERL](#perl), [FIXEDPOINT](#fixedpoint), [REG](#reg), and
[STRUCT](#struct) macros are deprecated.  They may be dropped in future
versions of this tool, and new code should avoid using them.

##### Folding generated code

If possible, consider configuring your editor to "fold" all code
between the PPSTART and PPSTOP comments out of your visual field.

In VIM, you could add the following to your .vimrc file:

    autocmd BufWinEnter,Syntax *.v,*.sv syn region ppoutput
      \ start="\/\*PPSTART\*" end="\*PPSTOP\*" fold contains=TOP keepend
    autocmd BufWinEnter,Syntax *.v,*.sv set foldmethod=syntax

##### This document

This user manual is generated directly from the verilogpp source code using the
`--mdhelp` option.  Please be sure to make improvements there, and regenerate
this file, rather than editing this file directly.

## Configuration File

TODO(jonmayer): Add documentation for the verilogpp configuration file.


## Macros

Quick links:

*   [AUTOINC](#Macro_AUTOINC)
*   [AUTOINTERFACE](#Macro_AUTOINTERFACE)
*   [AUTONET](#Macro_AUTONET)
*   [AUTOTOSTRING](#Macro_AUTOTOSTRING)
*   [EXEC](#Macro_EXEC)
*   [FIXEDPOINT](#Macro_FIXEDPOINT)
*   [FOREACH](#Macro_FOREACH)
*   [FORINST](#Macro_FORINST)
*   [FSM](#Macro_FSM)
*   [INST](#Macro_INST)
*   [PERL](#Macro_PERL)
*   [REG](#Macro_REG)
*   [STRUCT](#Macro_STRUCT)
*   [TIEOUTPUTSTOZERO](#Macro_TIEOUTPUTSTOZERO)

### AUTOINC {#Macro_AUTOINC}

The AUTOINC macro can be used to generate dependency metadata for all
sub-modules instantiated using INST and FORINST macros.  It is meant to be
placed at the top of a verilog source file, outside of any module declaration.

This macro is largely obsolete, as modern tool flows manage dependencies by
directly parsing Verilog source files.

The old way of managing Verilog dependencies was to tick-include all immediate
dependencies. AUTOINC generates tese tick-include directives as default
behavior.

Alternately, there was another older tool flow that managed dependencies using
specially formatted DEPEND comments in source code.  If the `depend_mode`
configuration option is set in verilogpp's configuration file, these comments
will be generated instead of tick-includes.

Example of use:

    // Alpha is a module that instantiates a Foo.

    /**AUTOINC**/

    module Alpha(...);

    /**INST foo.sv i_foo**/

    endmodule


### AUTOINTERFACE {#Macro_AUTOINTERFACE}

AUTOINTERFACE can be used to generate a set of ports for a module based on the
set of unbound ports exported by submodules.  The upshot of this is that adding
a port to a sub-module can automatically cause that port to be propagated to
the enclosing module's interface.

Rules:

* Any net that has zero drivers is declared as an input.

* Any net with zero load is declared as an output.

* Bidirectional ports on submodules are always propagated up.

* AUTOINTERFACE currently only knows about nets on submodules instantiated with
  INST or FORINST, and wires explicitly driven with assign statements.  It does
  not support a more sophisticated parsing of logic within a module, and the
  recommended use case is for modules that only stitch together instances,
  without adding new module-level logic.

In the following example, both the Alpha and Beta modules are instantiated in
an AlphaAndBeta module.  All ports are connected to nets of the same name.  Any
ports not used to communicate between Alpha and Beta are automatically added
to the AlphaAndBeta interface:

    module AlphaAndBeta (
      /**AUTOINTERFACE**/
    );

    /**AUTONET**/

    /**INST alpha.v i_a**/

    /**INST beta.v i_b**/

    endmodule

The above example might expand to something like this:

    module AlphaAndBeta (
      /**AUTOINTERFACE**/
      /*PPSTART*/
      input logic [3:0] in,
      output logic [31:0] out
      /*PPSTOP*/
    );

    /**AUTONET**/
    /*PPSTART*/
    logic [7:0] b;
    logic [15:0] c;
    /*PPSTOP*/

    /**INST alpha.v i_a**/
    /*PPSTART*/
    Alpha i_a (
      .in(in),
      .b(b),
      .c(c));
    /*PPSTOP*/

    /**INST beta.v i_b**/
    /*PPSTART*/
    Beta i_b (
      .b(b),
      .c(c),
      .out(out));
    /*PPSTOP*/

    endmodule

### AUTONET {#Macro_AUTONET}

Usage:

    /**AUTONET [--init] [--warn] [--cb]**/

AUTONET can be used to automatically declare all nets used to connect to
ports using the INST macro.

The following rules apply:

* a net with no drivers is declared as a reg.
* a net with one (or more) drivers is declared as a wire.
* a net that matches a signal in this block's own module interface
  list is omitted from the set of nets declared by AUTONET.
* The AUTONET macro won't keep track of the nets associated with
  a manually instantiated module (TODO(jonmayer): fix this.)

If a module only uses INST to declare module instances, then AUTONET can be
used to declare the complete interconnect.  AUTONET is also useful for quickly
declaring the wires and regs of an instantianted module-under-test for
testbenches.

If the "--init" option is supplied to the AUTONET macro, then verilogpp will
also generate code to initialize all signals that are inputs to instantiated
modules to zero.  (This is useful for unit tests.)

If the "--warn" option is supplied to the AUTONET macro, then verilogpp will
annotate the generated nets with warnings if a net appears to have zero
drivers, multiple drivers, or zero load.  Note that verilogpp does not
fully parse verilog, so AUTONET's warnings are only accurate in designs
that only contain expansions of INST macros.

If the "--cb" option is supplied, verilogpp will create appropriate clocking
blocks.  verilogpp's ideas of appropriate clocking blocks are:

* any undriven signal is considered an output from the clocking block.
* any unloaded signal is considered an input to the clocking block.
* interconnect signals (signals with at both loads and a driver) are
  not represented in the clocking block.
* Additionally, signals in different clock domains (identified by naming
  convention: signals in the foo_clk domain must share the foo_ prefix)
  are each broken out into their own clocking block.

For each clock domain, two clocking blocks are generated:

* a "cb" clocking block, which declares inputs and outputs, for use by
  driver code.
* an "mcb" clocking block, which declares all signals as inputs, for use
  by monitor code.

AUTONET is slightly magic in ways that improve style guide compliance and
readability, as follows:

* one-bit wide bit arrays (ie. "wire [0:0] foo") are declared by autonet as
  style-compliant single bit nets (ie. "wire foo").

* AUTONET recognizes that variables declared as "wire [$bits(some_type_t)-1:0]
  bar" are better declared as "some_type_t bar" -- and makes this so.  This
  only works if the data type is declared with a style guide compliant name
  (that is, the type name must end with "_t").

TODO(jonmayer): should interconnect signals be treated as inputs to
clocking blocks, instead of being ignored?

TODO(jonmayer): Add options to make AUTONET more flexible (filterable?).

Example:

    /**AUTONET**/


### AUTOTOSTRING {#Macro_AUTOTOSTRING}

AUTOTOSTRING automatically generates a set of functions for pretty-printing
any struct definitions found within the current source module.

For any struct "X" found in the current code unit, AUTOTOSTRING produces:

* X_ToString: A function for producing a human-readable string representation.
* X_ToJSON: A function for producing a JSON representation.

If no filename is specified, AUTOTOSTRING parses the current file.

If one or more filenames are specified, AUTOTOSTRING parses the specified
files for struct definitions.

For example, this source:

    typedef struct packed {
      logic [7:0] foo;
      logic bar;
      reg [127:0] zap;
    } foobarzap;
    //
    /**AUTOTOSTRING**/

Expands to this output file:

    typedef struct packed {
      logic [7:0] foo;
      logic bar;
      reg [127:0] zap;
    } foobarzap;
    //
    /**AUTOTOSTRING**/
    /*PPSTART*/
    function string foobarzap_ToString(foobarzap value);
      return {
        "foobarzap{",
        $sformatf("foo=%x,", value.foo),
        $sformatf("bar=%x,", value.bar),
        $sformatf("zap=%x", value.zap),
        "}"
      };
    endfunction
    //
    function string foobarzap_ToJSON(
        foobarzap value,
        string sep = "\n    ");
      return {
        "{", sep,
        $sformatf("\"bar\": %d,", value.bar), sep,
        $sformatf("\"foo\": %d,", value.foo), sep,
        $sformatf("\"zap\": %d", value.zap), sep,
        "}"
      };
    endfunction
    /*PPSTOP*/


### EXEC {#Macro_EXEC}


The EXEC macro invokes a separate code-generating program in a subshell, and
captures its output.  The stdout output of that program is the generated code.

The first line of the EXEC macro specifies the path to the program to execute,
as well as any commandline parameters.

TODO(jonmayer): make the rest of the macro text available to the program's
stdin.

Example:

    /**EXEC ./scripts/generate_code.py **/


### FIXEDPOINT {#Macro_FIXEDPOINT}

*DEPRECATED*: please use a struct instead.

The FIXEDPOINT macro constructs a family of `defines that describe a
fixed-point integer format.  The three parameters to this macro are:

* name: the name of the new fixed-point type
* intbits: the number of integral bits associated with this format.
* frabits: the number of fractional bits associated with this format.

The FIXEDPOINT macro will define four constants:

* (name)_INTBITS: the number of integral bits associated with this format.
* (name)_FRABITS: the number of fractional bits associated with this format.
* (name)_WIDTH: the total number of bits needed to represent this format.
* (name)_WORD: the word macro to use when declaring a word of this type.
* (name)_INT: the bit range of the integer field of this structure
* (name)_FRA: the bit range of the fractional field of this structure

Example:

    /**FIXEDPOINT DISTANCE 5 3 **/


### FOREACH {#Macro_FOREACH}

This macro is used to iterate over a list of items.  The macro text
is expanded so that the special character '%' is expanded to be the
iterand.

While the functionality provided by the FOREACH macro is similar to
SystemVerilog's `generate` statement, FOREACH provides the added
flexibility of being able to iterate over arbitrary (rather than
exclusively numeric) tokens, and the ability to concatenate strings
into new tokens.

For example:

    /**FOREACH a b c d
      assign bus_% = %;
    **/

Expands to:

    /**FOREACH a b c d
      assign bus_% = %;
    **/
    /*PPSTART*/
    assign bus_a = a;
    assign bus_b = b;
    assign bus_c = c;
    assign bus_d = d;
    /*PPSTOP*/

A literal '%' character can be embedded with the special sequence '%%'.

Unfortunately, the useful character '#' is used by verilogpp to denote a
comment in all verilogpp macro descriptions.  A literal '#' character can be
embedded with the special sequence '\#'.

The special token `${index}` expands to the index of the current iterable.
This can be useful for mapping non-integer iterables onto integers.

For example:

    /**FOREACH a b c
      assign signal[${index}] = %;
    **/

Expands to:

    /**FOREACH a b c
      assign signal[${index}] = %;
    **/
    /*PPSTART*/
    assign signal[0] = a;
    assign signal[1] = b;
    assign signal[2] = c;
    /*PPSTOP*/

For readability, the items to iterate over can be split across multiple
lines if the macro text is separated from the macro header by a sequence
of four dashes, like this:

    /**FOREACH alpha
               beta
               gamma
               omega
               ----
               reg %_reg;
    **/

A range can be iterated over as well.  There are 3 different ways to
specify a numeric range:

* a start and end with an ellipsis, ie. "0 ... 99"
* a start and end with a colon, ie. "0:99"
* a start, step, and end with colons, ie. "0:3:9"

For example:

    /**FOREACH 0 ... 99
       bitslice \#(SLICE=%) slice%(clk, rst_n, din[%], dout[%]);
    **/

    /**FOREACH 0:4:16
       nibbleslice \#(SLICE=(%/4) nibslice%(.din[(%+3):%]);
    **/

(Note the required escaping of the "#" character in the parameter
section of the instantiation.)

##### Optional flags

Optionally, the "--dense" flag can be supplied on the first line of
the FOREACH macro.  Under normal operation, FOREACH generates comments
indicating the beginning of each iteration of the generated code.  The
"--dense" flag disables this behavior.

One additional obscure feature: when using FOREACH to generate the final
set of signals in a module interface block, it is sometimes necessary to
ask the preprocessor to remove the final comma from the generated string.
The "--separator" flag is useful in this scenario, as it inserts a separator
string between each foreach item, but does not append the separator to
the final element.  For example:

    /**FOREACH a b c --separator=,
      input foo_%
    **/

TODO(jonmayer): Support '${enumerator}` as a preferred alterative to `%`.


### FORINST {#Macro_FORINST}

FORINST is a iterated version of the INST macro.

Usage:

    /**FORINST <path> <instance> <iterable>
    <specifications>
    **/

Where:

* "path" is the path to the module to be instantiated.
* "instance" is the instance name to use for the instantiation.
* "iterable" indicates an arbitrary sequence of items to iterate over.
* "specifications" is a list of zero or more lines used to map port names
  onto signal names.  This can take a couple of different forms, specified
  below.

See help for the INST macro for details on the path, instance, and
specifications parameters.

The "iterable" argument specifies a sequence of items to iterate over,
identical to the FOREACH macro.  That is, a list of tokens can be
specified:

     a b c d
     3 5 7 11 13

Or a numeric range can be specified, like this:

* a start and end with an ellipsis, ie. "0 ... 99"
* a start and end with a colon, ie. "0:99"
* a start, step, and end with colons, ie. "0:3:9"

The FORINST macro creates a series of instances of the specified module.  A
unique instance name is produced for each instance by concatenating the
"instance" argument with an underscore ("_") followed by the iteration value.

The "specifications" text of the FORINST macro can be used to specify
parameters, and to indicate how port names should be mapped onto signal names.
See the FORINST macro for a description of this mapping scheme.  The FORINST
macro's behavior is modified by making the iteration value available to
subsitution macros in the form of the "${enumerator}" variable.

In many ways, the FORINST macro functionality overlaps with SystemVerilog's
generate statement, but provides additional flexibility for mapping ports onto
nets, and does not require the use of unpacked arrays for interfaces.

Example:

    /**FORINST my_design/src/bus_keeper.v keeper A B C
      s/^busN_/bus${enumerator}_/;
    **/

Expands to:

    /**FORINST my_design/src/bus_keeper.v keeper A B C
      s/^busN_/bus${enumerator}_/;
    **/
    /*PPSTART*/
    BusKeeper keeper_A(
      .busN_enable(busA_enable),
      .busN_data(busA_data));

    BusKeeper keeper_B(
      .busN_enable(busB_enable),
      .busN_data(busB_data));

    BusKeeper keeper_C(
      .busN_enable(busC_enable),
      .busN_data(busC_data));
    /*PPSTOP*/


### FSM {#Macro_FSM}

This macro expands a terse FSM description into a synchronous
FSM implementation.  It is intended to be a more readable,
maintainble means of expressing FSMs.

The first token in the FSM macro is the name of the finite
state machine.

The body text of the FSM macro can consist of state transition
declarations, or of output decode assertions.

##### STATE TRANSITION DECLARATION

The following line declares a state transition:

    <state> -> <newstate> [if <condition>]

The above line declares:

* both "state" and "newstate" are states in the FSM.
* if the "if" clause is provided, then <condition> must
  evaluate to true for the state transition to occur.
* if the "if" clause is omitted, then the transition is
  the default transition that occurs if all other transition
  conditions are false (or not specified).
* conditions are evaluated in the order specified, so each
  condition implicitly depends on prior conditions being false.

Unless otherwise specified by an explicit default transition
declaration, states are stationary (ie. the FSM will remain
in the current state).

The initial state of an FSM is always the first state declared.

##### OUTPUT DECODE ASSERTION

After any state transition declaration, a list of assignements
can be made.  These assignments only take place when the condition
defined by the preceding state transition declaration line
evaluates to true.

For example:

    IDLE -> RUNNING if start
      pulse = 1'b1

Indicates that "pulse" will pulse high for one cycle, when in
the IDLE state and the "start" signal is active.

To declare outputs that should be true by default when in
a given state, use the "state:" declaration.  For example:

    RUNNING:
      is_running = 1'b1
      bar_the_foo = 1'b1
    RUNNING -> IDLE if done
      bar_the_foo = 1'b0
      # but is_running is still asserted

All variables assigned to by output decode assertions should be
pre-declared as 'reg' variables.

##### GENERAL

The generated FSM implementation exports its state variable as
a net named:

    <name>_state

The states are identified by localparam declarations.  These
constants are given names by converting the FSM name and the
state name to all upper case characters, and concatenating thus:

    <NAME>_<STATE>

For example:

    /**FSM copier
      default:
        read = 1'b0
        write = 1'b0
      idle -> read1 if start
         read = 1'b1
      read1 -> read2
      read2 -> read3
      read3 -> write1
      write1:
        write = 1'b1
      write1 -> write2 if ack
      write2 -> idle
    */

The generated FSM will export a net named "copier_state," and
will also export localparam symbols with names such as
"COPIER_IDLE", "COPIER_READ1", etc.

Note that the default transitions "idle -> idle" and "write1 ->
write1" are implicit.

##### OPTIONS

If the "--enum" option is specified on the first line of the FSM macro, then an
enum is used to generate the state variables instead of a bitvector and a set
of localparam declarations.


### INST {#Macro_INST}

    /**INST <path> <instance>
    <specifications> **/

Where:

* "path" is the path to the module to be instantiated.
* "instance" is the instance name to use for the instantiation.
* "specifications" is a list of zero or more lines used to map port names
  onto signal names.  This can take a couple of different forms.

The INST macro creates an instance of the specified module.  The
"specifications" text of the INST macro can be used to specify parameters,
and to indicate how port names should be mapped onto signal names.

Parameters can be specified for the instantiated block like this:

     parameter <param-name> <param-value>

Transformation of port names onto signal names occurs by applying a
set of transformation rules onto the port name to produce the signal
name that port should connect to.

The simplest transformation simply maps a port to a signal, like this:

     .portname(signalname)

Alternately, a regular expression substitution can be performed on the
port name to produce the signal name:

     s/^bus_/thisbus_/;

The INST macro is often used in conjunction with the AUTONET macro to
automatically generate correct net declarations for all nets mapped onto the
ports of INST-instantiated submodules.  Additionally, the AUTOINTERFACE macro
can be used to propagate unbound ports from INST-instantiated submodules to the
module interface.

See also: the AUTONET, AUTOINTERFACE, and FORINST macros.

Example:

    /**INST snout/rtl/sn_ear_eeprom.v ear_eeprom1
      parameter Width 12
      .clk(clk_sys)
      s/^i2c_/info_smbus_ear1_/;
    **/


### PERL {#Macro_PERL}


*DEPRECATED*: Use the EXEC macro instead.

This macro is expanded by evaluating an embedded perl script.
The STDOUT produced by the embedded perl script is the generated
verilog code.  Think of it like a "generate" statement on steroids.

This macro improves code by providing a more maintainable
alternative to the copy-paste-edit cycle.

Example:

    /**PERL
     * my @signals = qw(a b c d);
     * print "  " . join(",\n  ",
     *              map { ".out_$_(in_$_)" }
     *              @signals) . "\n";
     */


### REG {#Macro_REG}

*DEPRECATED*: Use a macro instead.

This macro provides a concise way to instantiate registers in a consistent
fashion.  Register mapping is specified using a simple RTL syntax:

    register <- value

Alternately, conditional loading of registers can be specified:

    register <- value if condition

Clock and reset default to "clock" and "reset_n" respectively, but can
be overridden with the "clk:" and "rst:" directives.

Example:

    /**REG
       foo_d <- foo
       bar <- bus_data if bus_wren & bus_sel
       clk:clk_sys
       bum_d <- bum
    **/


### STRUCT {#Macro_STRUCT}

*DEPRECATED*: This macro was useful when working with Verilog-2001 code,
but modern code should use SystemVerilog struct constructs.

The STRUCT macro is used to concatenate a series of signals
into a data-structure like bus, by defining a set of macros
to simplify the assembly and disassembly of that bus.

Each line is a data member, optionally prefixed by a word size.  If
word size is omitted, the data member is assumed to be a single bit wide.

By default, fields in the structure are concatenated in order so that
the first element is the least-signficant part of the structure.  If
the reverse order is preferred, supply the "--reverse" option to the
macro declaration, and the first item will be the most significant part
of the structure.

Options: if the "--reverse" option is supplied to the macro declaration
line, the fields of the structure are produced in reverse order (ie. first
item is the most signficiant part of the packed structure word).

Example:

    /**STRUCT mystructure
         foo
         bar
      16 word
      8  byte
    **/
    wire [`MYSTRUCTURE_WORD] x = {my_byte, my_word, my_bar, my_foo};

    /**STRUCT otherstruct --reverse
      8 msb
      8 lsb
    **/
    wire [`OTHERSTRUCT_WORD] s = {my_msb, my_lsb};


### TIEOUTPUTSTOZERO {#Macro_TIEOUTPUTSTOZERO}

This macro can be used to automatically tie all outputs of the current
module to zeroes, enabling quick creation of stub modules.

For example:

    module foo_stub(
      input logic a,
      output logic b, c, d);

    /**TIEOUTPUTSTOZERO**/

    endmodule

Expands to:

    module foo_stub(
      input logic a,
      output logic b, c, d);

    /**TIEOUTPUTSTOZERO**/
    /*PPSTART*/
    assign b = '0;
    assign c = '0;
    assign d = '0;
    /*PPSTOP*/

    endmodule


## License

`verilogpp` is distributed under the terms of the [Apache 2.0 License](LICENSE-2.0.txt).
Please see [CONTRIBUTING.md] for information on how to contribute.

While Google Inc retains copyright for much of the code of `verilogpp`, this is not
an official Google product.

