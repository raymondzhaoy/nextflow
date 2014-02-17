.. _process-page:

************
Processes
************

In Nextflow a `process` is the basic processing `primitive` to execute a user script.

The process definition starts with keyword the ``process``, followed by process name and finally the process `body`
delimited by curly brackets. The process body must contain a string which represents the command or, more generally,
a script that is executed by it. A basic process looks like the following example::

  process sayHello {

      "echo 'Hello world!' > file"

  }


more specifically a process may contain five definition blocks, respectively: directives,
inputs, outputs, shares and finally the process script. The syntax is defined as follows:

::

  process < name > {

     [ directives ]

     input:
      < process inputs >

     output:
      < process outputs >

     share:
      < shared input/output >

     [script:|exec:]
     < user script to be executed >

  }



Script
=======

The `script` block is a string statement that defines the command that is executed by the process to carry out its task.

A process contains one and only one script block, and it must be the last statement when the process contains
input, output and share declarations.

The entered string is executed as a `BASH <http://en.wikipedia.org/wiki/Bash_(Unix_shell)>`_ script in the
`host` system. It can be any command, script or combination of them, that you would normally use in terminal shell
or in a common BASH script.

The only limitation to the commands that can be used in the script statement is given by the availability of those
programs in the target execution system.


The script block can be a simple string or multi-line string. The latter simplifies the writing of non trivial scripts
composed by multiple commands spanning over multiple lines. For example::

    process doMoreThings {

      """
      blastp -db $db -query query.fa -outfmt 6 > blast_result
      cat blast_result | head -n 10 | cut -f 2 > top_hits
      blastdbcmd -db $db -entry_batch top_hits > sequences
      """

    }

As explained in the script tutorial section, strings can be defined by using a single-quote
or a double-quote, and multi-line strings are defined by three single-quote or three double-quote characters.

There is a subtle but important difference between them. Like in BASH, strings delimited by a ``"`` character support
variable substitutions, while strings delimited by ``'`` do not.

In the above code fragment the ``$db`` variable is replaced by the actual value defined somewhere in the
pipeline script.

.. warning:: Since Nextflow uses the same BASH syntax for variable substitutions in strings, you need to manage them
  carefully depending on if you want to evaluate a variable in the Nextflow context - or - in the BASH environment execution.

When you need to access a system environment variable  in your script you have two options. The first choice is as
easy as defining your script block by using a single-quote string. For example::

    process printPath {

       '''
       echo The path is: $PATH
       '''

    }

The drawback of this solution is that you will not able to access variables defined in the pipeline script context,
in your script block.

To fix this, define your script by using a double-quote string and `escape` the system environment variables by
prefixing them with a back-slash ``\`` character, as shown in the following example::


    process doOtherThings {

      """
      blastp -db \$DB -query query.fa -outfmt 6 > blast_result
      cat blast_result | head -n $MAX | cut -f 2 > top_hits
      blastdbcmd -db \$DB -entry_batch top_hits > sequences
      """

    }

In this example the ``$MAX`` variable has to be defined somewhere before, in the pipeline script.
`Nextflow` replaces it with the actual value before executing the script. Instead, the ``$DB`` variable
must exist in the script execution environment and the BASH interpreter will replace it with the actual value.


Scripts `à la carte`
--------------------

The process script is interpreted by Nextflow as a BASH script by default, but you are not limited to it.

You can use your favourite scripting language (e.g. Perl, Python, Ruby, R, etc), or even mix them in the same pipeline.

A pipeline may be composed by processes that execute very different tasks. Using `Nextflow` you can choose the scripting
language that better fits the task carried out by a specified process. For example for some processes `R` could be
more useful than `Perl`, in other you may need to use `Python` because it provides better access to a library or an API, etc.

To use a scripting other than BASH, simply start your process script with the corresponding
`shebang <http://en.wikipedia.org/wiki/Shebang_(Unix)>`_ declaration. For example::

    process perlStuff {

        """
        #!/usr/bin/perl

        print 'Hi there!' . '\n';
        """

    }

    process pyStuff {

        """
        #!/usr/bin/python

        x = 'Hello'
        y = 'world!'
        print "%s - %s" % (x,y)
        """

    }


