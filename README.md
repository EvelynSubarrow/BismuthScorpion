# BismuthScorpion
This is a proof of concept java virus, which infects jars contained in the same directory as itself. I conducted some
research before starting on this, and I couldn't find any java self-replication examples at all. To my knowledge, this
is the first of its kind.

## Mechanics overview

BismuthScorpion iterates through jar files contained in the same directory as itself. It opens each jar, checking that it contains
no files starting with a `蠍` (Archaic Chinese for "Scorpion"). If matching files are found, it doesn't continue, in order to prevent
reinfection.

Any signature files present in the jar are stripped, and an obfuscated version of ActualScorpion is inserted into the jarfile. As part
of the obfuscation process, it's given a randomised name, starting with `蠍`. Most methods and fields are renamed as part of the
obfuscation process, but the entry point (run: ()V) is retained for the sake of simplicity.

A random selection of classfiles are manipulated. Relevant constant pool references to the obfuscated ActualScorpion's run ()V are
inserted, and an invokestatic instruction referencing these is inserted into the first method of the class (guaranteed to be `<init>`,
the class constructor). If the java version is low enough that stack map frames aren't mandatory on branches, additional instructions
are inserted before and after, in order to defeat some analysis tools. If a stack frame map is present, the classfile is reverted, as
BismuthScorpion cannot handle these, and the bytecode manipulation will cause the classfile to fail verification. Most constructors
aren't complex enough to require stack frame maps.
