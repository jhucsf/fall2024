---
layout: default
title: "Assignment 5: Key/Value Store"
---

*Note*: Assignment 5 is a double assignment. Each milestone (MS1 and MS2)
is worth 1/6 of the assignments grade for the course, the same as
(individually) Assignments 1–4.

**Due:**

* Milestone 1 due **Wednesday, Nov 20th** by 11 pm
* Milestone 2 due **Friday, December 6th** by 11 pm (**Note**: late hours may *not* be used)

<div class='admonition danger'>
  <div class='title'>Warning!</div>
  <div class='content' markdown='1'>
This is a very substantial two-part assignment! While it doesn't involve
writing huge amounts of code, you will need to think carefully
about what you are doing. We **strongly** recommend that you
get started early, ask questions early, and leave plenty of
time for testing and debugging.
  </div>
</div>

## Grading Criteria

Note that Milestone 1 and Milestone 2 are both out of 100% since each one
is weighted as a full assignment.

Milestone 1:

* `Message` and `MessageSerialization` functionality: 25%
* `Table` functionality: 20%
* `ValueStack` functionality: 5%
* Client program functionality (`get_value`, `set_value`, `incr_value`): 40%
* Design and coding style: 10%

Milestone 2:

* Server program functionality (auto-commit mode, error handling): 60%
* Server program functionality (explicit transactions): 20%
* Report on synchronization: 10%
* Design and coding style: 10%

## Getting Started

Download [`csf_assign05.zip`](csf_assign05.zip) and unzip it.

To build all of the programs (unit test, clients, and server):

```
make depend
make -j
```

### Suggested Approach

For Milestone 1:

1. Get all of the unit tests in `unit_tests.cpp` to pass. This will
   involve implementing classes and functions in
   * `message.h`, `message.cpp`
   * `message_serialization.h`, `message_serialization.cpp`
   * `table.h`, `table.cpp`
   * `value_stack.h`, `value_stack.cpp`
2. Implement the client programs (`get_value.cpp`, `set_value.cpp`, and
   `incr_value.cpp`.) You can test them against the reference server
   implementation included in the starter code (`ref_server`.) You can
   also compare their behavior to the reference implementations of the
   client programs (`ref_get_value`, `ref_set_value`, `ref_incr_value`.)

In completing item 1, you will build the functionality you need to
represent, encode, and decode messages (requests and responses), and
also you will have the underlying table and operand stack functionality
needed for the server in Milestone 2.

For Milestone 2:

1. Modify `server.cpp` and `server.h` so that the server
   can create a server socket, listen for incoming connections,
   and create a `ClientConnection` object, start a client thread for
   each client connection, and within the client thread, invoke the
   `chat_with_client()` member function on the object to carry out
   the communication with the client.
2. Initially, focus on getting "auto-commit" to work. Each client
   request will access (at most) a single table, and the client
   connection thread should hold the table's lock only for the
   duration of handling one request.
3. Finally, implement transactions. This will be the most challenging
   part of this milestone!

## Overview