.. tip:: Since the actual location of the interpreter binary file can change across platforms, to make your scripts
   more portable it is wise to use the ``env`` shell command followed by the interpreter's name, instead of the absolute
   path of it. Thus, the `shebang` declaration for a Perl script, for example,
   would look like: ``#!/usr/bin/env perl`` instead of the one in the above pipeline fragment.


Conditional scripts
-------------------

Complex process scripts may need to evaluate conditions on the input parameters or use traditional flow control
statements (i.e. ``if``, ``switch``, etc) in order to execute specific script commands, depending on the current
inputs configuration.

Process scripts can contain conditional statements by simply prefixing the script block with the keyword ``script:``.
By doing that the interpreter will evaluate all the following statements as a code block that must return the
script string to be executed. It's much easier to use than to explain, for example::


    seq_to_align = ...
    mode = 'tcoffee'

    process align {
        input:
        file seq_to_aln from sequences

        script:
        if( mode == 'tcoffee' )
            """
            t_coffee -in $seq_to_aln > out_file
            """

        else if( mode == 'mafft' )
            """
            mafft --anysymbol --parttree --quiet $seq_to_aln > out_file
            """

        else if( mode == 'clustalo' )
            """
            clustalo -i $seq_to_aln -o out_file
            """

        else
            error "Invalid alignment mode: ${mode}"

    }


In the above example the process will execute the script fragment depending on the value of the ``mode`` parameter.
By default it will execute the ``tcoffee`` command, changing the ``mode`` variable to ``mafft`` or ``clustalo`` value,
the other branches will be executed.


Native execution
------------------

Nextflow processes can execute native code other than system scripts as shown in the previous paragraphs.

This means that instead of specifying the process command to be executed as a string script, you can
define it by providing one or more language statements, as you would do in the rest of the pipeline script.
Simply starting the script definition block with the ``exec:`` keyword, for example::

    x = Channel.from( 'a', 'b', 'c')

    process simpleSum {
        input:
        val x

        exec:
        println "Hello Mr. $x"
    }

Will display::

    Hello Mr. b
    Hello Mr. a
    Hello Mr. c


.. warning:: Native processes execution is an incubating feature and has the following limitations:
    they can only be executed by using a `local` executor and they cannot be used when the `merge`
    feature is specified.


Inputs
=======

Nextflow processes are isolated from each other but can communicate between themselves sending values through channels.

The `input` block defines which channels the process is expecting to receive inputs data from. You can only define one
input block at a time and it must contain one or more inputs declarations.

The input block follows the syntax shown below::

    input:
      <input classifier> <input name> [from <source channel>] [attributes]


An input definition starts with an input `classifier` and the input `name`, followed by the keyword ``from`` and
the actual channel over which inputs are received. Finally some input optional attributes can be specified.

.. note:: When the input name is the same as the channel name, the ``from`` part of the declaration can be omitted.

The input classifier declares the `type` of data to be received. This information is used by Nextflow to apply the
semantic rules associated to each classifier and handle it properly depending on the target execution platform
(grid, cloud, etc).

The classifiers available are the ones listed in the following table:

=========== =============
Classifier  Semantic
=========== =============
val         Lets you access the received input value by its name in the process script.
env         Lets you use the received value to set an environment variable named
            as the specified input name.
file        Lets you handle the received value as a file, staging it properly in the execution context.
stdin       Lets you forward the received value to the process `stdin` special file.
set         Lets you handle a group of input values having one of the above classifiers.
each        Lets you execute the process for each entry in the input collection.
=========== =============


Input of generic values
-------------------------

The ``val`` classifier allows you to receive data of any type as input. It can be accessed in the process script
by using the specified input name, as shown in the following example::

    num = Channel.from( 1, 2, 3 )

    process basicExample {
      input:
      val x from num

      "echo process job $x"

    }


In the above example the process is executed three times, each time a value is received from the channel ``num``
and used to process the script. Thus, it results in an output similar to the one shown below::

    process job 3
    process job 1
    process job 2

