# Introduction

`make4ht` is a simple build system for `tex4ht`. It is both executable which
simplifies `tex4ht` execution and a library which can be used to create
customized conversion programs. An example of such conversion program is [tex4ebook](https://github.com/michal-h21/tex4ebook)

## License

Permission is granted to copy, distribute and/or modify this software
under the terms of the LaTeX Project Public License, version 1.3.

How it works
------------

Default compilation script for `tex4ht`, named `htlatex`  compiles LaTeX files to `HTML` with this command  sequence:

    latex $latex_options 'code for loading tex4ht.sty \input{filename}'
    latex $latex_options 'code for loading tex4ht.sty \input{filename}'
    latex $latex_options 'code for loading tex4ht.sty \input{filename}'
    tex4ht options filename
    t4ht options filename

The problem is that this is inefficient when  you need to run program which
interact with LaTeX, such as `Makeindex` or `Bibtex`. In that case, you need to
create new script based on the default one, or run `htlatex` twice, which means
six LaTeX runs. 

Another problem is with `t4ht` application. It reads file `filename.lg`,
generated by `tex4ht`, where are instructions about generated files, `CSS`
instructions, calls to external applications, instructions for image
conversions etc. It can be instructed to copy generated files to some output
directory, but it doesn't preserve directory structure, so when you
have images in some subdirectory, they will be copied to the output directory,
but links will be pointing to non existing subdirectory.

Image conversion is directed with the 
[env file](http://www.tug.org/applications/tex4ht/mn35.html#index35-73001), 
with really strange syntax based on whitespace and 
[os dependent](http://www.tug.org/applications/tex4ht/mn-unix.html#index27-69005).
With `make4ht` build files, we have simple mean to fix these issues. We can
change image conversion parameters without need to modify the `env file`,
and call actions on the output files. These actions can be either external
programs such as `xslt` processors or `HTML tidy` or `Lua` functions.

The idea is to make system controlled by a build file. Because `Lua`
interpreter is included in modern TeX distributions and `Lua` is ideal language
for such task, it was chosen as language in which build script are written.

# Output file formats and extensions

The default output format used by `make4ht` is `xhtml`. You can request different 
format using `--format` option. Supported formats are:

 - `xhtml`
 - `html5`
 - `odt`

You can also request `make4ht` extensions using `--format` option.  

## Extensions

Extensions can be used to modify the build process without need to use a build file. They
may post-process the output files or request additional commands for the compilation.

The extensions can be enabled or disabled by appending `+EXTENSION` or `-EXTENSION` after
the output format name:

     make4ht -uf html5+tidy filename.tex

Available extensions:

`latexmk`

:    Use `Latexmk` for \LaTeX\ compilation.

`tidy`

:    Clean the `HTML` files using `Tidy` command.

# Build files

`make4ht` supports build files. These are `Lua` scripts which can adjust
the build process. You can request external applications like `bibtex` or `makeindex`,
pass options to the commands, modify the image conversion process, or post-process the 
generated files.

`make4ht` tries to load default build file named as `filename + .mk4 extension`.
You can choose different build file with `-e` or `--build-file` command line
option.

Sample build file:

    Make:htlatex()
    Make:match("html$", "tidy -m -xml -utf8 -q -i ${filename}")

`Make:htlatex()` is preconfigured command for calling LaTeX with `tex4ht`
loaded on the input file. In this case it will be called  one time. After
compilation, `tidy` command is executed on the output `html` file.

Note that you don't have to call `tex4ht` and `t4ht` commands explicitly in the
build file, they are called automatically. 

## User commands

You can add more commands like `Make:htlatex` using `Make:add` command: 

    Make:add("name", "command", {parameters}, repetition)

The `name` and `command` parameters are required, rest of the parameters are optional.

This defines `name` command, which can be then executed as `Make:name()` command.

### Provided commands

`Make:htlatex`

:    One call to TeX engine with special configuration for `tex4ht` loading.

`Make:latexmk`

:    Use `Latexmk` for the document compilation. `tex4ht` is loaded automatically.

`Make:tex4ht`

:    Process the DVI file and creates the output files.

`Make:t4ht`

:    Creates the CSS file.


### Command function

The `command` parameter can be either string template, or function:

    Make:add("text", "echo hello, input file: ${input}")
    Make:add("function", function(params) 
      for k, v in pairs(params) do 
        print(k..": "..v) 
      end, {custom="Hello world"}
    )

The template can get value of variables from the parameters table using
`${var_name}` placeholder. Templates are executed using operating system, so
they should invoke existing OS commands. Function commands may execute system
commands using `os.execute` function.


### Parameters table

`parameters` parameter is optional, it can be table or `nil` value, which
should be used if you want to use the `repetition` parameter, but don't want to
modify the parameters table. 

The table with default parameters is passed to all commands, they can be accessed from command functions  
or templates. When you specify your own parameters in the command definition, these additional 
parameters are added to the default parameters table for this particular
command. You can override the default parameters in the parameters table.



The default parameters are following:

htlatex 

:     used compiler

input 

:    it is output file name in fact

tex_file

:    input TeX file

latex_par 

:    parameters to `latex`

packages

:    insert additionl LaTeX code which is inserted before `\documentclass`.
     Useful for passing options to packages or additional packages loading

tex4ht_sty_par 

:    parameters to `tex4ht.sty`

tex4ht_par 

:     parameters to `tex4ht` application

t4ht_par 

:    parameters to `t4ht` application

outdir 

:     output directory

repetition 

:    limit number of command execution.

correct_exit 

:    expected `exit code` from the command. The compilation will be terminated
     if the command `exit code` is different.


### Repetition

Repetition is number which specify maximal number of executions of the particular command.
This is used for instance for `tex4ht` and `t4ht` commands, as they should be
executed only once in the compilation. They would be executed
multiple times if you include them in the build file, because they are 
called  by `make4ht` by default. Because these commands are allowed only one
`repetition`, the second execution is blocked.

### Expected exit code

You can set the expected exit code from a command with `correct_exit` key in the
parameters table. Compilation is stopped when command returns different exit
code. 

This mechanism isn't used for LaTeX (for all TeX engines and formats, in
fact), because it doesn't differentiate between fatal and non fatal errors, and
it returns the same exit code in all cases. Log parsing is used because of
that, error code `1` is returned in the case of fatal error, `0` is used
otherwise. The `Make.testlogfile` function can be used in the build file to
detect compilation errors in the TeX log file.

## File matches

Other type of action which can be specified in the build file are
matches  which may be called  on the generated files:

    Make:match("html$", "tidy -m -xml -utf8 -q -i ${filename}")

It tests filenames with `Lua` pattern matching and on matched items will
execute command or function, specified in the second argument. For functions,
two arguments are passed, first one is the current filename, the second one
table with parameters. These parameters are the same as in previous section,
except for `filename`, which contains generated output name.

### Filters

Some default match actions which you can use are in the `filter` module.  It
contains some functions which ares useful for fixing some `tex4ht` bugs or
shortcomings.

Example:

    local filter = require "make4ht-filter"
    local process = filter{"cleanspan", "fixligatures", "hruletohr"}
    Make:htlatex()
    Make:htlatex()
    Make:match("html$",process)


Filter module is located in `make4ht-filter`. Function is returned, 
which is used for building filter chains then. 

Built-in filters are:

cleanspan 

:    clean spurious span elements when accented characters are used

cleanspan-nat 

:    alternative clean span filter, provided by Nat Kuhn

fixligatures 

:    decompose ligatures to base characters

hruletohr 

:   `\hrule` commands are translated to series of underscore characters
    by `tex4ht`, this filter translate these underscores to `<hr>` elements

entites 

:    convert prohibited named entities to numeric entities (currently, only
     `&nbsp;`, as it causes validation errors, and `tex4ht` is producing it
     sometimes

fix-links

:    replace colons in local links and `id` attributes with underscores. Some
     cross-reference commands may produce colons in internal links, which results in
     validation error.

Function `filter` accepts also function arguments, in this case this function 
takes file contents as parameter and modified contents are returned.

Example:

    local filter  = require "make4ht-filter"                                    
    local changea = function(s) return s:gsub("a","z") end
    local process = filter{"cleanspan", "fixligatures", changea}            
    Make:htlatex()                                                              
    Make:htlatex()
    Make:match("html$",process) 

In this case, spurious span elements are joined, ligatures are decomposed,
and then all letters 'a' are replaced with 'z' letters.

## Image conversion

It is possible to convert parts of LaTeX input to pictures, it is used 
for example for math or diagrams in `tex4ht`. 

These pictures are stored in special `dvi` file, on which `dvi to image` 
commands are called. 

This conversion is normally configured in the `env file`, 
which is system dependent and which has a bit unintuitive syntax.
This configuration is processed by `t4ht` application and conversion
commands are called for all pictures.

It is possible to disable `t4ht` image processing and configure image 
conversion in the make file:

    Make:image("png$",
    "dvipng -bg Transparent -T tight -o ${output}  -pp ${page} ${source}")                                                       


`Make:image` takes two parameters, pattern to match image name and action.
Action can be either string template with conversion command, 
or function which takes table with parameters as argument.

There are three parameters:

  - output - output image file name
  - source - `dvi` file with the pictures
  - page   - page number of the converted image

## The `mode` variable

There is global `mode` variable available in the build file. It contains
contents of `--mode` command line option.  It can be used to run some commands
conditionally. For example:

     if mode == "draft" then
       Make:htlatex{} 
     else
       Make:htlatex{}
       Make:htlatex{}
       Make:htlatex{}
     end

In this example (which is the default configuration used by `make4ht`),
LaTeX is called only once when `make4ht` is called with `draft` mode:
    
    make4ht -m draft filename

## The `settings` table

You may want to access to the parameters also outside commands, file matches
and image conversion functions. For example, if you want to convert your file to
the `OpenDocument Format`, you can use the following settings, based on `oolatex`
command:

    settings.tex4ht_sty_par = settings.tex4ht_sty_par ..",ooffice"
    settings.tex4ht_par = settings.tex4ht_par .. " ooffice/! -cmozhtf"
    settings.t4ht_par = settings.t4ht_par .. " -cooxtpipes -coo "


# Command line options

    make4ht - build system for tex4ht
    Usage:
    make4ht [options] filename ["tex4ht.sty op." "tex4ht op." 
         "t4ht op" "latex op"]
    -b,--backend (default tex4ht) Backend used for xml generation.
         possible values: tex4ht or lua4ht
    -c,--config (default xhtml) Custom config file
    -d,--output-dir (default "")  Output directory
    -e,--build-file (default nil)  If build file name is different 
         than `filename`.mk4
    -f,--format  (default nil)  Output file format
    -l,--lua  Use lualatex for document compilation
    -m,--mode (default default) Switch which can be used in the makefile
    -n,--no-tex4ht  Disable dvi file processing with tex4ht command
    -s,--shell-escape Enables running external programs from LaTeX
    -u,--utf8  For output documents in utf8 encoding
    -v,--version  Print version number
    -x,--xetex Use xelatex for document compilation
    <filename> (string) Input file name


You can still use `make4ht` in same way as `htlatex`

    make4ht filename "customcfg, charset=utf-8" "-cunihtf -utf8" "-dfoo"

Note that this will not use `make4ht` routines for output directory making and
copying. If you want to use them, change the line above to:

    make4ht -d foo filename "customcfg, charset=utf-8" "-cunihtf -utf8"  

This call has the same effect as the following:

    make4ht -u -c customcfg -d foo filename

Output directory doesn't have to exist, it will be created automatically. 
Specified path can be relative to current directory, or absolute:

    make4ht -d use/current/dir/ filename
    make4ht -d ../gotoparrentdir filename
    make4ht -d ~/gotohomedir filename
    make4ht -d c:\documents\windowspathsareworkingtoo filename

# Troubleshooting 

## Incorrect handling of command line arguments for `tex4ht`, `t4ht` or `latex`

Sometimes, you may get a similar error:

> make4ht:unrecognized parameter: i

It may be caused by a following `make4ht` invocations:

     make4ht hello.tex "customcfg,charset=utf-8" "-cunihtf -utf8" -d foo

The command line option parser is confused by mixing options for `make4ht` and
`tex4ht` in this case and tries to interpret the `-cunihtf -utf8`, which are
options for `tex4ht` command as `make4ht` options. To fix that, you can either
move either move the `-d foo` directly after `make4ht` command:

     make4ht -d foo hello.tex "customcfg,charset=utf-8" "-cunihtf -utf8" 

Or, you can add space before `tex4ht` options:

     make4ht hello.tex "customcfg,charset=utf-8" " -cunihtf -utf8" -d foo

The former way is preferable, though.

