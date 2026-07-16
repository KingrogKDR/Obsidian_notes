A virtual filesystem view that is rather a routing layer of an underlying filesystem consisting of a single upper layer and one or many lower layers. The lower layer is read-only while any changes in the overlay filesystem is reflected on the upper layer.
# OverlayFS Objects

They can of two types:

- Directory
- Non-directory (files, symlinks, device node, etc.)

Directory report the overlay filesystem in its `st_dev` .

Non-directories report the underlying filesystem it actually belongs to, for e.g upper/lower. They are uniquely identified using `st_dev` + `st_ino` fields.

If in the special situation where all the overlay layers lie in the same filesystem, then:

`st_dev` = overlay filesystem

`st_ino` = underlying filesystem inode no. for that file

This way it is easier to differentiate between overlay filesystem objects and actual objects that belong to the underlying filesystem.

If the overlay layers do not exist in the same filesystem, we can use “xino” to make it appear that they are in the same filesystem but only on 64-bit systems.

The “xino” feature uses a unique identifier for a file using `st_ino` + `fsid` (id of the actual filesystem it belongs to).

The `fsid` is mainly stored in the upper bits of the inode number because they are rarely used by the underlying filesystem. If somehow they were to overflow into these high inode bits, then the xino behavior for that inode is disabled.

We can use `-o xino=on` alongside mount overlay command to enable xino behavior. To automate xino behavior, we can use `xino=auto` and based on persistent inode numbers, xino is either enabled or disabled.

# Upper and Lower

Upper is normally writable but can be read-only. Then the entire overlay filesystem becomes read-only.

The lower layer is always read-only.

They may or may not be in the same filesystem. Therefore it is apt to call them directory trees instead of filesystem.