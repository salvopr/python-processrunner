Python ProcessRunner
====================
Designed to make reading from external processes easier. Does not permit interactive communication with the process, but rather provides (usually) simplified consumption of process output, especially targeted at logging use cases.

Output can be sent to multiple locations. E.g. the stdout and stderr of an external process can be written to one or multiple files, AND also to the pipes of the local process.

Several convenience functions simplify common use cases. All the classes and functions take the command to execute in the subprocess.Popen/.call format, a list of strings starting with the command name, followed by any arguments for that command.


Provided classes
================
ProcessRunner
  The ProcessRunner class uses subprocess.Popen. It does not use the ``shell=True`` flag. All processes started by the class are saved in ``PROCESSRUNNER_PROCESSES``. A list of currently active processes started by the class can be retrieved by calling ``getActiveProcesses()``, which IS NOT a class member.

  *Parameters*
    - **command** REQUIRED ``[string,]`` The command to execute along with all parameters.
    - **cwd** OPTIONAL ``string`` Directory to switch into before execution. CWD does not apply to the location of the process executable itself, which must be on PATH or called with an absolute path.

  *Properties*
    - Non-blocking: Returns immediately; the external process is managed in a new thread

ProcessRunner class methods
---------------------------
collectLines
  Obtains all output from stdout, stderr, or both in a list.

  *Parameters*
    - **procPipeName** OPTIONAL ``string`` One of stdout or stderr. If neither is provided, the output of both pipes will be collected.

  *Properties*
    - Blocking: Returns once the external command exits.

getCommand
  Returns the ``[string,]`` originally provided as the ``command`` parameter to ``ProcessRunner``.

  *Parameters*
    - None

  *Properties*
    - None

getLineFromPipe
  Obtain one line from a pipe client.

  *Parameters*
    - **clientId** REQUIRED ``string`` Pipe manager client queue ID.
    - **procPipeName** REQUIRED ``string`` One of stdout or stderr.

  *Properties*
    - Blocking

getPopen
  Obtain the underlying ``subprocess.Popen`` instance.

  *Parameters*
    - None

  *Properties*
    - None

isAlive
  Returns ``True`` if the external process is still running, else retuns ``False``.

  *Parameters*
    - None

  *Properties*
    - None

isQueueEmpty
  Returns ``True`` if there are no lines in the queue for a given pipe client, otherwise returns ``False``.

  *Parameters*
    - **clientId** REQUIRED ``string`` Pipe manager client queue ID.
    - **procPipeName** REQUIRED ``string`` One of stdout or stderr.

  *Properties*

mapLines
  Run a function against each line presented by one pipe manager.
  Returns a reference to a ``dict`` that can be used to monitor the status of
  the function. When the process is dead, the queues are empty, and all lines
  are processed, the dict will be updated (the value will be changed from ``False`` to ``True``). This can be used as a blocking
  mechanism by functions invoking mapLines.
  Status dict format: ``{"complete":bool}``

  *Parameters*
    - **func** REQUIRED ``function`` A function that takes one parameter, the line from the pipe and returns the line with any desired changes.
    - **procPipeName** REQUIRED ``string`` One of "stdout" or "stderr".

  *Properties*
    - Non-blocking: Returns immediately.
    - Returns a dict reference that is used to determine completion.

poll
  A proxy to the underlying ``subprocess.Popen.poll`` method. Returns ``NoneType`` if the external process is alive, otherwise the exit code as an ``int``.

  *Parameters*
    - None

  *Properties*
    - None

registerForClientQueue
  Register to get a client queue on a pipe manager. The ID for the queue is
  returned from the method as a string.

  *Parameters*
    - **procPipeName** REQUIRED ``string`` One of "stdout" or "stderr".

  *Properties*
    - None

unRegisterClientQueue
  Unregister a client queue from a pipe manager. Prevents clients from waiting on other clients that will never perform additional reads.

  *Parameters*
    - **procPipeName** REQUIRED ``string`` One of "stdout" or "stderr".
    - **clientId** REQUIRED ``string`` ID of the client queue on this pipe manager.

  *Properties*
    - None