.. note:: The `channel` guarantees that items are delivered in the same order as they have been sent - but -
  since the process is executed in a parallel manner, there is no guarantee that they are processed in the
  same order as they are received. In fact, in the above example, value ``3`` is processed before the others.


When the ``val`` has the same name as the channel from where the data is received, the ``from`` part can be omitted.
Thus the above example can be written as shown below::

    num = Channel.from( 1, 2, 3 )

    process basicExample {
      input:
      val num

      "echo process job $num"

    }


Input of files
-----------------

The ``file`` classifier allows you to receive a value as a file in the process execution context. This means that
Nextflow will stage it in the process execution directory, and you can access it in the script by using the name
specified in the input declaration. For example::

    proteins = Channel.path( '/some/path/*.fa' )

    process blastThemAll {
      input:
      file query_file from proteins

      "blastp -query ${query_file} -db nr"

    }

In the above example all the files ending with the suffix ``.fa`` are sent over the channel ``proteins``.
Then, these files are received by the process which will execute a `BLAST` query on each of them.

When the file input name is the same as the channel name, the ``from`` part of the input declaration can be omitted.
Thus, the above example could be written as shown below::

    proteins = Channel.path( '/some/path/*.fa' )

    process blastThemAll {
      input:
      file proteins

      "blastp -query $proteins -db nr"

    }


It's worth noting that in the above examples, the name of the file in the file-system is not touched, you can
access the file even without knowing its name because you can reference it in the process script by using the
variable whose name is specified in the input file parameter declaration.

There may be cases where your task needs to use a file whose name is fixed, it does not have to change along
with the actual provided file. In this case you can specify its name by specifying the ``name`` attribute in the
input file parameter declaration, as shown in the following example::

    input:
        file query_file name 'query.fa' from proteins


Or alternatively using a shorter syntax::

    input:
        file 'query.fa' from proteins


Using this, the previous example can be re-written as shown below::

    proteins = Channel.path( '/some/path/*.fa' )

    process blastThemAll {
      input:
      file 'query.fa' from proteins

      "blastp -query query.fa -db nr"

    }


What happens in this example is that each file, that the process receives, is staged with the name ``query.fa``
in a different execution context (i.e. the folder where the job is executed) and an independent process
execution is launched.

.. tip:: This allows you to execute the process command various time without worrying the files names changing.
  In other words, `Nextflow` helps you write pipeline tasks that are self-contained and decoupled by the execution
  environment. This is also the reason why you should avoid whenever possible to use absolute or relative paths
  referencing files in your pipeline processes.


.. TODO describe that file can handle channels containing any data type not only file


Multiple input files
----------------------

A process can declare as input file a channel that emits a collection of values, instead of a simple value.

In this case, the script variable defined by the input file parameter will hold a list of files. You can
use it as shown before, referring to all the files in the list, or by accessing a specific entry using the
usual square brackets notation.

When a target file name is defined in the input parameter and a collection of files is received by the process,
the file name will be appended by a numerical suffix representing its ordinal position in the list. For example::

    fasta = Channel.path( "/some/path/*.fa" ).buffer(count:3)

    process blastThemAll {
        input:
        file 'seq' from fasta

        "echo seq*"

    }

Will output::

    seq1 seq2 seq3
    seq1 seq2 seq3
    ...

The target input file name can contain the ``*`` and ``?`` wildcards, that can be used
to control the name of staged files. The following table shows how the wildcards are
replaced depending on the cardinality of the received input collection.

============ ============== ==================================================
Cardinality   Name pattern     Staged file names
============ ============== ==================================================
 1           ``file*.ext``   ``file.ext``
 1           ``file?.ext``   ``file1.ext``
 1           ``file??.ext``  ``file01.ext``
 many        ``file*.ext``   ``file1.ext``, ``file2.ext``, ``file3.ext``, ..
 many        ``file?.ext``   ``file1.ext``, ``file2.ext``, ``file3.ext``, ..
 many        ``file??.ext``  ``file01.ext``, ``file02.ext``, ``file03.ext``, ..
============ ============== ==================================================

The following fragment shows how a wildcard can be used in the input file declaration::


    fasta = Channel.path( "/some/path/*.fa" ).buffer(count:3)

    process blastThemAll {
        input:
        file 'seq?.fa' from fasta

        "cat seq1.fa seq2.fa seq3.fa"

    }


