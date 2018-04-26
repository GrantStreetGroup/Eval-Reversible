# NAME

Eval::Reversible - Evals with undo stacks

# SYNOPSIS

```perl
use Eval::Reversible;

my $reversible = Eval::Reversible->new(
    failure_warning => "Undoing actions..",
);

$reversible->run_reversibly(sub {
    # Do something with a side effect
    open my $fh, '>', '/tmp/file' or die;

    # Specify how that side effect can be undone
    # (assuming '/tmp/file' did not exist before)
    $reversible->add_undo(sub { close $fh; unlink '/tmp/file' });

    operation_that_might_die($fh);
    operation_that_might_get_SIGINTed($fh);

    close $fh;
    unlink '/tmp/file';

    $reversible->clear_undo;
    $reversible->failure_warning("Wasn't quite finished yet...");

    another_operation_that_might_die;
    $reversible->add_undo(sub { foobar; });

    $reversible->disarm;

    # This could die without an undo stack
    another_operation_that_might_die;

    $reversible->arm;

    # Previous undo stack back in play
});

# Alternative caller
Eval::Reversible->run_reversibly(sub {
    my $reversible = $_[0];

    $reversible->add_undo(...);
    ...
});

# Alternative function interface
reversibly {
    to_undo { ... };
    die;
} 'Failed to run code; undoing...';
```

# DESCRIPTION

Run code and automatically reverse their side effects if the code fails.  This is done by
way of an undo stack.  By calling ["add\_undo"](#add_undo) right after a side effect, the effect is
undone on the event that the ["run\_reversibly"](#run_reversibly) sub dies.  For example:

```perl
$reversible->run_reversibly(sub {
    print "hello\n";
    $reversible->add_undo(sub { print "goodbye\n" });
    die "uh oh\n" if $something_bad;
});
```

This prints "hello" if `$something_bad` is false.  If it's true, then both "hello" and
"goodbye" are printed and the exception "uh oh" is rethrown.

Upon failure, any code refs provided by calling ["add\_undo"](#add_undo) are executed in reverse
order.  Conceptually, we're unwinding the stack of side effects that `$code` performed
up to the point of failure.

# ATTRIBUTES

## failure\_warning

This is the message that will warn as soon as the operation failed.  After this, the undo
stack is unwound, and the exception is printed.  Default is no message.

## undo\_stack

The undo stack, managed in LIFO order, as an arrayref of coderefs.

This attribute has the following handles, which is what you should really interact with:

### add\_undo

Adds another coderef to the undo stack via push.

### pop\_undo

Removes the last coderef and returns it via pop.

### clear\_undo

Clears all undo coderefs from the stack.  Handy if the undo stack needs to be cleared out
early if a "point of no return" has been reached prior the end of the ["run\_reversibly"](#run_reversibly)
code block.  Alternatively, ["disarm"](#disarm) could be used, but it doesn't clear the existing
stack.

### is\_undo\_empty

Returns a boolean that indicates whether the undo stack is empty or not.

## armed

Boolean that controls if ["run\_reversibly"](#run_reversibly) code blocks will actually run the undo stack
upon failure.  Turned on by default, but this can be enabled and disabled at will before
or inside the code block.

Has the following handles:

### arm

Arms the undo stack.

### disarm

Disarms the undo stack.

# METHODS

## run\_reversibly

```
$reversible->run_reversibly($code);
Eval::Reversible->run_reversibly($code);
```

Executes a code reference (`$code`) allowing operations with side effects to be
automatically reversed if `$code` fails or is interrupted.  Automatically clears the
undo stack before the start of the code block.

Can be called as a class method, which will auto-create a new object, and pass it along
as the first parameter to `$code`.

If `$code` is interrupted with SIGINT, the side effects are undone and an
exception "SIGINT\\n" is thrown.

## run\_undo

```
$reversible->run_undo;
```

Runs the undo stack thus far.  Always runs the bottom of the stack first (LIFO order).  A
finished run will clear out the stack via pop.

Can be called outside of ["run\_reversibly"](#run_reversibly) if the eval was successful, but the undo
stack still needs to be ran.

# EXPORTABLE FUNCTIONS

Eval::Reversible also supports an exportable function interface.  Though its usage is
somewhat legacy, the functions are prototyped for reduced sigilla.

None of the functions are exported by default.

## reversibly

```
reversibly {
    ...
} 'Failure message';
```

Creates a new localized Eval::Reversible object and calls ["run\_reversibly"](#run_reversibly) on it.  An
optional failure message can be added to the end of coderef.

## to\_undo

```
# Only inside of a reversibly block
to_undo { rollback_everything };
```

Adds to the existing undo stack.  Dies if called outside of a ["reversibly"](#reversibly) block.

# SEE ALSO

[Scope::Guard](https://metacpan.org/pod/Scope::Guard), [Data::Transactional](https://metacpan.org/pod/Data::Transactional), [Object::Transaction](https://metacpan.org/pod/Object::Transaction).

# AUTHOR

Grant Street Group, `<developers@grantstreet.com>`

# COPYRIGHT & LICENSE

Copyright 2018 Grant Street Group, All Rights Reserved.
