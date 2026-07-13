A global resource abstraction where it appears that every process within the namespace have their own isolated instance of that resource. Only the processes inside that namespace can see changes to that resource. 

![[namespaces.png]]

Two critical behaviors of namespaces:
- Inheritance: Every new process that is created is a part of its parent's namespace. We can assign different namespaces for specific resource categories of the child process using the `unshare --namespace_type process` command.
- Lifecycle: A namespace only exists until something holds a reference to it. The most common reference is a process. If all processes in the namespace exit, then the kernel releases the namespace (unless something else is keeping a reference to it).
	### Are there exceptions?
	- Yes. A namespace can remain alive even with no running processes if **another kernel object holds a reference**.	
Examples:
```
Network Namespace
    ↑
 bind mount
```

- If a network namespace is bound to mount, even if every process exits, the network namespace still exists because the bind mount references it.

```zsh
touch /tmp/netns
mount --bind /proc/self/ns/net /tmp/netns
```

- The same idea applies to file descriptors. As long as that file descriptor remains open, the namespace cannot be destroyed

```C
fd = open("/proc/self/ns/uts", O_RDONLY);
```

# Observations:
- Namespaces are independent of each other
- For a process, we can set a different namespace corresponding a specific resource category, and as for all the other resource categories, it still depends on the parent's namespaces.

For e.g if we use

```zsh
unshare --uts processA
```

Then processA has a new UTS namespace for itself, but it's other resources depend on it's parent's namespace.