Parametric input file names
----------------------------

When the input file name is specified by using the ``name`` file clause or the short `string` notation, you
are allowed to use other input values as variables in the file name string. For example::


  process simpleCount {
    input:
    val x from species
    file "${x}.fa" from genomes

    """
    cat ${x}.fa | grep '>'
    """
  }


In the above example, the file name of the input file is set by using the current value of the ``x`` input value.

This allows you to stage the input files in the script working directory with a name that is coherent
with the current execution context.


Input of type 'stdin'
-----------------------

The ``stdin`` input classifier allows you the forwarding of the value received from a channel to the
`standard input <http://en.wikipedia.org/wiki/Standard_streams#Standard_input_.28stdin.29>`_
of the command executed by the process. For example::

    str = Channel.from('hello', 'hola', 'bonjour', 'ciao').map { it+'\n' }

    process printAll {
       input:
       stdin str

       """
       cat -
       """

    }

It will output::

    hola
    bonjour
    ciao
    hello




Input of type 'env'
---------------------

The ``env`` classifier allows you to define an environment variable in the process execution context based
on the value received from the channel. For example::

    str = Channel.from('hello', 'hola', 'bonjour', 'ciao')

    process printEnv {

        input:
        env HELLO from str

        '''
        echo $HELLO world!
        '''

    }

::

    hello world!
    ciao world!
    bonjour world!
    hola world!



Input of type 'set'
--------------------

The ``set`` classifier allows you to group multiple parameters in a single parameter definition. It can be useful
when a process receives, in input, tuples of values that need to be handled separately. Each element in the tuple
is associated to a corresponding element with the ``set`` definition. For example::

     tuple = Channel.from( [1, 'alpha'], [2, 'beta'], [3, 'delta'] )

     process setExample {
         input:
         set val(x), file('latin.txt')  from tuple

         """
         echo Processing $x
         cat - latin.txt > copy
         """

     }


In the above example the ``set`` parameter is used to define the value ``x`` and the file ``latin.txt``,
which will receive a value from the same channel.

In the ``set`` declaration items can be defined by using the following classifiers: ``val``, ``env``, ``file`` and ``stdin``.

A shorter notation can be used by applying the following substitution rules:

============== =======
long            short
============== =======
val(x)          x
file(x)         (not supported)
file('name')    'name'
file(x:'name')  x:'name'
stdin           '-'
env(x)          (not supported)
============== =======

Thus the previous example could be rewritten as follows::

      tuple = Channel.from( [1, 'alpha'], [2, 'beta'], [3, 'delta'] )

      process setExample {
          input:
          set x, 'latin.txt' from tuple

          """
          echo Processing $x
          cat - latin.txt > copy
          """

      }

File names can contain parametric values as explained in the `Parametric input file names`_ section.


Input repeaters
----------------

The ``each`` classifier allows you to repeat the execution of a process for each item in a collection,
every time new data is received. For example::

  sequences = Channel.path('*.fa')
  methods = ['regular', 'expresso', 'psicoffee']

  process alignSequences {
    input:
    file seq from sequences
    each mode from methods

    """
    t_coffee -in $seq -mode $mode > result
    """

  }


In the above example every time a file of sequences is received as input by the process,
it executes three T-coffee tasks, using a different value for the ``mode`` parameter.

This is useful when you need to `repeat` the same task for a given set of parameters.

.. note:: When multiple repeaters are declared, the process is executed for each *combination* them.

Take in consideration the following example. The process declares, in input, a channel receiving a
generic ``shape`` of values. Each time a new shape value is received, it `draws` it
in two different colors and three different sizes::

    shapes = Channel.from('circle','square', 'triangle' .. )

    process combine {
      input:
      val shape from shapes
      each color from 'red','blue'
      each size from 1,2

      "echo draw $shape $color with size: $size"

    }