wait
  Block until the external process exits and pipe managers have finished reading from the external pipes.

  *Parameters*
    - None

  *Properties*
    - Chainable

which
  Verify a given command exists. Returns absolute path to exec as a string, or None if not found.

  *Parameters*
    - **program** REQUIRED ``string`` The name or full path to desired executable.

  *Properties*
    - Static


Provided convenience functions
==============================
runCommand
  The runCommand function returns the process exit code, and stdout and stderr are connected to local stdout and stderr.

  *Parameters*
    - **command** REQUIRED ``[string,]`` The command to execute along with all parameters.
    - **outputPrefix** OPTIONAL ``string`` String to prepend to all output lines. Defaults to 'ProcessRunner> '.

  *Properties*
    - Blocking: Returns once the external command exits.

ssh
  The ssh function runs a command on a remote host, and returns the SSH exit code. stdout and stderr are connected to local stdout and stderr.

  *Parameters*
    - **remoteAddress** REQUIRED ``string`` IP or hostname for target system.
    - **remotecommand** REQUIRED ``string`` The command to run on the target system.
    - **outputPrefix** OPTIONAL ``string`` String to prepend to all output lines. Defaults to 'ssh> '.

  *Properties*
    - Blocking: Returns once the external command exits.

WriteOut
  The WriteOut function is used to prepend lines from the external process with a given string. Given a pipe and a string, it returns a function that accepts a line of text, then writes that line to the provided pipe, prepended with a user provided string. Useful when handling output from processes directly. See example use below.

  *Parameters*
    - **pipe** REQUIRED ``pipe`` A system pipe to write the output to.
    - **outputPrefix** REQUIRED ``string`` A string to prepend to each line.
      - This can also be any object that can be cast to a string.

  *Properties*
    - Return type is a function.

getActiveProcesses
  The getActiveProcesses function returns a list of ``ProcessRunner`` instances that are currently alive.

  *Takes no parameters*


Custom Exceptions
=================
CommandNotFound
  Exception thrown when the command to execute isn't available.


Examples
==============

Simple
------
Use SCP to copy a local file to a remote host, using SSH key-based authentication.

::

  # Run a command, wait for it to complete, and gather its return code
  command = ["scp", "-o", "BatchMode=yes", "-o", "StrictHostKeyChecking=no", "/path/to/local/file", clientAddress+":/tmp/"]
  result = ProcessRunner(command).wait().poll()

Complex
-------
Execute a command and while it runs write lines from the external process stdout and stderr to both the corresponding local pipes, as well as corresponding files. Further, prefix the local pipe output with dedicated notes, and prefix the file output with timestamps.

::

  # Logging files
  stdoutFile = open(workingDir+'/stdout.txt', 'a')
  stderrFile = open(workingDir+'/stderr.txt', 'a')

  # Date/time notation for output lines in files
  class DateNote:
      def init(self):
          pass
      def __repr__(self):
          return datetime.now().isoformat() + " "

  # Start the process
  proc = ProcessRunner(command)

  # Attach output mechanisms to the process's output pipes. These are handled asynchronously, so you can see the output while it is happening
  # Write to the console's stdout and stderr, with custom prefixes for each
  proc.mapLines(WriteOut(pipe=sys.stdout, outputPrefix="validation-stdout> "), procPipeName="stdout")
  proc.mapLines(WriteOut(pipe=sys.stderr, outputPrefix="validation-stderr> "), procPipeName="stderr")

  # Write to the log files, prepending each line with a date/time stamp
  proc.mapLines(WriteOut(pipe=stdoutFile, outputPrefix=DateNote()), procPipeName="stdout")
  proc.mapLines(WriteOut(pipe=stderrFile, outputPrefix=DateNote()), procPipeName="stderr")

  # Block regular execution until the process finishes
  result = proc.wait().poll()

  # Wait until the queues are emptied to close the files
  while not proc.areAllQueuesEmpty():
      time.sleep(0.01)

  stdoutFile.close()
  stderrFile.close()
