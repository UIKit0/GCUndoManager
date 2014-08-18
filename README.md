GCUndoManager
=============

A reimplementation of NSUndoManager that is highly compatible with the original but much more debuggable.

Task Coalescing

GCUndoManager supports task coalescing, where a series of identical tasks within a group are collapsed to a single task. This can be disabled and is not enabled by default. There are two coalescing approaches available, set using -setCoalescingKind: The first kGCCoalesceLastTask is just to discard tasks based on the most recent one accepted. This is good for property changes consisting only of a single property, for example an object's location, that is repeatedly changed by a drag. Thus task sequences are coalesced as follows:

AAAAAA > A
ABBBBB > AB
ABBBBA > ABA
However, where property changes do not consist of a single property change per drag event, but have several parts, the simple coalescing behaviour will not be able to help, as:

ABABABAB > ABABABAB
For this kind of sequence, the second coalescing kind kGCCoalesceAllMatchingTasks could be used. This coalesces tasks based on the presence of any match within the group, not just the last one. This results in the following behaviour:

ABABABAB > AB
ABCABCABC > ABC
but ABBBBBA > AB
The last example shows that this mode is not as general purpose as the first. Applications can set the coalescing mode as they wish depending on how property changes are made during a repeated sequence. Or they can leave it in the first mode and incur some inefficiency for the second example cases. Note that coalescing is always performed with respect to the current open group, so can be 'restarted' by opening a subgroup.

Implementation details

Unlike NSUndoManager, GCUndoManager is not a 'black box'. Internally, it represents the recorded actions using two kinds of object, GCUndoGroup and GCConcreteUndoTask. Both are subclasses of the semi-abstract GCUndoTask class. GCConcreteUndoTask further stores the actual data change as an NSInvocation which it retains. In turn the invocation retains its arguments and target. Groups can be nested to any depth as with NSUndoManager. There are no special marker or sentinel objects used to demark the start and end of groups, everything is stored and managed as a straightforward tree. The Undo and Redo stacks themselves are NSMutableArray instances, and a group stores its contents also using a NSMutableArray. Like the NSUndoManager in 10.6, GCUndoManager uses a proxy object based on NSProxy that is returned by -prepareWithInvocationTarget: The proxy prevents the situation where a property defined by the undo manager itself can't be recorded because the undo manager will not forward methods it already responds to. While the use of the proxy can be conditionally compiled out, it is recommended and will work on any version of Mac OS. No API is private and internal operations are well factored to permit overriding anywhere that makes sense. You can peek at the current undo and redo tasks, get the stacks themselves, pop the tasks with and without invoking them, and many other things. Also for assistance with debugging complex undo groups consisting of a series of individual tasks, -explodeTopUndoAction will 'unpack' the current top-level group on the Undo stack into separate tasks which can be individually undone.

When a top level group is opened, it is immediately pushed onto the relevant stack. The data member 'mOpenGroupRef' tracks the currently open group, which might be nested within another if it is not a top-level group. All task recording is done with reference to this group. If the top-level group is empty when the top-level is closed, the empty group is popped and discarded, which addresses the empty group bug. Note that this automatic removal can be disabled - for applications that do not submit tasks to the undo manager but merely subscribe to notifications and manage their own undo stacks, disabling this would be appropriate. However, in that case replacing NSUndoManager may not be worthwhile. It is because GCUndoManager adds a top-level group to the stack when the group is opened rather than when it is closed that it is able to be far less finicky about strict balance, and makes recovery from an imbalance much easier.

Update, 1/1/2009

Updated GCUndoManager has now been tested with Core Data and has been found to work correctly, after some minor tweaks. This version changes its memory management policy for retaining of tasks' targets: as per NSUndoManager and general rules, GCUndoManager no longer retains its targets by default. The undo manager should not hold stale targets, because -removeAllActionsWithTarget: is required to be called whenever any such targets are deallocated. However, for some designs retaining targets may simplify the use of the undo manager quite considerably, so you can now opt-in to this behaviour using -setRetainsTargets:passing an argument of YES. When targets are retained, clearing the task stacks must avoid re-entrancy, and GCUndoManager now includes a simple lock to ensure that.

Update, 20/7/2011

Updated to include the new notification used by NSDocument in Lion (10.7). The source is now hosted on Github, so any further changes will be made to the repository there.