Will output::

    draw circle red with size: 1
    draw circle red with size: 2
    draw circle red with size: 3
    draw circle blue with size: 1
    draw circle blue with size: 2
    draw circle blue with size: 3
    draw square red with size: 1
    draw square red with size: 2
    draw square red with size: 3
    draw square blue with size: 1
    draw square blue with size: 2
    draw square blue with size: 3
    draw triangle red with size: 1
    draw triangle red with size: 2
    draw triangle red with size: 3
    draw triangle blue with size: 1
    draw triangle blue with size: 2
    draw triangle blue with size: 3
    ..


Outputs
========

The `output` declaration block allows to define the channels used by the process to send out the results produced.

It can be defined at most one output block and it can contain one or more outputs declarations.
The output block follows the syntax shown below::

    output:
      <output classifier> <output name> [into <target channel>] [attribute [,..]]

Output definitions start by an output `classifier` and the output `name`, followed by the keyword ``into`` and
the actual channel over which outputs are sent. Finally some optional attributes can be specified.

.. note:: When the output name is the same as the channel name, the ``into`` part of the declaration can be omitted.


.. TODO the channel is implicitly create if does not exist

The provided classifiers are:

- *val*: handle data of any type;
- *file*: the output is managed as file to be staged in the process context;
- *stdout*: the received data is redirected to the process `stdout` special file;


The classifiers that can be used in the output declaration block are the ones listed in the following table:

=========== =============
Classifier  Semantic
=========== =============
val         Sends variable's with the name specified over the output channel.
file        Sends a file produced by the process with the name specified over the output channel.
stdout      Sends the executed process `stdout` over the output channel.
set         Lets to send multiple values over the same output channel.
=========== =============


Output values
-------------------------

The ``val`` classifier allows to output a `value` defined in the script context. In a common usage scenario,
this is a value which has been defined in the `input` declaration block, as shown in the following example::

   methods = ['prot','dna', 'rna']

   process anyValue {
     input:
     val x from methods

     output:
     val x into receiver

     "echo $x > file"

   }

   receiver.subscribe { println "Received: $it" }


Output files
-----------------

The ``file`` classifier allows to output one or more files, produced by the process, over the specified channel.
For example::


    process randomNum {

       output:
       file 'result.txt' into numbers

       '''
       echo $RANDOM > result
       '''

    }

    numbers.subscribe { println "Received: " + it.text }


In the above example the process, when executed, creates a file named ``result.txt`` containing a random number.
Since a file parameter using the same name is declared between the outputs, when the task is completed that
file is sent over the ``numbers`` channel. A downstream `process` declaring the same channel as `input` will
be able to receive it.

.. note:: If the channel specified as output has not been previously declared in the pipeline script, it
  will implicitly created by the output declaration itself.


.. TODO explain Path object

Multiple output files
-----------------------

When declaring an output file it is possible to specify its name using the usual Linux wildcards characters ``?`` and ``*``.
This allows to output all the files matching the specified file name pattern as single item. For example::

    process splitLetters {

        output:
        file 'chunk_*' into letters

        '''
        printf 'Hola' | split -b 1 - chunk_
        '''
    }

    letters.subscribe { println it *.text }

::

    [H, o, l, a]


.. TODO Advanced file output

Parametric output file names
-----------------------------

In the output file name you can use values defined in the input declaration block as variables. For example::


  process align {
    input:
    val x from species
    file seq from sequences

    output:
    file "${x}.aln" into genomes

    """
    t_coffee -in $seq > ${x}.aln
    """
  }

In the above example, each time the process is executed an alignment file is produced whose name depends
by the actual value of the ``x`` input.


Output 'stdout' special file
-------------------------------

The ``stdout`` classifier allows to `capture` the `stdout` output of the executed process and send it over
the channel specified in the output parameter declaration. For example::

    process echoSomething {
        output:
        stdout channel

        "echo Hello world!"

    }

    channel.subscribe { print "I say..  $it" }





Output 'set' of values
--------------------------

The ``set`` classifier allows to send multiple values into a single channel. This feature is useful
when you need to `together` the result of multiple execution of the same process, as shown in the following
example::

    query = Channel.path '*.fa'
    specie = Channel.from 'human', 'cow', 'horse'

    process blast {

    input:
        val specie
        file query

    output:
        set val(specie), file('result') into blastOuts


    "blast -db nr -query $query" > result

    }