In this assignment, you will implement client programs and a server program
for an in-memory key/value store slightly reminiscent of [Redis](https://redis.io/).

### Tables

The central concept in the key/value store is the *table*. The server maintains
some number of tables, each of which has a name. A table is a collection of
*tuples*. Each tuple consists of a *key* and a *value*. The key and the value
in a tuple are both strings. Two tuples in a table are not allowed to have the
same key (i.e., keys are unique within a table.)  Each key in a table must
be an *identifier*, meaning that there is a restriction on what strings
are considered valid keys. Each value in a table must be a sequence of
non-whitespace characters.

If you are thinking "a table sounds like a map of string to string", that is
precisely the right idea, and we recommend implementing tables using
STL `map` objects.

Here is an example a table called "Fruit":

Key        | Value
---------- | ----------
Apples     | 8
Lemons     | 12
Mangoes    | 2
Oranges    | 4
Persimmons | 19

### Committing and Rolling Back Changes

Rather than making direct changes to a table, a client *proposes* changes
to the table. Such proposed changes are "tentative".
For example, let's say that a client makes the following changes
to the "Fruit" table shown previously:

1. Change the value of "Lemons" to 5
2. Add a tuple "Avocados,13"

The table is now in the following state:

Key        | Value
---------- | ----------
Apples     | 8
*Avocados* | *13*
Lemons     | <strike>12</strike> *5*
Mangoes    | 2
Oranges    | 4
Persimmons | 19

The tentative (proposed) changes are shown using <strike>strikeout</strike>
(for overwritten values) and *italics* (for added or modified keys
or values.)

After proposing changes to a table, the client either *commits* the
changes or *rolls them back*. Committing the changes means that the
proposed changes become part of the "official" contents of the table.
Committing the proposed changes shown above would result in the state
of the "Fruit" table looking like this:

Key        | Value
---------- | ----------
Apples     | 8
Avocados   | 13
Lemons     | 5
Mangoes    | 2
Oranges    | 4
Persimmons | 19

Rolling back changes mean abandoning the proposed changes and reverting
the table to its previous contents:

Key        | Value
---------- | ----------
Apples     | 8
Lemons     | 12
Mangoes    | 2
Oranges    | 4
Persimmons | 19

### Network Protocol

Clients communicate with the server using a simple line-oriented network protocol.
Each message is encoded as a single line of text terminated with a newline
(`\n`) character.

A message consists of a *command* followed by zero or more *arguments*.
Tokens (command and argument) are separated by one or more space
(` `) characters. (Exception: a *quoted\_text* argument can have
spaces within the argument.)

The following table summarizes the messages.

Message               | Type     | Description
--------------------- | -------- | --------------------
LOGIN *username*      | Request  | Client logs in
CREATE *table*        | Request  | Ask server to create named table
PUSH *value*          | Request  | Push a value onto the operand stack
POP                   | Request  | Pop (discard) the top value from the operand stack
TOP                   | Request  | Retrieve the top value from the operand stack
SET *table* *key*     | Request  | Set value of tuple named by *key* in *table*<br>to the value popped from the operand stack
GET *table* *key*     | Request  | Push value of tuple named by *key* in *table*<br>onto the operand stack
ADD                   | Request  | Pop two integers from operand stack,<br>add them, push sum
MUL                   | Request  | Pop two integers from operand stack,<br>add them, push product
SUB                   | Request  | Pop right and left integers from operand<br>stack, subtract right from left, push difference
DIV                   | Request  | Pop right and left integers from operand<br>stack, divide left by right, push quotient
BEGIN                 | Request  | Client begins a transaction
COMMIT                | Request  | Client commits a transaction
BYE                   | Request  | Client logs out (end of connection)
OK                    | Response | Request was carried out
FAILED *quoted\_text* | Response | Request wasn't carried out, but connection continues<br>(recoverable error)
ERROR *quoted\_text*  | Response | Request wasn't carried out, and connection finishes<br>(fatal error)
DATA *value*          | Response | Response to the TOP command; *value* is the value<br>at the top of the operand stack

*Request* messages are sent from a client to the server, and *Response* messages are
sent from the server back to a client (to respond to a request.)

The arguments *username*, *table*, and *key* are all *identifiers*. An identifier
must begin with a letter (a-z or A-Z), and the rest of the identifier must consist of
letters, underscores (`_`), and/or digits (0-9).

A *value* argument is a sequence of 1 or more non-whitespace characters.

A *quoted\_text* argument begins with `"`, is followed by zero or more
non-`"` characters, and ends with a `"`.

Some examples of arguments (as they would appear in an encoded message):

Argument             | Examples
-------------------- | -----------
*username*           | `alice`, `bob`, `Cornelius`, `little_dorit`
*table*              | `Fruit`, `invoices`, `line_items`
*key*                | `Apples`, `foo_bar`
*value*              | `90125`, `invoice1824`
*quoted\_text*       | `"No such table"`, `"Could not acquire lock"`

The maximum number of characters in an encoded message, including the
terminating newline character, is 1024. This limit is represented by the
`Message::MAX_ENCODED_LEN` constant.

### Message Sequencing

A message sent by the client is a *request*.

A client sends a series of requests to the server. The first message sent by
the client must be a LOGIN message. The last message (assuming that the
client voluntarily chooses to end the conversation with the server)
should be the BYE message.

The server sends a single *response* message after processing each
client request message. The following table summarizes the meaning
of the response:

Response      | Description
------------- | ------------------
OK            | Success: the server carried out the request
FAILED        | The operation requested by the client failed, but the session can continue
ERROR         | The operation requested by the client failed or was invalid, and the session has ended
DATA          | Success: the specified value is the value at the top of the operand stack

For most operations, OK is the expected response.

The TOP request is the only one for which the DATA response is the expected one.
A TOP request allows the client to get the data value at the top of the
operand stack.

A FAILED response means that the server wasn't able to carry out the request,
but the session can continue (i.e., the client can send additional requests.)
A FAILED response is sent back if, as part of processing the request, an
`OperationException` or `FailedTransaction` exception is thrown. (See
the [Exceptions](#exceptions) section.)

An ERROR response means that the server wasn't able to carry out the request,
and the session has ended. The client should close the connect if it receives
this response. An ERROR response is sent if, as part of processing the request,
an `InvalidMessage` exception is thrown. So, the ERROR response is only used
when the client's request is invalid or not properly formed.

### The Operand Stack

When a client connects to the server, the server creates an *operand stack* for
the client. The operand values are strings corresponding to the *value* operand
type (see the [Network Protocol](#network-protocol) section.)

The PUSH and GET requests, if successful, push a single value onto the client's
operand stack.

The ADD, MUL, SUB, and DIV requests, if successful, pop two operands from the client's
operand stack, and push a single result value onto the operand stack.

The POP request, if successful, pops the top value from the client's operand stack.

The TOP request, if successful, allows the client to receive (via a DATA response)
the top value on the operand stack. Note that TOP does not modify the operand stack
in any way.

The purpose of the operand stack is to allow a client to execute a series of
operations that updated values in a table. For example, the following series of requests would
attempt to increase the numeric value of the "Apples" key of the "Fruit" table
by 10:

```
GET Fruit Apples
PUSH 10
ADD
SET Fruit Apples
```

### Transactions

A *transaction* allows a series of operations to be bundled into a unit which
either succeeds or fails as a whole.  For example, consider the example of
incrementing the value of a single tuple by 1:

```
GET Fruit Apples
PUSH 1
ADD
SET Fruit Apples
```

If two clients attempt this sequence of operations concurrently, it is not
guaranteed that the value of the tuple will increase by 2. For example,
let's say the initial value of the "Apples" tuple is 13. The following
interleaving in the processing of requests from the two clients might occur:

<div class='highlighter-rouge'><pre>
<i>Client 1</i>                 <i>Client 2</i>                 <i>Explanation</i>

GET Fruit Apples                                  <i>Client 1 gets the value 13, pushes it</i>
                         GET Fruit Apples         <i>Client 2 gets the value 13, pushes it</i>
PUSH 1                                            <i>Client 1 pushes 1</i>
ADD                                               <i>Pop 1, pop 13, push sum 14</i>
SET Fruit Apples                                  <i>Client 1 sets the value 14</i>
                         PUSH 1                   <i>Client 2 pushes 1</i>
                         ADD                      <i>Pop 1, pop 13, push sum 14</i>
                         SET Fruit Apples         <i>Client 2 sets the value 14</i>
</pre></div>

Assuming each request is handled separately, and that the SET operations
are executed using *auto commit* (more about this later), then this
sequence of operations would result in the value of the "Apples" tuple
in the "Fruit" table being incremented by only 1 (i.e., final value of 14
rather than 15.)

The BEGIN and COMMIT requests begin and end a transaction (respectively.)
For example, the following series of requests would attempt to increment
a single value using a transaction:

```
BEGIN
GET Fruit Apples
PUSH 1
ADD
SET Fruit Apples
COMMIT
```

Note that just because a client *attempts* a transaction doesn't guarantee
that the transaction will *succeed*. Any operation attempted within a
transaction could result in a FAILED response, which indicates that
the transaction has failed, and the server implicitly *rolls back* any proposed
modifications to tables made by operations attempted within the transaction.

If the COMMIT request succeeds (i.e., the server responds with an
OK message), then the transaction has succeeded, and proposed changes
to tables made within the transaction have been successfully
*committed*.

## Implementation and Testing

This section discusses important implementation details, and covers how you
should test your code to make sure it works correcty.

### Unit Tests

The unit test program tests the `Message`, `Table`, and `ValueStack` classes,
as well as the `encode` and `decode` functions in the `MessageSerialization`
namespace.

The unit tests are the primary functional specification for these classes
and functions. They also serve as an example of how your client and server
programs can *use* these classes and functions for network communication and
processing client requests.

To compile and run the unit test program:

```
make unit_tests
./unit_tests
```

If all of the unit tests pass, congratulations, you are ready to work on the
clients and server.

### `Message`, `MessageSerialization`

The `Message` class represents a single message (either a request or a response.)
It has helper functions to access the message's arguments (key, table, value, etc.),
and to check whether or not the contents of the message are *valid*, meaning that
they conform to the requirements specified above in the [Network Protocol](#network-protocol)
section.

The functions `MessageSerialization::encode` and `MessageSerialization::decode`
convert between `Message` objects and the "encoded" form of a message as a
string of characters. The `encode` function converts from `Message` to string,
and the `decode` function converts from string to `Message`. These will be
very helpful when implementing the actual network communication in the clients
and server!

Note that `encode` and `decode` should throw an `InvalidMessage` exception
in the following situations:

* In `encode`, if the resulting encoded message exceeds the maximum message length
* In `decode`, if the source encoded message exceeds the maximum message length
* In `decode`, if the source encoded message isn't terminated by a newline (`\n`)
  character
* In `decode`, if the resulting `Message` object isn't valid (i.e., if its
  `is_valid()` member function returns false)

### `ValueStack`

A `ValueStack` object represents a stack of string values. When a client connects
to the server, the server will use a `ValueStack` object to handle operations that
push values to and/or pop values from the client's operand stack.

The member functions of `ValueStack` should be fairly self-explanatory.
There are unit tests for testing the behavior of these member functions.

Note that the `get_top()` and `pop()` member functions should throw
`OperationException` if they are called when the stack is empty.

### `get_value` Client

The `get_value` client retrieves the value of one key
and, if the value is successfully retrieved, prints it out
as a single line of text and exits with exit code 0.

Its invocation syntax is

<div class='highlighter-rouge'><pre>
./get_value <i>hostname</i> <i>port</i> <i>username</i> <i>table</i> <i>key</i>
</pre></div>

Example run (user input in **bold**):

<div class='highlighter-rouge'><pre>
$ <b>./get_value localhost 5000 alice fruit apples</b>
42
</pre></div>

Your implementation should use `open_clientfd` to open a TCP connection
to the server, then send LOGIN, GET, and TOP messages to log in and
attempt to retrieve the value associated with the key. It should also
send a BYE message to politely terminate the connection.

If the value is retrieved successfully, it should be printed out
on a single line of text. Make sure the line is terminated by a
newline (`\n`) character.

If the client can't connect, if any communication errors occur, or if
the server responds to any of the request messages with an ERROR or
FAILED response, the client should print (to `std::cerr`)
a message of the form

<div class='highlighter-rouge'><pre>
Error: <i>message text</i>
</pre></div>

and then exit with a non-zero exit code.

If the server responded to any request with an ERROR or FAILED
response, *message text* should be the contents of the *quoted text*
argument the server sent back. Otherwise, any descriptive text is fine.

### `set_value` Client

The `set_value` client sets the value associated with a specific key.

Its invocation syntax is

<div class='highlighter-rouge'><pre>
./set_value <i>hostname</i> <i>port</i> <i>username</i> <i>table</i> <i>key</i> <i>value</i>
</pre></div>

Example run (user input in **bold**):

<div class='highlighter-rouge'><pre>
$ <b>./set_value localhost 5000 alice fruit apples 67</b>
</pre></div>

As with the `get_value` client, use `open_clientfd` to create a TCP connection to
the server. It should then send LOGIN, PUSH, and SET messages to set
the value associated with the key to the specified value, and then
use BYE to politely terminate the connection.

If `set_value` is successful, it should exit with an exit code of 0 without
printing any output.

The error handling requirements are the same as for `get_value`.

### `incr_value` Client

The `incr_value` client attempts to increase the numeric integer value associated
with a specific key by 1.

This client has an optional first argument, `-t`, which specifies that
attempting to increment the value associated with the key should be done
in a transaction.

Its invocation syntax is

<div class='highlighter-rouge'><pre>
./incr_value <i>hostname</i> <i>port</i> <i>username</i> <i>table</i> <i>key</i>
./incr_value -t <i>hostname</i> <i>port</i> <i>username</i> <i>table</i> <i>key</i>
</pre></div>

Example run (user input in **bold**):

<div class='highlighter-rouge'><pre>
$ <b>./incr_value localhost 5000 alice fruit apples</b>
</pre></div>

Like the other clients, `incr_value` should use `open_clientfd` to create a TCP
connection to the server.

A typical sequence of messages to increment a value would be
GET, PUSH, ADD, and SET. Surrounding these operations with BEGIN and
COMMIT would attempt to execute them as a transaction.

The client should send BYE to politely terminate the connection
(if no errors occur prior to that point.)

If `incr_value` is successful, it should exit with an exit code of 0 without
printing any output.

The error handling requirements are the same as for `get_value`.

### The Server

The server listens for connection requests from clients, and creates a thread
for each accepted client.

The server is responsible for creating and managing tables on behalf of
the clients. It is also responsible for using appropriate synchronization
when the data of a table is accessed and/or modified. The general rule is that if
the server reads from or writes to a table, it needs to be holding that
table's *lock*. Each `Table` object should have a mutex (`pthread_mutex_t`)
to serve as its lock.

When the server reads a request from a client and processes the request,
it will do so in either *autocommit mode* or *transaction mode*.
Autocommit mode is the default (i.e., each client starts out in autocommit
mode.)

When in autocommit mode,
the server uses a blocking call to `pthread_mutex_lock` to lock the table
accessed by the request (if there is one.) When it finishes handling the request, it
uses `pthread_mutex_unlock` to unlock the accessed table (if there is
one.) This has the effect of making each individual request an atomic
operation. However, it does mean that a *series* of operations, such
as GET/PUSH/SET to increment the value of a tuple, aren't atomic.

When in transaction mode (started by a BEGIN request from the client),
the server will use `pthread_mutex_trylock` to (attempt to) lock any
table accessed within the transaction. The essential thing to understand
is this: if any call to `pthread_mutex_trylock` in a transaction is
unsuccessful, the transaction has failed, and the server must roll back
any modifications made to tables touched by the transaction, as well as
release any successfully-acquired locks.  This implies that the server must
keep track of which `Table` objects are currently locked, in order to know
which must be unlocked at the end of the transaction. If a transaction
succeeds (when the client sends a COMMIT request), the server commits the
modifications to any modified tables, and releases the locks on all locked tables.

Note that in addition to the possibility of a transaction failing because
the server fails to acquire a lock on a table used by the transaction,
another reason a transaction could fail is if any request in the transaction
cuases an `OperationException`.

When a transaction ends, either by failing or succeeding, the connection
returns to the default auto-commit mode.

<div class='admonition info'>
  <div class='title'>Avoiding Deadlock</div>
  <div class='content' markdown='1'>
The reason that transactions must use `pthread_mutex_trylock` rather than
`pthread_mutex_lock` to lock tables is to avoid *deadlocks*. The problem is
that when many clients are attempting to perform transactions, they may
access (and need to lock) tables in different orders. When multiple tasks
attempt to lock multiple resources in an inconsistent order, deadlocks are
possible.

Having transactions fail when a lock can't be immediately acquired is a very
simple strategy for avoiding deadlock, and the cost of requiring the client
to explicitly retry the transaction.

Note that there is no worry about deadlock for clients in autocommit mode.
For those clients, they are only ever waiting for, acquiring, and releasing
one lock at a time. A deadlock situation always involves multiple locks.
  </div>
</div>

The `Table` class has member functions `lock`, `unlock`, and `trylock`.
These should call `pthread_mutex_lock`, `pthread_mutex_unlock`, and
`pthread_mutex_trylock` (respectively) on the table's internal mutex.
The `trylock` member function returns `bool` in order to let the caller
know whether or not the table's lock was successfully acquired.
(All of the discussion above about locking operations should be read as
meaning that the server code calls these member functions on a `Table`
object.)

Note that *nested transactions* are not allowed. If a BEGIN request is
received at a point where a transaction is already underway, the
server should fail the current transaction.

### Handling Exceptions in the Server

In general, any time your server code encounters a runtime error that means
it can't continue normally, it should throw an exception.  Four exception
types are provided:

Exception Type | Purpose
-------------- | -------
`InvalidMessage` | A received message was invalid (does not conform to the specification in the [Network Protocol](#network-protocol) or [Message Sequencing](#message-sequencing) sections, or the protocol was violated (e.g., the client sent a response message rather than a request message)
`CommException`  | An I/O operation failed
`OperationException` | A request couldn't be processed because of missing or invalid data, e.g., TOP request when the stack is empty, ADD request when the top two values aren't both integers, etc.
`FailedTransaction` | A table used within a transaction couldn't be locked immediately (because another client has the lock), or client attempted to start a nested transaction

In general, `InvalidMessage` and `CommException` represent unrecoverable errors
(the conversation with the client must end),
while `OperationException` and `FailedTransaction` represent recoverable errors
(the conversation with the client should continue.)

You should find that exceptions make it very straightforward to deal with runtime errors.
If you consistently throw an exception to indicate a runtime error, all of the code
that follows the place where the exception is thrown can assume that the
exception *didn't* get thrown, i.e., the runtime error the exception signifies
didn't occur.

There is one *hugely* important consideration for handling exceptions in the server:
you must ensure that all table locks are released in a timely manner, including when
exceptions occur.

You will most likely want `ClientConnection` to keep track of which tables
have been locked by the current transaction. That way, when the transaction
ends (either succeeding or failing), you can unlock all tables that
were locked by the transaction. Keeping track of which tables are locked
is also important to avoid attempting to acquire a lock on a table that
has already been locked. Once a transaction has locked a particular table,
it doesn't need to lock that table again in the same transaction.

You will need to think carefully about how to handle exceptions.
The "recoverable" exceptions (`OperationException` and `FailedTransaction`)
should not end the conversation with the client, while the
"unrecoverable" exceptions (`CommException` and `InvalidMessage`)
should end the conversation with the client. As noted above, any
time a transaction fails, all updates to tables must be rolled back,
and all locks held on tables must be released. The `catch` blocks
you use to handle exceptions thrown in the course of communicating with
the client (in the `ClientConnection` class) are a good place to put
the failed transaction recovery code.

### Manual Testing of Clients

One way to "manually" test client programs is to run the `nc` (netcat) program
using the `-l` option to "pretend" to be a server. Then, you can execute one
of your client programs against it. Messages sent by the client will appear
in the terminal window in which you're running netcat. Any lines you type
in the server terminal window will be send as a message back to the client
program.

For example, let's say you run netcat in one terminal window:

```
nc -l 5001
```

Then you could run `get_value` in another terminal window:

```
./get_value localhost 5001 alice fruit apples
```

Here's what the server terminal window might look like in order to
communicate with the client. (Response messages typed into this
terminal window are shown in **bold**):

<div class='highlighter-rouge'><pre>
LOGIN alice
<b>OK</b>
GET fruit apples
<b>OK</b>
TOP
<b>DATA 42</b>
BYE
<b>OK</b>
</pre></div>

Having netcat pretend to be a server is an excellent way to see how
your client program is behaving.

You can also test your client program against the reference server
implementation. E.g., you could run the reference server in its
own terminal window:

```
./ref_server 5002
```

Then have a client program connect to it from a different terminal window:

```
./get_value localhost 5002 alice fruit apples
```

<div class='admonition info'>
  <div class='title'>Note!</div>
  <div class='content' markdown='1'>
When you run the reference server, you will need to create at least
one table before your client programs will be able to do anything
interesting. You can use netcat for this. For example, assuming the
server is listening for connections on port 5003 (user input
in **bold**):

<div class='highlighter-rouge'><pre>
$ <b>nc localhost 5003</b>
<b>LOGIN alice</b>
OK
<b>CREATE fruit</b>
OK
<b>BYE</b>
OK
</pre></div>

The example above would create a table named "fruit", allowing you
to get, set, and increment values in that table using your clients.
  </div>
</div>

The following screencast video walks through how to test the clients
using the reference server and netcat:

<https://jh.hosted.panopto.com/Panopto/Pages/Viewer.aspx?id=4f42576a-20ae-40fb-99b3-b22200df509f>

### Automated Testing of Clients

You can do some automated testing of your `incr_value` client program as follows.

1. Download [`ms1_concurrency_tests.zip`](ms1_concurrency_tests.zip) into
   your working directory, e.g., by running the command  
   ```
   curl -O https://jhucsf.github.io/spring2024/assign/ms1_concurrency_tests.zip
   ```
   <p>
2. Unzip the zipfile:  
   ```
   unzip ms1_concurrency_tests.zip
   ```
   <p>
3. Make the scripts and `supervise` program executable:  
   ```
   chmod a+x supervise incr_value_concurrent.sh incr_value_worker.sh ref_client.rb
   ```
   <p>
4.  Make sure your `incr_value` client is up to date:  
    ```
    make depend
    make incr_value
    ```

Once you've done the above steps, run the test script using one of
the following two invocations:

```
./supervise ./incr_value_concurrent.sh PORT
./supervise ./incr_value_concurrent.sh -t PORT
```

Replace `PORT` with a port number, e.g., "5000".

The first invocation runs `incr_value` without transactions (autocommit mode),
and the second invocation runs `incr_value` with transactions.

If the last line of output printed by the script is "`Success!`", then
your `incr_value` client worked as expected.

This test script is the same one that the MS1 autograder uses for the
two `incr_value` concurrency tests.

### Server Implementation (in Milestone 2)

In Milestone 2, your main task is to implement the server program.
(Reminder: the [Suggested Approach](#suggested-approach) section has our
recommendations on how to approach this milestone.)

In the `Server::server_loop` member function, you should add a loop
to wait for incoming connections using the `accept` system call, and
for each accepted connection, create a `ClientConnection` object
to encapsulate the new connection, and start a thread which will call
the `Server::client_worker` static member function, which in turn will
call the `ClientConnection` object's `chat_with_client` member function.

The `chat_with_client` member function should implement a loop in which
each iteration reads a request message from the client, processes the request,
and sends a response to the client.

If a client sends an invalid message to the server, the server should
send back an ERROR response.

If an error occurs communicating with the client (e.g., an error occurs
reading from the client socket), the server should terminate the connection
and end the client connection thread. It's a good idea to have the server
print an error message for diagnostic purposes, but this is not required.

Note that client threads should be detached, since the main thread
won't call `pthread_join` to wait for them to complete. When a
client thread's start function executes, it can call `pthread_detach`
to detach itself:

```c
pthread_detach( pthread_self() );
```

<div class='admonition caution'>
  <div class='title'>Note!</div>
  <div class='content' markdown='1'>
The starter code didn't include this call in the suggestion for the `client_worker`
function, so you should add it yourself.
  </div>
</div>

### Synchronization

In addition to the synchronization required to access and modify data in
a table (as described in [The Server](#the-server) section),
you will also need to implement synchronization to manage the creation of
and access to `Table` objects. The `Server` object should maintain a collection
of `Table` objects created in response to CREATE messages. Because multiple
client threads will be using this collection, accesses to the collection will
need to be synchronized.

### Manual Testing of the Server

Initially, you will want to test your server "manually" using netcat.

First, start your server in one terminal window:

<div class='highlighter-rouge'><pre>
./server <i>port</i>
</pre></div>

Any port number 1024 or above may be used. Note that if the port you specify
is currently in use, `open_listenfd` will return an error result, and the
server should print an error message and exit. If this occurs, just choose
a different port.

In a second terminal window, run netcat to connect to your server:

<div class='highlighter-rouge'><pre>
nc localhost <i>port</i>
</pre></div>

At this point, lines that you type (as input to the `nc` program) will be sent to
the server as request messages, and any responses the server sends back will
appear as lines of text.

You may find it helpful to run a netcat client against the reference server
so that you can get a better sense of how your server is expected to respond
to requests.

One challenge of implementing the server is that it is not particularly "observable",
so it can be hard to know what it is doing or not doing. It can be very helpful
for the server to print diagnostic messages to indicate what it is doing.
When the server is not behaving the way you expect, these diagnostic messages can
help you determine where it is going wrong.

### Automated Testing of the Server

The zipfile [`ms2_tests.zip`](ms2_tests.zip) has some automated server functionality
tests.

To use it:

1. Download it:  
   ```
   curl -O https://jhucsf.github.io/spring2024/assign/ms2_tests.zip
   ```
   <p>
2. Unzip it:  
   ```
   unzip ms2_tests.zip
   ```
   <p>
3. Make the scripts, `supervise` program, and `ref_incr_value` programs executable:  
   ```
   chmod a+x scripts/*.sh scripts/*.rb supervise ref_incr_value
   ```
   <p>
4. Make sure your `server` executable is up to date:  
   ```
   make depend
   make server
   ```

The following invocations run the test scripts:

```
./supervise ./scripts/server_get_set_multi_interleaved.sh PORT
./supervise ./scripts/server_concurrency_increment_notrans.sh PORT
./supervise ./scripts/server_concurrency_increment_trans.sh PORT
./supervise ./scripts/server_concurrency_trans_multiple_tables.sh PORT
```

Replace `PORT` with a port number, e.g., "5000".

### Synchronization Report

For Milestone 2, your `README.txt` should explain your approach to synchronization.
In particular:

* What data structures needed to be synchronized, and why?
* How did you synchronize the data structures requiring synchronization?
* Why are you confident that the server is free of race conditions
  and deadlocks?

## Submitting

Edit the `README.txt` file to summarize each team member's contributions
and (for Milestone 2) to add the report on how you approached synchronization in the server
program.

You can create a zipfile of your solution using the command `make solution.zip`.

Submit your zipfile to Gradescope as **Assignment 5 MS1** or
**Assignment 5 MS2**, depending on which milestone you are
submitting.

Note that there should be only **one** submission for your group.
Make sure that all group members are added to the submission.
