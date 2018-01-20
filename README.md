# P5-modules-to-P6-porting-guide
Porting Guide for porting Pumpkin Perl 5 Modules to Rakudo Perl 6

# undef vs Nil

`Nil` is the closest thing that Perl 6 has to Perl 5's `undef`.  `Nil` is the
value that indicates the absence of a value.  If you assign `Nil` to a variable,
it will reset the variable to its default value.  If you haven't specified a
default value, and you haven't specified a type, it will set the variable to
`Any` upon assignment of Nil.

If you need to be able to pass a `Nil` value around, you **can** do this, but it
requires some care: you will need to either only bind to values that can have
a `Nil` value, or make sure the default value of the variable is `Nil`.

    my $a := frobnicate(42);                 # binding
    my \b  = frobnicate(666);                # no sigil means binding
    my $c is default(Nil) = frobnicate(747); # specific default

    my $d; $d = Nil; # Any

# Exporter
Perl 6 has an Exporter built in, so you don't need to `use` one.

The basic functionality is explained at
[Exporting and Selective Importing](https://docs.perl6.org/language/modules#Exporting_and_Selective_Importing)

## Don't want to export anything by default

If you don't want to export anything by default, then mark the subroutines with
`is export(:FOO)` where `FOO` is a string that has meaning to you as a developer.

    sub frobnicate($a) is export(:FOO) { ... }

By specifying a specific name, you're preventing the standard export logic to
export that sub if you don't specify anything in the `use` statement.

Outside of the scope of any class in the compilation unit, you must create a
subroutine named `EXPORT` that takes any positional arguments.  Something like:

    sub EXPORT(*@args) {
        if @args { 
            my $imports := Map.new( |(EXPORT::FOO::{ @args.map: '&' ~ * }:p) );
            if $imports != @args {   
                die "PACKAGENAME doesn't know how to export: "
                  ~ @args.grep( { !$imports{$_} } ).join(', ')
            }   
            $imports
        }   
        else {
            Map.new
        }   
    }

This `EXPORT` sub will be executed when a `use` of the file is done.  If it gets
any arguments, it will look if they're marked with "is export(:FOO)".  If not all
arguments were found, it means one is trying to import things the module doesn't
know about.  So let the world know it can't do that.  Otherwise return the `Map`
with the export information and let the system deal with it.

## Moose and friends (OO)

You don't need them. OO is builtin to Perl 6 to the extent that almost everything
is an object:

    class Geo::IP2Location::Lite

    has %!file is required;

You can also trivially define your own types:

    subset IPv4 of Str where / (\d ** 1..3) ** 4 % '.' /;

Then use them within method signatures:

    method get_country ( IPv4 $ip ) { ... }

View [Classes and Objects](https://docs.perl6.org/language/classtut) for much more
information about this

## pack / unpack

pack / unpack are still experimental in Perl 6, however we are able to get to the
native data types with NativeCall so can implement most of what we need:

    use NativeCall;

    method !read8 ( IO::Handle $handle, Int $position ) {
        $handle.seek($position-1, SeekFromBeginning);
        my $data = $handle.read(1);
        nativecast((int8), Blob.new($data));
    }

This is roughly equivalent to:

    sub read8 {
        my ($handle, $position) = @_;
        my $data = "";
        seek($handle, $position-1, 0);
        read($handle, $data, 1);
        return unpack("C", $data);
    }

## Tests

You will find the Test functions map quite nicely in Perl 6 to the Pumpkin Perl 5
modules, and most of the extra Test:: namespace functions are builtin to the Perl 6
[Test](https://docs.perl6.org/language/testing) class

    use Test;

    plan 10;

    ...

    for %ips.keys -> $k {
        is( $ip2.get_country_short( $k ),%ips{$k},"$k resolves to { %ips{$k} }" );
    }

Running them just requires the `--exec` argument to prove:

    /Volumes/code_partition/geo-ip2location-lite-p6 > prove --exec perl6 -Ilib -r
    ./t/004_exceptions.t ... ok
    ./t/005_lookup.t ....... ok
    ./t/010_lookup_full.t .. ok
    All tests successful.
    Files=3, Tests=35,  1 wallclock secs ( 0.02 usr  0.01 sys +  0.94 cusr  0.12 csys =  1.09 CPU)
    Result: PASS