In the above example a `BLAST` task is executed for each pair of ``specie`` and ``query`` that are received.
When the task completes a new tuple containing the value for ``specie`` and the file ``result`` is sent to the ``blastOuts`` channel.


A `set` declaration can contain any combination of the following classifiers, previously described: ``val``, ``file`` and ``stdout``.

.. tip:: Variable identifiers are interpreted as `values` while strings literals are interpreted as `files` by default,
  thus the above output `set` can be rewritten using a short notation as shown below.


::

    output:
        set specie, 'result' into blastOuts



File names can contain parametric values as explained in the `Parametric output file names`_ section.

Shares
=======

Share declarations are a special type of process parameter that can act as an `input` and `output` parameter at the same time.

The share block is declared by using the syntax shown below::

  share:
    <share classifier> <parameter name> [from <source channel>] [into <target channel>] [attributes]


A share definition begins with the share `classifier` followed by the parameter `name`. Optionally the keyword ``from`` 
can be used to specify the channel over which data is received, and the keyword ``into``, followed by a channel name, 
can be used to specify the channel where the produced data has to be sent.

Share parameters accept only the classifiers listed in the following table:

=========== =============
Classifier  Semantic
=========== =============
val         Lets you access the input value in the process script and/or to send it over the output channel.
file        Lets you handle the input value as a file, staging it properly in the execution context and/or send it as result over the output channel.
=========== =============


Share parameters have some important differences compared to input or output parameters:

* A share parameter can only receive a single input value, which is bound in the script evaluation context before the process first execution.
* If a share parameter declares an output channel it emits exactly one value, after the process last execution.
* It allows you to `share` the parameter's value across multiple process executions.
* Whenever a share parameter is declared, the process is executed serially, instead of in a parallel manner.

.. warning:: Share parameters cannot be used in a process that uses the `merge` feature.


Share generic values
---------------------

The share ``val`` classifier allows you to declare a parameter whose value can be accessed, 
in the script context, across multiple executions of the same process. For example::

    process printCount {
      input:
      val cheers from 'Bonjour', 'Ciao', 'Hello', 'Hola'

      share:
      val count from 1 into result

      script:
      count += count
      "echo $cheers world! "

    }


    result.subscribe  { println "Result = $it" }

Will output::

    Result = 16


This example shows how the `state` of the ``count`` variable is maintained throughout the process executions, and
how on process termination the final value is sent over the channel declared by the ``into`` keyword.

Some caveats about shared value parameters:

* The ``from`` declaration can be used to initialise the parameter by specifying a variable defined in the 
  pipeline script or by using a `literal` value (as in the above example).

* When the ``from`` declaration is omitted, the parameter is initialised to the variable's value in the script scope
  having the same name as the parameter.

* When a variable with the parameter's name doesn't exist in the script scope and no ``from`` is specified,
  the parameter is initialised to ``null``.



Share file
------------

The share ``file`` classifier allows you to declare a parameter that shares its state using a file. For example::

  process saveHello {
      input:
      val cheers from 'Bonjour', 'Ciao', 'Hello', 'Hola'

      share:
      file greetings into result

      "echo '$cheers' >> $greetings "

    }

    result.subscribe { println it.text  }


It will print::

    Bonjour
    Ciao
    Hello
    Hola


This example shows how the file content is shared through the process executions. When the process 
completes, the file is sent over the channel declared as output.


.. _process-directives:

Directives
==========

Using the `directive` declarations block you can provide optional settings that will affect the execution of the current
process.

They must be entered at the top of the process `body`, before any other declaration blocks (i.e. ``input``, ``output``, etc) 
and have the following syntax::

    name value [, value2 [,..]]

Some directives are generally available to all processes, some others depends on the `executor` currently defined.

The directives are:

* `cache`_
* `echo`_
* `errorStrategy`_
* `executor`_
* `storeDir`_
* `validExitStatus`_


cache
---------

The ``cache`` directive allows you to store the process results to a local cache. Any following attempt to execute
the process along with the same inputs will cause the process execution to be skipped, producing the stored data as
the actual results.

