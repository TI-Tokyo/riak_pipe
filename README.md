Riak Pipelines
==============

![Riak Pipe OpenRiak Status](https://github.com/OpenRiak/riak_pipe/actions/workflows/erlang.yml/badge.svg?branch=openriak-3.2)

# Overview

`riak_pipe` is most simply described as "UNIX pipes for Riak." In much
the same way you would pipe the output of one program to another on the
command line (e.g. `find . -name *.hrl | xargs grep define |
uniq | wc -l`), `riak_pipe` allows you to pipe the output of a function
on one vnode to the input of a function on another (e.g.
`kvget | xform | reduce`).

There are two big departures that `riak_pipe` makes from UNIX pipes. The
first is that input and output streams are message-based, not character
based. A function emits zero or more messages, instead of writing bytes
to a file descriptor.

The second difference is that `riak_pipe` pipeline specifications also
include information about where an input should be sent. The decision is
made by a function evaluated on the input messages, which produces a
160-bit hash that can be compared to the ownership of partitions in Riak
Core's ring. This is how parallelism is controlled in `riak_pipe`: the
"same" work function will be run on multiple vnodes to handle multiple
inputs concurrently, if the consistent hashing function points to a
different partition for each input.

# How does it work?

## Pipelines, Fittings, Behaviors, ...

From a conceptual level, the top-level object in `riak_pipe` is the
**pipeline**. A pipeline is composed of one or more **fittings**. Each
fitting is a specification of a **behavior** and a **partitioner**. The
behavior is what defines how an **input** turns into zero or more
**outputs** (i.e. the "program" of the UNIX pipe). The partitioner is
what defines which vnode will be handling the processing for each input.

## Vnodes as queue managers

As with all applications built atop Riak Core, the primary operational
component of `riak_pipe` is the vnode. In `riak_pipe`, a vnode is
primarily a queue manager.

Every input delivered to a vnode specifies which fitting it is intended
for (and, hence, the behavior that should be used to process it). The
vnode, upon receiving an input, adds it to a queue of other inputs for
that fitting.

The first time a `riak_pipe` vnode receives a message for a fitting it
has never seen before, it also starts a **worker process** (hereafter
referred to as simply a **worker**) to consume the queue. The worker is
where the behavior is evaluated. In this way, each fitting's queue is
consumed in parallel to others, as the worker for each queue operates
independently.

Workers are also the actors that send outputs to the next vnode that
will process them. This is important because that sending process will
**block** until the input is added to the queue assigned to the
appropriate worker on the next vnode.

The sizes of queues are limited, so if a worker is particularly slow,
its queue will reach capacity and workers adding new inputs will block
until the worker catches up. This back pressure prevents processes and
their mailboxes from growing unboundedly when workers outrun their
followers.

## Coordinators

For each fitting to be executed, there is a **coordinator** for the
cluster as a whole, separate from the per-vnode worker. Each coordinator
is responsible for conveying its fitting's behavior as well as handling
the coordination necessary at the end of input.

## Sinks

Each pipeline has a **sink**, where all of the final output is sent
(and, optionally, log messages as well). By default, the sink and the
client process that initiated the pipeline are one and the same.

# Step by Step

## Start the pipeline

The first step in performing an operation on `riak_pipe` is to set up
the coordinators to manage the pipeline. Each coordinator knows the
details for the behavior of its fitting, the partition choice for
distributing output to the workers for the next fitting, and the `pid`
of the next coordinator.

`riak_pipe` exposes a client interface `riak_pipe` exposes a client interface
(see the [*API*](#api) section) for starting these coordinators.
It begins by starting a "builder" process, which then starts and monitors
each of the coordinators, starting with the final one (so that its `pid`
can be passed to its predecessor). When all of the coordinators have started,
a structure capturing their details is returned to the client.

## Send inputs

All inputs are sent directly to the vnodes that will process them. They
are tagged with the `pid` of the coordinator that holds the details of
how they will be processed. A vnode will add each input to a queue it
maintains for that fitting. The input-sending operation will block until
the input has been added to the queue.

Sending inputs directly to vnodes is done instead of sending every input
to the coordinator and distributing from there, in order to remove a
bottleneck and reduce messaging.

Blocking input-sending until the enqueueing process completes is used as
a means of providing back pressure for workers that would otherwise
overwhelm their downstream partners. When a queue is already at its
limit, the acknowledgment of reception of the input will be withheld
until room has been made in the queue. Because each vnode maintains only
one worker per fitting, the total number of blocking workers (and thus
the size of the waiting list) should never exceed the number of
partitions in the cluster.

## Fetch fitting

The first time a vnode receives an input for a fitting that it has not
seen before, it requests the details of that fitting from the `pid` in
the input tag (the coordinator). Once it has those details, it may spin
up a worker and begin processing items in its queue.

## Process inputs

When workers are idle, they ask their owning vnode for the next item in
their queue. If none is available, the request blocks until new inputs
arrive. If an input is available, it is dequeued, and sent to the worker
for asynchronous processing. At this point, a blocking request to add an
input to the queue may be honored.

## Send end-of-inputs

When a client has no more inputs to add to the pipeline, it notifies the
first coordinator of the end of its inputs (EOI). The coordinator then
notifies each of the vnodes that is working for it that no more inputs
will be coming. The coordinator created this list of workers by
remembering every partition that has asked for the details of the
fitting (and monitoring for crashes of those vnodes).

## Wait for done

When a vnode receives an end-of-inputs message from a coordinator, it
marks the worker's queue. When the worker processes the final element in
the queue (including any that may have been blocking), the vnode shuts
down the worker, and notifies the coordinator that it has finished.

## Forward end-of-inputs

When a coordinator receives all of the 'done' messages from vnodes that
were working for it, it forwards the end-of-inputs messages to the next
coordinator, where the eoi/done-signaling begins again, or the sink.

This form of end-of-inputs signaling works because all input sending is
synchronous (blocking until confirmed queue addition) in `riak_pipe`.
This means that no inputs will be in flight, in delayed messages, when
end-of-inputs is sent. In addition, synchronizing all "done" messages
for a fitting in the coordinator means that no additional tracking of
which workers have finished is necessary.

# API

## Client

### Pipeline Specification

Each pipeline is constructed via a call to `riak_pipe:exec/2`.

The first argument to `riak_pipe:exec/2` is a list of fitting
specifications, in the order that data will flow through them. Fitting
specifications are given as `fitting_spec` records, as defined in
`include/riak_pipe.hrl`.

``` erlang
#fitting_spec {
   name = foo, %% term(), anything to help you help you find results
               %% and log output from this fitting

   module = riak_pipe_w_pass, %% atom(), must be a valid module name,
                              %% the name of the module implementing the
                              %% riak_pipe_vnode_worker behavior for this
                              %% fitting

   arg = undefined, %% term(), opaque static argument included in the
                    %% fitting_details structure passed to the worker
                    %% module (useful for initial configuration)

   chashfun = fun(_) -> <<0:160/integer>> end,
                              %% arity-1 function() | 'follow'
                              %% specification of how to distribute
                              %% inputs for this fitting

   nval = 1,%% positive integer, default 1, indicates how many vnodes
            %% may be asked to handle an input for this fitting,
            %% before declaring the input unfit for processing

   q_limit = 64 %% positive integer, default 64, sets the maximum
                %% number of elements allowed in a worker's queue,
                %% the actual queue limit is the lesser of this value
                %% and the worker_q_limit variable in riak_pipe's
                %% application environment (default 4096)
}
```

The example above would create a fitting named "foo" (this name would
appear in error, log, and result messages generated by this fitting).
The workers spawned would all run the `riak_pipe_w_pass` module against
their inputs (see the [*Behavior*](#behavior) section of this document
for more examples of fitting worker behaviors included with `riak_pipe`).
Finally, there would be only one worker spawned for the entire cluster,
on the partition owning range in which the hash "0" falls, since the
`chashfun` function always produces "0", regardless of input.

Using the value `follow` in the chashfun field means inputs for the
fitting should be sent to the same vnode that generated them. This is
useful for maintaining data locality for a series of operations, instead
of potentially pushing each modification's output across inter-node
links.

The second argument to `riak_pipe:exec/2` is a proplist of options for
the pipeline as a whole. Currently supported options are:

#### *sink*

`fitting()` \| **`undefined`**

Where workers for the final fitting should send messages. Leave this
`undefined` to deliver outputs as messages to the client, the process
that called `riak_pipe:exec/2`.

#### *log*

`sink` \| `lager` \| `sasl` \| **`undefined`**

If set to `sink`, log messages will be sent to the sink process (just as
outputs from the final fitting are). If set to `lager`, log messages are
printed to the Riak log on whatever node produces them. If set to
`sasl`, log messages are printed to the SASL log on whatever node
produces them. If left `undefined`, log messages will be ignored
silently.

#### *trace*

`all` \| `[trace_item()]` \| **`undefined`**

If set to `all`, send all trace messages produced to the log. If set to
a list, only send messages to the log if one of their types matches one
of the types listed. If left undefined, all trace messages are ignored
silently.

The `riak_pipe:exec/2` function will return a tuple of the form `{ok,
Pipe}`, `Pipe` being a handle to the pipeline that was created (in the
form of a `pipe` record, as defined in `include/riak_pipe.hrl`), which
you will use to send inputs, to indicate the end of inputs, and to
receive outputs later.

### Sending inputs

Once you have the pipe handle, you can send inputs to it using
`riak_pipe:queue_work/2`:

``` erlang
riak_pipe:queue_work(Pipe, "please process this list").
```

The `queue_work/2` function evaluates the `chashfun` function (from the
first fitting's specification) against `Input`, and then sends `Input`
to the vnode owning the range in which the hash falls.

### Sending eoi

When you have sent all the inputs you want to the pipeline, tell it that
you're done with `riak_pipe:eoi/1`:

``` erlang
riak_pipe:eoi(Pipe)
```

This allows the coordinator to begin shutting down its workers. When all
of a fitting's workers finish, the coordinator will automatically
forward the `eoi` (End Of Inputs) to the following coordinator, and the
final coordinator will send its `eoi` to the sink.

### Receiving outputs

Results are delivered to the sink as `pipe_result` records (defined in
`include/riak_pipe.hrl`).

``` erlang
#pipe_result {
   ref = #Ref<w.x.y.z>, %% reference(), an opaque reference, the same
                        %% given in the sink's #fitting.ref field
                        %% (useful for using the same process as
                        %% multiple sinks)

   from = foo %% term(), the #fitting_spec.name field for the fitting
              %% that produced this result

   result = "please process this list" %% term(), the result produced
                                       %% by the worker
}
```

When the final coordinator finishes, its `eoi` is delivered as a
`pipe_eoi` record (also defined in `include/riak_pipe.hrl`).

``` erlang
#pipe_eoi {
   ref = #Ref<w.x.y.z>, %% reference(), an opaque reference, the same
                        %% given in the sink's #fitting.ref field
                        %% (useful for using the same process as
                        %% multiple sinks)
}
```

If you'd rather receive the entire result set at once, instead of
streamed, you can use the `riak_pipe:collect_results/1` function:

``` erlang
{ok, Results, LogMessages} = riak_pipe:collect_results(Pipe).
```

The `receive_result/1` function is also exported from `riak_pipe` to
make it easy to wait for the next piece of output:

``` erlang
consume(Pipe) ->
   case riak_pipe:receive_result(Pipe) of
      {result, {From, Result}} ->
         io:format("Received ~p from ~p!~n", [Result, From]),
         consume(Pipe);
      {log, {From, Msg}} ->
         io:format("Logged ~p from ~p!~n", [Msg, From]),
         consume(Pipe);
      eoi ->
         io:format("Done!~n")
   end.
```

### Receiving log and trace messages

If you set `{log, sink}` in the options sent to `riak_pipe:exec/2`, then
logging messages (as well as trace messages, if you enabled them) will
be delivered to the sink, in a similar manner that results are, but as a
`pipe_log` record (defined in `include/riak_pipe.hrl`).

``` erlang
#pipe_log {
   ref = #Ref<w.x.y.z>, %% reference(), an opaque reference, the same
                        %% given in the sink's #fitting.ref field
                        %% (useful for using the same process as
                        %% multiple sinks)

   from = foo %% term(), the #fitting_spec.name field for the fitting
              %% that produced this result

   msg = {processed, "please process this string"}
              %% term(), the log message
}
```

See the [*Receiving outputs*](#receiving-outputs) section above for
details about using the `collect_results/1` and `receive_result/1`
functions exported from `riak_pipe` instead of pulling these records out
of a mailbox explicitly.

### Inspecting Performance

While a pipeline is running, some information about what its workers are
up to can be fetched using `riak_pipe:status/1`. For example:

``` erlang
{ok, Pipe} = riak_pipe:example_start().
ok = riak_pipe:queue_work(Pipe, "foo").
Status = riak_pipe:status(Pipe).
```

The code above starts a pipeline with one fitting named `empty_pass`,
sends it one input, and then requests its status. The `Status` received
on the last line, will be something like the following:

``` erlang
[{empty_pass,[[{node,'riak@127.0.0.1'},
               {partition,22835963083295358096932575511191922182123945984},
               {fitting,<0.469.0>},
               {name,empty_pass},
               {module,riak_pipe_w_pass},
               {state,waiting},
               {inputs_done,false},
               {queue_length,0},
               {blocking_length,0},
               {started,{1308,854463,966730}},
               {processed,1},
               {failures,0},
               {work_time,52},
               {idle_time,10054032}]]}]
```

That is, `Status` is a list with one entry per fitting. The example
pipeline had only one fitting, so this list has only one entry.

Each fitting's entry is a 2-tuple of the form `{Name, WorkerList}`. The
`Name` is the name provided to `riak_pipe:exec/2`, or `empty_pass` in
our example case. `WorkerList` is a list containing the status of each
worker.

The status of each worker is represented as a proplist of details. Since
the example sent only one input, and it was processed successfully, only
one worker exists in this example list. The details indicate which node
the worker is running on, and which partition (vnode) it's working for.
The also indicate that there's nothing more waiting in the worker's
queue, that one inputs was processed, and that it took 52 microseconds
to process that input. Details about the other fields in this proplist
can be found in the documentation of `riak_pipe_vnode:status/1`.

## Behavior

Fitting behaviors have it easy. They need only expose three functions:
`init/2`, `process/3`, and `done/1`. All of the details about consuming
items from the queue maintained in the vnode is handled by the
`riak_pipe_vnode_worker` module.

It's a good idea to add `-behavior(riak_pipe_vnode_worker).` to the top
of your fitting behavior module. That will give the compiler a clue that
it should warn you if you forget to export a required function.

### Init

When a vnode starts up a worker, the behavior module's `init/2` function
is called with the partition number of the owning vnode, and the details
of the fitting the worker handles input for. The `init/2` function
should return a tuple of the form `{ok, State}`, where `State` is a term
that will be passed to the module's `process/3` function later.

Unless your behavior will not be producing any output, it will want to
stash the partition number and fitting details somewhere, as they are
required parameters for sending output.

### Process

The behavior's `process/3` module will be called each time a new input
is pulled from the queue. The parameters to `process/3` will be the new
input, a "last in preflist" indicator (see
[Aside: Preflist Forwarding and Nval](#aside-preflist-forwarding-and-nval)
below), and the module's state (initialized in `init/2`). The function
must return a tuple of the form `{Result, NewState}` where `NewState` is
a potentially modified version of the state passed in (this `NewState`
will be passed in as the state when evaluating the next input, much like
the "accumulator" of a list-fold function). The `Result` may be any of
the following:

#### `ok`

processing succeeded, and work may begin on the next input

#### `forward_preflist`

this input should be forwarded to the next vnode in its preflist, for
processing there

#### `{error, Reason}`

this input generated an error; it should not be retried on other nodes
in the preflist, and an error should be logged

If processing the given input should produce some output, `process/3`
should call `riak_pipe_vnode_worker:send_output/3`:

``` erlang
riak_pipe_vnode_worker:send_output(
   Output,
   State#state.partition,
   State#state.fitting_details).
```

Much like when a client sends inputs, the `send_output` function will
evaluate the `chashfun` function for the next fitting against the
`Output` given, and send that `Output` to the chosen vnode. Remember
that this function will block until the item has been added to the
vnode's queue for the next fitting.

### Done

A behavior's `done/1` function is called when the worker's queue is
empty after it receives the `eoi` (End Of Inputs) message from its
coordinator. This is the last chance that the behavior will have to
produce output. The function should return `ok`. The process running the
behavior will terminate shortly after `done/1` finishes.

### Optional: Validate Argument

If a behavior uses a static argument (the `arg` field in the
`fitting_spec` passed to `riak_pipe:exec/2`), it can validate the
argument before processing begins by exporting a `validate_arg/1`
function. If it does, the function will be called once, in the process
calling `riak_pipe:exec/2`.

If the argument is valid, `validate_arg/1` should return `ok`. If the
argument is invalid, `validate_arg/1` should return a tuple of the form
`{error, "This message explains the trouble."}`, explaining the error to
the user.

See the `riak_pipe_v` module for some useful included validators.

### Optional: Archive & Handoff

The Riak Core concept of "handoff" migrates vnodes from one node to
another when cluster membership changes. For `riak_pipe`, the handoff
process is mostly copying the worker queues from the old node to the new
node. If, however, a behavior maintains some state between input
processing, that state must also be moved.

To allow `riak_pipe` to move a worker's state from one node to another,
a behavior should export `archive/1` and `handoff/2` functions).

First, on the old node, the `archive/1` function will be called with the
last state returned from `init/2` or `process/3`. The function should
return a tuple of the form `{ok, Archive}`, where `Archive` is any
serializable term that represents the state needing to be transferred to
the new node. The worker terminates soon after `archive/1` completes.

When the new vnode receives a handoff message from the old node, it
makes sure that it has queues ready for the work (this may involve
starting a new worker, if it does not have one running already). It then
passed the `Archive` to its worker. The worker will evaluate the
behavior's `handoff/2` function with the `Archive` and the behavior's
current `State` as arguments. The function must return a tuple of the
form `{ok, NewState}`, where `NewState` is a possibly modified version
of the `State` variable, representing its merge with `Archive`.
Processing then continues as normal.

The `riak_pipe_w_reduce` module included with `riak_pipe` is a good
example of how `archive/1` and `handoff/2` can be implemented.

### Optional: Logging and Tracing

It can be useful for debugging and monitoring to have a behavior produce
logging and/or trace statements. The facilities for doing so are
exported from the `riak_pipe_log` module.

The `riak_pipe_log:log/2` function can be used to log anything,
unqualified. Simply pass it the fitting details (the second parameter of
the behavior's `init/2` function), and the message (any term) to be
logged.

The `riak_pipe_log:trace/3` function can be used to filter the output of
log messages. Pass it the fitting details and the message, as with
`log/2`, but also pass it a list of "topics" for this message. If your
topics are included in the topics that were passed as options to the
pipeline setup, the trace will be logged; otherwise, it will be dropped.

The `trace/3` function automatically adds two topics every time it is
called: the name of the fitting (from the `#fitting_spec` passed to
`riak_pipe:exec/2`) and the Erlang node name. This makes it easy to
trace work done on a specific node, or in a specific fitting.

You may also find the macros defined in `include/riak_pipe_log.hrl`
useful for logging and tracing. The `L` macro converts directly to a
call to `riak_pipe_log:log/2`, but is much shorter. The `T` macro
converts to a call to `riak_pipe_log:trace/3`, but also adds the calling
module's name to the list of topics. That is:

``` erlang
%% these two lines do the same thing
?L(FittingDetails, "my log message").
riak_pipe_log:log(FittingDetails, "my log message").

%% these two lines are also equivalent
?T(FittingDetails, [], "my trace message").
riak_pipe_log:trace(FittingDetails, [?MODULE], "my log message").
```

### Aside: Preflist Forwarding and Nval

As noted in the [*Process*](#process) section above, a fitting behavior may
return `forward_preflist` as its result. If it does so, the input will
be forwarded to the next vnode in its preflist.

"Preflist" is a concept from Riak Core. The main idea is that it may be
possible to evaluate an input on any of several different vnodes. The
preflist is an ordered list of these vnodes. Its length is determined by
the `nval` parameter of the fitting's specification.

Riak Pipe uses the preflist in order. That is, the first vnode in the
preflist is asked to evaluate the input. If that vnode's worker asks to
forward it along, only then is the next vnode in the preflist asked to
process the input.

When the final vnode in the preflist is given the input for processing,
its fitting behavior's `process/3` function will have the `LastPreflist`
parameter set to `true`. If the final vnode's worker again asks to
forward the input, an error is logged (either in the node's log, or via
a message to the sink, depending on `log` and `trace` execution
options).

This same variety of forwarding is used if a vnode worker should exit
abnormally, and then fail to restart. All items in the worker's blocking
queue and working queue, as well as all future inputs sent to the vnode
for that worker, are forwarded to the next vnode in the preflist.

### Aside: Processing Errors

Fitting behaviors can raise errors a few ways: via preflist exhaustion,
explicit error return, or exception.

As noted in the previous section, one way to raise an error is simply to
request that an input be forwarded past the end of the preflist. This
generates a trace error, with a proplist full of information, including
`{type, forward_preflist}`. If the preflist was empty, the proplist will
also contain `{error, [preflist_exhausted]}`.

A fitting behavior module may also explicitly return an `{error,
Reason}` tuple. If so, a similar trace error will be generated, but the
proplist will include `{type, result}` and `{error, Reason}`.

If the behavior raises an exception, yes another trace error is
generated, but the proplist now includes `{type, Type}` and `{error,
Error}` where `Type` and `Error` are matches from a `catch Type:Error
->` clause surrounding the call to the behavior. In this case, the
worker will also exit. If more inputs arrive for this fitting (or have
already arrived and are waiting in the queue), the worker will be
restarted, in the same manner it was started initially. This is meant to
give the behavior a chance to refresh any stateful resources it may have
been holding when the exception occurred.

The proplists generated by each of these error types also include useful
information like the `module` that implents the behavior, the
`partition` on which the worker was running, the `details` of the
fitting, the `input` that was being processed when the error occured,
the `modstate` state of the behavior module, and a `stack` trace.

If an error occurs that cannot be caught by the catch clause surrounding
the `process/3` evaluation, a similar, but limited, error trace will be
generated, with `{reason, Reason}` in the proplist (where `Reason` is
the exit reason received by the worker's vnode).

For these error traces to be visible, two execution options need to be
set: `log` and `trace`. The `log` option should be set to `lager` to put
these errors in the node's log, `sasl` to put them in the SASL log, or
`sink` to have them delivered to the sink. The `trace` option should be
set to at least `[error]`, though `all` will also work.

# Included Fittings

`riak_pipe` includes some standard fittings. They are all named with the
prefix `riak_pipe_w_`.

## `riak_pipe_w_pass`

The "pass" behavior simply emits its input as its output. It is
primarily useful for demonstration of the worker API, and for catching
the simple log/trace output it produces. It should tolerate whatever
partition function you throw at it, because it won't matter where it is
run.

## `riak_pipe_w_tee`

The "tee" behavior operates just like the "pass" behavior, but also
sends its input as output **directly to the sink**. It is primarily
useful for taking a look at intermediate results. Remember that results
delivered to the client are tagged with the name of the fitting that
produced them, so name your fittings wisely. "Tee" should also tolerate
whatever partition function you throw at it.

## `riak_pipe_w_xform`

The "xform" behavior is a simple transform operator. It expects an
arity-3 function as its argument. For each input it receives, it
evaluates that function on the input, partition, and fitting details for
the worker. The function should emit whatever outputs are appropriate
for it. The "follow" partition function is recommended for this
behavior, since that keeps data local to the node, instead of clogging
inter-node channels, but it should tolerate any partition function you
throw at it anyway.

## `riak_pipe_w_reduce`

The "reduce behavior" is a simple accumulating reducer. It expects an
arity-4 function as its argument. For each input it receives, it
evaluates the function on the cons of that input to the result of any
previous evaluations (or the empty list, if the function has never been
evaluated before).

The input to the fitting must be of the form `{Key, Value}`. Results are
maintained (and the function is evaluated) per-key.

When the behavior receives its "done" message, it emits the accumulated
result for each key as an output of the form `{Key,
Output}`.

Care should be taken when choosing a partition function for the "reduce"
behavior. If a function is used that produces two different partitions
for the same key, for example based on which node evaluates the
partition function, downstream phases will see two results for that key
(one from each reducer). This can be useful in some cases (for instance
two-stage reduce, where "follow" partitioning is used to reduce results
locally, before an identical reduce with a consistent partition function
is used to reduce globally), but surprising in many others.

## External: `riak_kv_pipe_get`

Riak KV includes a "get" behavior intended to aid computation on data
stored in Riak KV. The behavior expects its inputs to be of the form
`{Bucket, Key}`. The outputs it produces are of the form `{ok,
riak_object()}` or `{error, Reason}`.

The `riak_core_util:chash_key/1` function should always be used with the
KV "get" fitting. This partition function always chooses the head of the
preflist for the incoming bucket/key, ensuring that the index for the
`riak_pipe` vnode evaluating the input is the same as the index for the
KV vnode storing the data. This allows the behavior to efficiently ask
the KV vnode directly for the data, instead of working through the
`riak_kv_get_fsm`.

## Internal: `riak_pipe_w_crash`

The "crash" behavior is used in testing Riak Pipe. Using argument values
and inputs, it allows a pipeline to be setup that can later be forced to
crash in specific ways.

## Internal: `riak_pipe_w_fwd`

The "forward" behavior is used when a vnode worker exits abnormally, and
then also fails to restart. This fitting behavior simply returns
`forward_preflist` for every input it receives. Note that writing a
fitting spec to use `riak_pipe_w_fwd` means that the fitting will only
ever produce errors due to preflist exhaustion.

# Riak KV MapReduce Emulation

The `riak_kv_mrc_pipe` module in the Riak KV application provides a
compatibility layer for running existing Riak KV MapReduce queries on
top of `riak_pipe`. The `riak_kv_mrc_pipe:mapred/2` function accepts the
same input as the `riak_client:mapred/2` function. Support is currently
provided for `map` and `reduce` phases implemented in Erlang, specified
using the `{qfun, function()}` or `{modfun, Module,
Function}` syntax.

# Additional Documentation

A diagram recording the supervisor/link/monitor structure of the Erlang
processes involved in Riak Pipe is included in the file
`riak_pipe_monitors.dot`. The comments at the top of that file describe
how to render it to an image using Graphviz.

# Testing

System-level tests for Riak Pipe are included with the
[riak_test](https://github.com/basho/riak_test) repository.
You'll find them in the `tests` directory with names that start with
`pipe_verify_`.
