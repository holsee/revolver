# revolver

revolver is a round-robin load balancer for Erlang processes.

Compared to poolboy (https://github.com/devinus/poolboy) or worker_pool (https://github.com/inaka/worker_pool),
revolver is very simple and small (~150 LOC) with no concept of a lease and no assumptions about the workers.

The idea is that you have an existing OTP supervisor with children (your pool) and like to balance accross those.
This design makes it perfect for the fast paralellization of many equally sized tasks.

In addition, revolver has the notion of beeing disconnected.
So when parts of you application are temporarily unavailable (think database connections),
revolver acts as a fuse and lets the unaffected parts run while trying to reconnect.

## Building

```
git clone git://github.com/odo/revolver.git
cd revolver
./rebar get-deps compile
```

## Performance

On modern hardware, revolver can hand out more than 100 000 pids per second. So it will add an overhead < 10 microseconds to each call.

## Usage

### balancing

The idea is to balance accross processes owned by a supervisor. In this example we use the sasl supervisor (```sasl_sup```) which has two sub-processes.
In a typical setting, you would balance between a large set of identical gen_server processes.

```erlang
1> application:start(sasl).
ok
2> revolver:balance(sasl_sup, sasl_pool).
{ok,<0.43.0>}
3> revolver:pid(sasl_pool).
<0.40.0>
4> revolver:pid(sasl_pool).
<0.37.0>
5> revolver:pid(sasl_pool).
<0.40.0>
6> revolver:pid(sasl_pool).
<0.37.0>
```
### refreshing the pool

Members of the pool may die and revolver will notice and take them out of the rotation.
Eventually it will need to ask the supervisor again for its children.
When this happens is specified by the ratio of workers you need to have active.
The default is 1.0 (all of them) but if you have many workers and expect them to die frequently you might want to specify a smaller ration:

```erlang
revolver:balance(sasl_sup, sasl_pool2, 0.75).
```

## calling all members

You can do a series of calls to all members of a pool inside revolver:

```erlang
1> application:start(sasl).
ok
2> revolver:balance(sasl_sup, sasl_pool).
{ok,<0.43.0>}
3> revolver:map(sasl_pool, fun(Pid) -> proplists:get_value(status, process_info(Pid)) end).
[waiting,waiting]
```

## disconnect and reconnect

If the pool disappears (the supervisor exits or has no children) revolver returns {error, disconnected} when asked for a pid
and tries to refresh the pool periodically in the background. The delay for the refresh can be specified at start.

```erlang
1>  application:start(sasl).
ok

2>  revolver:balance(sasl_sup, sasl_pool, 1.0, 10000).
{ok,<0.44.0>}
=INFO REPORT==== 2-Oct-2014::09:55:41 ===
revolver: Found 2 new processes of 2 total for sasl_sup, connected.

3> revolver:pid(sasl_pool).
<0.38.0>
```

Everything fine so far. Now we stop sasl (the pool):

```erlang
4>  application:stop(sasl).
ok

=INFO REPORT==== 2-Oct-2014::09:55:53 ===
revolver: The process <0.41.0> (child of sasl_sup) died.

=ERROR REPORT==== 2-Oct-2014::09:55:53 ===
revolver: Reloading children from supervisor sasl_sup.

=ERROR REPORT==== 2-Oct-2014::09:55:53 ===
revolver_utils: Supervisor sasl_sup not running, disconnected.

=ERROR REPORT==== 2-Oct-2014::09:55:53 ===
revolver trying to reconnecty in 10000 ms.

=INFO REPORT==== 2-Oct-2014::09:55:53 ===
    application: sasl
    exited: stopped
    type: temporary
5>  revolver:pid(sasl_pool).
{error,disconnected}
```

So revolver is disconnected and says so if you ask for a pid.
As soon as the pool is there again, revolver reconnects.

```erlang
6>  application:start(sasl).
ok
=INFO REPORT==== 2-Oct-2014::09:56:13 ===
revolver: Found 2 new processes of 2 total for sasl_sup, connected.

7>  revolver:pid(sasl_pool).
<0.58.0>
```



# Tests

```./rebar eunit skip_deps=true```