The caching feature generates a unique `key` by indexing the process script and inputs. This key is used
identify univocally the outputs produced by the process execution.


The cache is enabled by default, you can disable it for a specific process by setting the ``cache``
directive to ``false``. For example:: 

  process noCacheThis {
    cache false

    script:
    <your command string here>
  }

The ``cache`` directive possible values are shown in the following table:

===================== =================
Value                 Description
===================== =================
``false``             Disable cache feature.
``true`` (default)    Cache process outputs. Input files are indexed by using the meta-data information (name, size and last update timestamp).
``'deep'``            Cache process outputs. Input files are indexed by their content.
===================== =================


echo
-----

By default the `stdout` produced by the commands executed in all processes is ignored.
Setting the ``echo`` directive to ``true`` you can forward the process `stdout` to the current top
running process `stdout` file, showing it in the shell terminal.

For example::

    process sayHello {
      echo true

      script:
      "echo Hello"
    }

::

    Hello

Without specifying ``echo true`` you won't see the ``Hello`` string printed out when executing the above example.


errorStrategy
--------------

The ``errorStrategy`` directive defines how an error condition is managed by the process. By default when an error status
is returned by the executed script, the process stops immediately. This in turn forces the entire pipeline to stop.

When setting the ``errorStrategy`` directive to ``ignore`` the process doesn't stop on an error condition,
it just reports a message notifying you of the error event.

For example::

    process ignoreAnyError {
       errorStrategy 'ignore'

       script:
       <your command string here>
    }

.. tip:: By definition a command script fails when it ends with a non-zero exit status. To change this behavior
  see `validExitStatus`_.


executor
---------

The `executor` defines the underlying system where processes are executed. By default a process uses the executor
defined globally in the ``nextflow.config`` file.

The ``executor`` directive allows you to configure what executor has to be used by the process, overriding the default
configuration. The following values can be used:

============== ==================
Name            Executor
============== ==================
``local``      The process is executed in the computer where `Nextflow` is launched.
``sge``        The process is executed using a Sun Grid Engine / `Open Grid Engine <http://gridscheduler.sourceforge.net/>`_.
``lsf``        The process is executed via the `Platform LSF <http://en.wikipedia.org/wiki/Platform_LSF>`_ job scheduler.
``slurm``      The process is executed via the SLURM job scheduler.
``dnanexus``   The process is executed into the `DNAnexus <http://www.dnanexus.com/>`_ cloud.
============== ==================

The following example shows how to set the process's executor::


   process doSomething {

      executor 'sge'

      script:
      <your script here>

   }


.. note:: Each executor provides its own set of configuration options that can set be in the `directive` declarations block.
   See :ref:`executor-page` section to read about specific executor directives.


storeDir
---------

The ``storeDir`` directive allows you to define a directory that is used as `permanent` cache for your process results.

In more detail, it affects the process execution in two main ways:

#. The process is executed only if the files declared in the `output` clause do not exists in directory specified by
   the ``storeDir`` directive. When the files exist the process execution is skipped and these files are used as
   the actual process result.

#. Whenever a process complete successfully the files listed in the `output` declaration block are copied in the directory
   specified by the ``storeDir`` directive.

The following example shows how use the ``storeDir`` directive to create a directory containing a BLAST database
for each specie specified by an input parameter::

  genomes = Channel.path(params.genomes)

  process formatBlastDatabases {

    storeDir '/db/genomes'

    input:
    file specie from genomes

    output:
    file "${dbName}.*" into blastDb

    script:
    dbName = specie.baseName
    """
    makeblastdb -dbtype nucl -in ${specie} -out ${dbName}
    """

  }



validExitStatus
-------------------

A process is terminated when the executed command returns an error exit status. By default any error status
other than ``0`` is interpreted as an error condition.

The ``validExitStatus`` directive allows you to fine control which error status will represent a successful command execution.
You can specify a single value or multiple values as shown in the following example::


    process returnOk {
        validExitStatus 0,1,2

         script:
         """
         echo Hello
         exit 1
         """
    }


In the above example, although the command script ends with a ``1`` exit status, the process
will not return an error condition because the value ``1`` is declared as a `valid` status in 
the ``validExitStatus`` directive.
