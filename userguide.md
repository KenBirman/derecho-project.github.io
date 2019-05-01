# A Brief User Guide

## Installation
Derecho is a library that helps you build replicated, fault-tolerant services in a datacenter with RDMA networking. Here's how to start using it in your projects.

### Prerequisites
* Linux (other operating systems don't currently support the RDMA features we use)
* A C++ compiler supporting C++17: GCC 7.3+ or Clang 7+
* The following system libraries: `rdmacm` (packaged for Ubuntu as `librdmacm-dev 1.0.21`), and `ibverbs` (packaged for Ubuntu as `libibverbs-dev 1.1.8`).
* CMake 2.8.1 or newer, if you want to use the bundled build scripts

### A Few Choices
On download, you are actually being given a Derecho configuration file that includes some default values we've selected for convenient development rather than for optimal deployment performance.  Here are some of the configuration defaults to consider changing (you'll get to hear details about how to change them later)
* You need to tell Derecho where the leader who will restart the group is running: "DERECHO/leader_ip=127.0.0.1".  Otherwise your application won't start because the first process you launch will think of itself as the leader (this IP number means "localhost") and other processes won't be able to connect to it!  We also recommend pinging from each machine where you will run a process to each machine where you have some other member.  Often people don't realize that Linux has built-in, pre-enabled firewalls and that things like ping won't necessarily break through those by default.  You may need to disable the firewall or punch a hole in it for the Derecho port ("DERECHO/leader_gms_port=23580").  Otherwise, ping could work and yet Derecho still wouldn't be able to communicate.
* TCP rather than RDMA.  Everyone has TCP but not every machine has RDMA-enabled NICs, and even when you have the NICs, you may not have permission to use RDMA.  So Derecho out of the box is set up to use TCP, even though this is actually quite slow compared to RDMA.  You would need to change provider from the default ("RDMA/provider=sockets", which is a confusing value, but standard for the MPI folks) to "RDMA/provider=verbs" or whatever your hardware supports (see the list in our configuration file itself, which has many options).  We actually prefer "TCP" to "sockets" and we don't know anyone who really thinks "verbs" is an obvious way to switch to using RDMA hardware, but if we don't use the standard terminology, our file would be inconsistent with LibFabric.  
* Buffer sizes.  Derecho needs some hints about the expected sizes of messages you will send, which will shape its memory footprint.  Since many people prefer that our runtime images be small, those defaults are small (DERECHO/max_payload_size=10240; if you go over this, we give an error and don't sent the message).  Obviously, if you plan to multicast videos at RDMA line speeds, you will need to crank these parameters to the limit!
* During development, many people play with small groups that might have just 2 members, or with toy setups where a group might have perhaps 4 members, even if they are thinking of a future setup with hundreds.  Then they kill 1 of the 2, or 2 of the 4, as a test.  The problem this creates is that in Derecho's logic, you've created a potential "split brain"!  If a network link between 2 nodes fails, we could end up with 2 air traffic controllers managing the same plane or runway, which is obviously unsafe.  So the question arises: should the system shut down if a group of size K splits into two subgroups of size K/2?  Or should we just assume you were testing and provoked this by killing a few members?  By default, Derecho assumes you were in test mode!  You need to tell it to enable the sensing of minority partitions, by setting a configuration parameter (DERECHO/disable_partitioning_safety=false; by default, it will be true).  But if you do, those same tests would cause the surviving half of your system to shut down, because 1 might not be a majority of 2, and 2 might not be a majority of 4, etc.  
* Derecho groups either use multicast, or are persistent, and we figure out which you intended automatically.  If we see that you have a field of type persistent<T> then we will treat the whole shard or subgroup as persistent.  If you don't have any such field, we will just assume that all your data is held in memory.  Moreover, persistent data is automatically versioned.  Be aware that this version log isn't stored in any random place: the persisted data will be held in a hidden folder in the local file system where your application launches ("PERS/file_path=.plog").  If you don't override this, we will create that directory for you, but you might not realize that we have been putting data in it.  There is also an option "PERS/reset=false", meaning that we will reload saved data when the system restarts.  Change this to true if you want to debug and don't want to delete the files manually every time.  In some of our experiments we change the path to be on "/dev/shm/volatile_t", and this forces the system to use an in-memory file system Linux supports for higher speed storage.  But as the name suggests, this path isn't really persistent across shutdowns.

### Getting Started
Since this repository uses Git submodules to refer to some bundled dependencies, a simple `git clone` will not actually download all the code. To download a complete copy of the project, run

    git clone --recursive https://github.com/Derecho-Project/derecho.git

Once cloning is complete, to compile the code, `cd` into the `derecho` directory and run:
* mkdir Release
* cd Release
* `cmake -DCMAKE_BUILD_TYPE=Release ..`
* `make`

This will place the binaries and libraries in the sub-dierectories of `Release`.
The other build type is Debug. If you need to build the Debug version, replace Release by Debug in the above instructions. We explicitly disable in-source build, so running `cmake .` in `derecho` will not work.

To add your own executable (that uses Derecho) to the build system, simply add an executable target to CMakeLists.txt with `derecho` as a "linked library." You can do this either in the top-level CMakeLists.txt or in the CMakeLists.txt inside the "applications" directory. It will look something like this:

    add_executable(my_project_main my_project_main.cpp)
	target_link_libraries(my_project_main derecho)

To use Derecho in your code, you simply need to 
- include the header `derecho/derecho.h` in your \*.h or \*.cpp files, and
- specify a configuration file, either by setting environment variable `DERECHO_CONF_FILE` or by placing a file named `derecho.cfg` in the working directory. A sample configuration file along with an explanation can be found in `conf/derecho-default.cfg`. 

The configuration file consists of three sections: **DERECHO**, **RDMA**, and **PERS**. The **DERECHO** section includes core configuration options for a Derecho instance, which every application will need to customize. The **RDMA** section includes options for RDMA hardware specifications. The **PERS** section allows you to customize the persistent layer's behavior. 

#### Configuring Core Derecho
Applications need to tell the Derecho library which node is the initial leader with the options **leader_ip** and **leader_gms_port**. Each node then specifies its own ID (**local_id**) and the IP address and ports it will use for Derecho component services (**local_ip**, **gms_port**, **rpc_port**, **sst_port**, and **rdmc_port**). 

The other important parameters are the message sizes. Since Derecho pre-allocates buffers for RDMA communication, each application should decide on an optimal buffer size based on the amount of data it expects to send at once. If the buffer size is much larger than the messages an application actually sends, Derecho will pin a lot of memory and leave it underutilized. If the buffer size is smaller than the application's actual message size, it will have to split messages into segments before sending them, causing unnecessary overhead.

There are three options to control the size of messages: **max_payload_size**, **max_smc_payload_size**, and **block_size**.
No message bigger than **max_payload_size** will be sent by Derecho. Messages equal to or smaller than **max_smc_payload_size** will be sent through SST multicast (SMC), which is more suitable than RDMC for small messages. **block_size** defines the size of unit sent in RDMC (messages bigger than **block_size** will be split internally and sent in a pipeline).

Please refer to the comments in [the default configuration file](https://github.com/Derecho-Project/derecho/blob/master/conf/derecho-default.cfg) for more explanations on **window_size**, **timeout_ms**, and **rdmc_send_algorithm**.

#### Configuring RDMA Devices
The most important configuration entries in this section are **provider** and **domain**. The **provider** option specifies the type of RDMA device (i.e. a class of hardware) and the **domain** option specifies the device (i.e. a specific NIC or network interface). This [Libfabric document](https://www.slideshare.net/seanhefty/ofi-overview) explains the details of those concepts.

The **tx_depth** and **rx_depth** configure the maximum of pending requests that can be waiting for acknowledgement before communication blocks. Those numbers can be different from one device to another. We recommend setting them as large as possible.

Here are some sample configurations showing how Derecho might be configured for two common types of hardware.

**Configuration 1**: run Derecho over TCP/IP with Ethernet interface 'eth0':

```
...
[RDMA]
provider = sockets
domain = eth0
tx_depth = 256
rx_depth = 256
...
```

**Configuration 2**: run Derecho over verbs RDMA with RDMA device 'mlx5_0':

```
...
[RDMA]
provider = verbs
domain = mlx5_0
tx_depth = 4096
rx_depth = 4096
...
```


#### Configuring Persistent Behavior
The application can specify the location for persistent state in the file system with **file_path**, which defaults to the `.plog` folder in the working directory. **ramdisk_path** controls the location of states for `Volatile<T>`, which defaults to tmpfs (ramdisk). **reset** controls weather to clean up the persisted state when a Derecho service shuts down. We default this to true. **Please set `reset` to `false` for normal use of `Persistent<T>`.**

#### Specify Configuration with Command Line Arguments
We also allow applications to specify configuration options on the command line. Any command line configuration options override the equivalent option in configuration file. To use this feature while still accepting application-specific command-line arguments, we suggest using the following code:

```cpp
#define NUM_OF_APP_ARGS () // specify the number of application arguments.
int main(int argc, char* argv[]) {
    if((argc < (NUM_OF_APP_ARGS+1)) || 
       ((argc > (NUM_OF_APP_ARGS+1)) && strcmp("--", argv[argc - NUM_OF_APP_ARGS - 1]))) {
        cout << "Invalid command line arguments." << endl;
        cout << "USAGE:" << argv[0] << "[ derecho-config-list -- ] application-argument-list" << endl;
        return -1;
    }
    Conf::initialize(argc, argv); // pick up configurations in the command line list
    // pick up the application argument list and continue ...
    ...
}
```
Then, call the application as follows, assuming the application's name is `app`:

```bash
$ app --DERECHO/local_id=0 --PERS/reset=false -- <application-argument-list>
```
Please refer to the [bandwidth_test](https://github.com/Derecho-Project/derecho/blob/master/applications/tests/performance_tests/bandwidth_test.cpp) application for more details.

### Setup and Testing
There are some sample programs in the folder applications/demos that can be run to test the installation. In addition, there are some performance tests in the folder applications/tests/performance\_tests that you may want to use to measure the performance Derecho achieves on your system. To be able to run the tests, you need a minimum of two machines connected by RDMA. The RDMA devices on the machines should be active. In addition, you need to run the following commands to install and load the required kernel modules:

```
sudo apt-get install rdmacm-utils rdmacm-utils librdmacm-dev libibverbs-dev ibutils libmlx4-1
infiniband-diags libmthca-dev opensm ibverbs-utils libibverbs1 libibcm1 libibcommon1
sudo modprobe -a rdma_cm ib_uverbs ib_umad ib_ipoib mlx4_ib iw_cxgb3 iw_cxgb4 iw_nes iw_c2 ib_mthca
```
Depending on your system, some of the modules might not load which is fine.

RDMA requires memory pinning of memory regions shared with other nodes. There's a limit on the maximum amount of memory a process can pin, typically 64 KB, which Derecho easily exceeds. Therefore, you need to set this to unlimited. To do so, append the following lines to /etc/security/limits.conf:
* `[username] hard memlock unlimited`
* `[username] soft memlock unlimited`

where `[username]` is your linux username. A `*` in place of the username will set this limit to unlimited for all users. Log out and back in again for the limits to reapply. You can test this by verifying that `ulimit -l` outputs `unlimited` in bash.

The persistence layer of Derecho stores durable logs of updates in memory-mapped files. Linux also limits the size of memory-mapped files to a small size that Derecho usually exceeds, so you will need to set the system parameter `vm.overcommit_memory` to `1` for persistence to work. To do this, run the command

    sysctl -w vm.overcommit_memory = 1


We currently do not have a systematic way of asking the user for RDMA device configuration. So, we pick an arbitrary RDMA device in functions `resources_create` in `sst/verbs.cpp` and `verbs_initialize` in `rdmc/verbs_helper.cpp`. Look for the loop `for(i = 1; i < num_devices; i++)`. If you have a single RDMA device, most likely you want to start `i` from `0`. If you have multiple devices, you want to start `i` from the order (zero-based) of the device you want to use in the list of devices obtained by running `ibv_devices` in bash.

A simple test to see if your setup is working is to run the test `bandwidth_test` from applications/tests/performance\_tests. To run it, go to two of your machines (nodes), `cd` to `Release/applications/tests/performance_tests` and run `./bandwidth_test 0 10000 15 1000 1 0` on both. The programs will ask for input.
The input to the first node is:
* 0 (its node ID)
* 2 (number of nodes for the experiment)
* IP address of node 1
* IP address of node 2

Replace the node ID 0 by 1 for the input to the second node.
As a confirmation that the experiment finished successfully, the first node will write a log of the result in the file `data_derecho_bw`, which will be something along the lines of `12 0 10000 15 1000 1 0 0.37282`. Full experiment details including explanation of the arguments, results and methodology is explained in the source documentation for this program.

## Using Derecho
The file `simple_replicated_objects.cpp` within applications/demos shows a complete working example of a program that sets up and uses a Derecho group with several Replicated Objects. You can read through that file if you prefer to learn by example, or read on for an explanation of how to use various features of Derecho.

### Replicated Objects
One of the core building blocks of Derecho is the concept of a Replicated Object. This provides a simple way for you to define state that is replicated among several machines and a set of RPC functions that operate on that state.

A Replicated Object is any class that (1) is serializable with the mutils-serialization framework and (2) implements a static method called `register_functions()`. The [mutils-serialization](https://github.com/mpmilano/mutils-serialization) library should have more documentation on making objects serializable, but the most straightforward way is to inherit `mutils::ByteRepresentable`, use the macro `DEFAULT_SERIALIZATION_SUPPORT`, and write an element-by-element constructor. The `register_functions()` method is how your class specifies to Derecho which of its methods should be converted to RPC functions and what their numeric "function tags" should be. It should return a `std::tuple` containing a pointer to each RPC-callable method, wrapped in the template function `derecho::rpc::tag`, whose template parameter is an integer constant. We have provided a default implementation of this function with the macro `REGISTER_RPC_FUNCTIONS`, which registers each method in its argument using the integer constant generated by the macro `RPC_NAME`. Here is an example of a Replicated Object declaration that uses the default implementation macros: 

```cpp
class Cache : public mutils::ByteRepresentable {
    std::map<std::string, std::string> cache_map;

public:
    void put(const std::string& key, const std::string& value);
    std::string get(const std::string& key); 
    bool contains(const std::string& key);
    bool invalidate(const std::string& key);
    Cache() : cache_map() {}
    Cache(const std::map<std::string, std::string>& cache_map) : cache_map(cache_map) {}
    DEFAULT_SERIALIZATION_SUPPORT(Cache, cache_map);
    REGISTER_RPC_FUNCTIONS(Cache, put, get, contains, invalidate);
};
```

This object has one field, `cache_map`, so the DEFAULT\_SERIALIZATION\_SUPPORT macro is called with the name of the class and the name of this field. The second constructor, which initializes the field from a parameter of the same type, is required for serialization support. The object has four RPC methods, `put`, `get`, `contains`, and `invalidate`, so the REGISTER\_RPC\_FUNCTIONS macro is called with the name of the class and the names of these methods. When these RPC functions are called, they will be identified with the tags `RPC_NAME(put)`, `RPC_NAME(get)` `RPC_NAME(contains)`, and `RPC_NAME(invalidate)`.

### Groups and Subgroups

Derecho organizes nodes (machines or processes in a system) into Groups, which can then be divided into subgroups and shards. Any member of a Group can communicate with any other member, and all run the same group-management service that handles failures and accepts new members. Subgroups, which are any subset of the nodes in a Group, correspond to Replicated Objects; each subgroup replicates the state of a Replicated Object and any member of the subgroup can handle RPC calls on that object. Shards are disjoint subsets of a subgroup that each maintain their own state, so one subgroup can replicate multiple instances of the same type of Replicated Object. A Group must be statically configured with the types of Replicated Objects it can support, but the number of subgroups and their exact membership can change at runtime according to functions that you provide. 

Note that more than one subgroup can use the same type of Replicated Object, so there can be multiple independent instances of a Replicated Object in a Group even if those subgroups are not sharded. A subgroup is usually identified by the type of Replicated Object it implements and an integral index number specifying which subgroup of that type it is. 

To start using Derecho, a process must either start or join a Group by constructing an instance of `derecho::Group`, which then provides the interface for interacting with other nodes in the Group. (The difference between starting and joining a group is simply a matter of calling a different constructor). A `derecho::Group` expects a set of variadic template parameters representing the types of Replicated Objects that it can support in its subgroups. For example, this declaration is a pointer to a Group object that can have subgroups of type LoadBalancer, Cache, and Storage:

```cpp
std::unique_ptr<derecho::Group<LoadBalancer, Cache, Storage>> group;
```

#### Defining Subgroup Membership

In order to start or join a Group, all members (including processes that join later) must define a function that provides the membership (as a subset of the current View) for each subgroup. The membership function's input is the type of Replicated Object associated with a subgroup, the current View, and the previous View if there was one. Since there can be more than one subgroup that implements the same Replicated Object type (as separate instances of the same type of object), the return type of the function is a vector-of-vectors: the index of the outer vector identifies which subgroup is being described, and the inner vector contains an entry for each shard of that subgroup. For backwards-compatibility reasons, the membership function must be wrapped in a struct called SubgroupInfo before being passed into the Group constructor.

Derecho provides a default subgroup membership function that automatically assigns nodes from the Group into disjoint subgroups and shards, given a policy that describes the desired number of nodes in each subgroup/shard. It assigns nodes in ascending rank order, and leaves any "extra" nodes (not needed to fully populate all subgroups) at the end (highest rank) of the membership list. During a View change, this function attempts to preserve the correct number of nodes in each shard without re-assigning any nodes to new roles. It does this by copying the subgroup membership from the previous View as much as possible, and assigning idle nodes from the end of the Group's membership list to replace failed members of subgroups.

There are several helper functions in `subgroup_functions.h` that construct AllocationPolicy objects for different scenarios, to make it easier to set up the default subgroup membership function. Here is an example of how the default membership function could be configured for two types of Replicated Objects using these functions:

```cpp
derecho::SubgroupInfo subgroup_function {derecho::DefaultSubgroupAllocator({
    {std::type_index(typeid(Foo)), derecho::one_subgroup_policy(derecho::even_sharding_policy(2,3))},
    {std::type_index(typeid(Bar)), derecho::identical_subgroups_policy(
            2, derecho::even_sharding_policy(1,3))}
})};
```
Based on the policies constructed for the constructor argument of DefaultSubgroupAllocator, when the function is called for subgroup Foo, it will create one subgroup, with two shards of 3 members each. When the function is called for subgroup Bar, it will create two subgroups of type Bar, each of which has only one shard of size 3. Note that the order in which subgroups are allocated is the order in which their Replicated Object types are listed in the Group's template parameters, so this instance of the default subgroup allocator will assign the first 6 nodes to the Foo subgroup and the second 6 nodes to the Bar subgroups the first time it runs.

More advanced users may, of course, want to define their own subgroup membership functions. The demo program `simple_replicated_objects.cpp` shows a relatively simple example of a user-defined membership function. In this program, the SubgroupInfo contains a C++ lambda function that implements the `shard_view_generator_t` type signature and handles subgroup assignment for Replicated Objects of type Foo, Bar, and Cache:

```cpp
[](const std::type_index& subgroup_type,
   const std::unique_ptr<derecho::View>& prev_view, derecho::View& curr_view) {
    if(subgroup_type == std::type_index(typeid(Foo)) || subgroup_type == std::type_index(typeid(Bar))) {
        if(curr_view.num_members < 3) {
            throw derecho::subgroup_provisioning_exception();
        }
        derecho::subgroup_shard_layout_t subgroup_vector(1);
        std::vector<node_id_t> first_3_nodes(&curr_view.members[0], &curr_view.members[0] + 3);
        subgroup_vector[0].emplace_back(curr_view.make_subview(first_3_nodes));
        curr_view.next_unassigned_rank = std::max(curr_view.next_unassigned_rank, 3);
        return subgroup_vector;
    } else { //subgroup_type == std::type_index(typeid(Cache))
        if(curr_view.num_members < 6) {
            throw derecho::subgroup_provisioning_exception();
        }
        derecho::subgroup_shard_layout_t subgroup_vector(1);
        std::vector<node_id_t> next_3_nodes(&curr_view.members[3], &curr_view.members[3] + 3);
        subgroup_vector[0].emplace_back(curr_view.make_subview(next_3_nodes));
        curr_view.next_unassigned_rank += 3;
        return subgroup_vector;
    }
};
```
For all three types of Replicated Object, the function creates one subgroup and one shard. For the Foo and Bar subgroups, it assigns first three nodes in the current View's members list (thus, these subgroups are co-resident on the same three nodes), while for the Cache subgroup it assigns nodes 3 to 6 on the current View's members list. Note that if there are not enough members in the current view to assign 3 nodes to each subgroup, the function throws `derecho::subgroup_provisioning_exception`. This is how subgroup membership functions indicate to the view management logic that a view has suffered too many failures to continue executing (it is "inadequately provisioned") and must wait for more members to join before accepting any more state updates.


#### Constructing a Group

Although the subgroup allocation function is the most important part of constructing a `derecho::Group`, it requires a few additional parameters.
* A set of **callback functions** that will be notified when each Derecho message is delivered to the node (stability callback) or persisted to disk (persistence callback). These can be null, and are probably not useful if you're only using the Replicated Object features of Derecho (since the "messages" will be serialized RPC calls).
* A set of **View upcalls** that will be notified when the group experiences a View change event (nodes fail or join the group). This is optional and can be empty, but it can be useful for adding additional failure-handling or load-balancing behavior to your application.
* For each template parameter in the type of `derecho::Group`, its constructor will expect an additional argument of type `derecho::Factory`, which is a function or functor that constructs instances of the Replicated Object (it's just an alias for `std::function<std::unique_ptr<T>(void)>`). 

### Invoking RPC Functions

Once a process has joined a Group and one or more subgroups, it can invoke RPC functions on any of the Replicated Objects in the Group. The options a process has for invoking RPC functions depend on its membership status:

* A node can perform an **ordered send** to invoke an RPC function on a Replicated Object only when it is a member of that object's subgroup and shard. This sends a multicast to all other members of the object's shard, and guarantees that the multicast will be delivered in order (so the function call will take effect at the same time on every node). An ordered send waits for responses from each member of the shard (specifically, it provides a set of Future objects that can be used to wait for responses), unless the function invoked had a return type of `void`, in which case there are no responses to wait for.
* A node can perform a **P2P query** or **P2P send** to invoke a read-only RPC function on any Replicated Object, regardless of whether it is a member of that object's subgroup and shard. P2P send is used for `void`-returning functions, while P2P query is used for non-`void` functions. These peer-to-peer operations send a message directly to a specific node, and it is up to the sender to pick a node that is a member of the desired object's shard. They cannot be used for mutative RPC function calls because they do not guarantee ordering of the message with respect to any other (peer-to-peer or "ordered") message, and could only update one replica at a time.

Ordered sends are invoked through the `Replicated` interface, whose template parameter is the type of the Replicated Object it communicates with. You can obtain a `Replicated` by using Group's `get_subgroup` method, which uses a template parameter to specify the type of the Replicated Object and an integer argument to specify which subgroup of that type (remember that more than one subgroup can implement the same type of Replicated Object). For example, this code retrieves the Replicated object corresponding to the second subgroup of type Cache:

```cpp
Replicated<Cache>& cache_rpc_handle = group->get_subgroup<Cache>(1);
```

The `ordered_send` method uses its template parameter, which is an integral "function tag," to specify which RPC function it will invoke; if you are using the `REGISTER_RPC_FUNCTIONS` macro, the function tag will be the integer generated by the `RPC_NAME` macro applied to the name of the function. Its arguments are the arguments that will be passed to the RPC function call, and it returns an instance of `derecho::rpc::QueryResults` with a template parameter equal to the return type of the RPC function. Using the Cache example from earlier, this is what RPC calls to the "put" and "contains" functions would look like:

```cpp
cache_rpc_handle.ordered_send<RPC_NAME(put)>("Foo", "Bar");
derecho::rpc::QueryResults<bool> results = cache_rpc_handle.ordered_send<RPC_NAME(contains)>("Foo");
```

P2P (peer-to-peer) sends and queries are invoked through the `ExternalCaller` interface, which is exactly like the `Replicated` interface except that it only provides the `p2p_send` and `p2p_query` functions. ExternalCaller objects are provided through the `get_nonmember_subgroup` method of Group, which works exactly like `get_subgroup` (except for the assumption that the caller is not a member of the requested subgroup). For example, this is how a process that is not a member of the second Cache-type subgroup would get an ExternalCaller to that subgroup:

```cpp
ExternalCaller<Cache>& p2p_cache_handle = group->get_nonmember_subgroup<Cache>(1);
```

When invoking a P2P send or query, the caller must specify, as the first argument, the ID of the node to communicate with. The caller must ensure that this node is actually a member of the subgroup that the ExternalCaller targets (though it can be in any shard of that subgroup). Nodes can find out the current membership of a subgroup by calling the `get_subgroup_members` method on the Group, which uses the same template parameter and argument as `get_subgroup` to select a subgroup by type and index. For example, assuming Cache subgroups are not sharded, this is how a non-member process could make a call to `get`, targeting the first node in the second subgroup of type Cache:

```cpp
std::vector<node_id_t> cache_members = group.get_subgroup_members<Cache>(1)[0];
derecho::rpc::QueryResults<std::string> results = p2p_cache_handle.p2p_query<RPC_NAME(get)>(cache_members[0], "Foo");
```

#### Using QueryResults objects

The result of an ordered query is a slightly complex object, because it must contain a `std::future` for each member of the subgroup, but the membership of the subgroup might change during the query invocation. Thus, a QueryResults object is actually itself a future, which is fulfilled with a map from node IDs to futures as soon as Derecho can guarantee that the query will be delivered in a particular View. (The node IDs in the map are the members of the subgroup in that View). Each `std::future` in the map will be fulfilled with either the response from that node or a `node_removed_from_group_exception`, if a View change occurred after the query was delivered but before that node had a chance to respond.

As an example, this code waits for the responses from each node and combines them to ensure that all replicas agree on an item's presence in the cache:

```cpp
derecho::rpc::QueryResults<bool> results = cache_rpc_handle.ordered_query<RPC_NAME(contains)>("Stuff");
bool contains_accum = true;
for(auto& reply_pair : results.get()) {
    bool contains_result = reply_pair.second.get();
    contains_accum = contains_accum && contains_result;
}
```

Note that the type of `reply_pair` is `std::pair<derecho::node_id_t, std::future<bool>>`, which is why a node's response is accessed by writing `reply_pair.second.get()`.

### Tracking Updates with Version Vectors

Derecho allows tracking data update history with a version vector in memory or persistent storage. A new class template is introduced for this purpose: `Persistent<T,ST>`. In a Persistent instance, data is managed in an in-memory object of type T (we call it the "current object") along with a log in a datastore specified by storage type ST. The log can be indexed using a version number, an index, or a timestamp. A version number is a 64-bit integer attached to each version; it is managed by the Derecho SST and guaranteed to be monotonic. A log is also an array of versions accessible using zero-based indices. Each log entry also has an attached timestamp (microseconds) indicating when this update happened according to the local real-time clock. To enable this feature, we need to manage the data in a serializable object T, and define a member of type Persistent<T> in the Replicated Object in a relevant group. Persistent\_typed\_subgroup_test.cpp gives an example.

```cpp
/**
 * Example for replicated object with Persistent<T>
 */
class PFoo : public mutils::ByteRepresentable {
    Persistent<int> pint;
public:
    virtual ~PFoo() noexcept (true) {}
    int read_state() {
        return *pint; 
    }
    bool change_state(int new_int) {
         if(new_int == *pint) {
           return false;
         }

         *pint = new_int;
         return true;
    }
     
    // constructor with PersistentRegistry
    PFoo(PersistentRegistry * pr) : pint(nullptr,pr) {}
    PFoo(Persistent<int> & init_pint) : pint(std::move(init_pint)) {}
    DEFAULT_SERIALIZATION_SUPPORT(PFoo, pint);
    REGISTER_RPC_FUNCTIONS(PFoo, read_state, change_state);
};
```
	
For simplicity, the versioned type is int in this example. You set it up in the same way as a non-versioned member of a replicated object, except that you need to pass the PersistentRegistry from the constructor of the replicated object to the constructor of the `Persistent<T>`. Derecho uses PersistentRegistry to keep track of all the Persistent<T> objects in a single Replicated Object so that it can create versions on updates. The Persistent<T> constructor registers itself in the registry.

By default, the Persistent<T> stores its log in the filesystem (in a folder called .plog in the current directory). Application can specify memory as the storage location by setting the second template parameter: `Persistent<T,ST_MEM>` (or `Volatile<T>` as syntactic sugar). We are working on more store storage types including NVM.

Once the version vector is set up with Derecho, the application can query the value with the get() APIs in Persistent<T>. In [persistent_temporal_query_test.cpp](https://github.com/Derecho-Project/derecho/blob/master/derecho/experiments/persistent_temporal_query_test.cpp), a temporal query example is illustrated.

