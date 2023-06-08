# BismuthScorpion
This is a proof of concept java virus, which infects jars contained in the same directory as itself, and incorporates
a few novel java anti-analysis techniques.

## Self-replication in Java, prior and post
The obvious caveat is that this is what I'm aware of. I did a little research when I started out with Bismuth and
couldn't find anything, but I missed Strangebrew, Cheshire's author possibly missed me, and Neko's author did
completely their own thing (but probably won't be willing to answer our questions, even if I knew who to ask).

* [Strangebrew](http://virus.wikidot.com/strangebrew) (1998) - Proof of concept Java virus, samples seem to have been lost to time
* BismuthScorpion (2019)
* Cheshire (2022) - [source](https://git.blackmarble.sh/backup/MalwareSourceCode/-/tree/main/Java/Virus.Java.Cheshire.a) [slides](https://github.com/mgrube/recon_22/blob/main/Samsara_Recon.pdf) - Self-contained example presented at REcon 2022 
* Neko (2023) - [source](https://github.com/clrxbl/NekoClient) - Decompilation of an unobfuscated (apparently uploaded by the attacker by accident lol) version of the final stage of malware involved in an interesting incident affecting Minecraft players. The malware doesn't inject itself, but rather a tiny stub which loads a classfile from a URL. Elegant, fragile, uses a library to handle the heavy lifting of injection, and as someone who has spent FAR TOO LONG staring at the java classfile docs, I resent this. The self-appointed incident responders seem single-mindedly dedicated to the task of failing to prevent this happening again.

## Countermeasures
* Ensure your jarfiles are read only (easy)
* Sign jarfiles, and [require signatures](https://blog.frankel.ch/jvm-security/2/) (hard)

## Licence
Creative Commons BY-NC-SA 4.0

## Mechanics overview
### run
This is the entry point. This returns if it's already been run by one of the many invocations from constructors.
This function iterates through jar files in the same directory, calling `infectJar` on them.

### infectJar
The jarfile is opened. Any signature files present are stripped (it's more likely they'll be present than mandatory), and this method returns
early if any filenames contain the infection signature, which consists of every odd-indexed character being 'w' or greater.

ActualScorpion is streamed into into a .class file in the root of the jar, with a random name complying with the above signature, through the `transformSelf`
function, which obfuscates the class, and updates it with its new name.

A random selection of classfiles present in the jar are streamed through `injectInvoke`, which inserts an invocation referencing
ActualScorpion's `run`, reverting the file if this method returns true, indicating a condition which would prevent classfile verification.

### injectInvoke
This inserts the relevant references to the obfuscated ActualScorpion's `run` into the constant pool, and prepends an invokestatic instruction
referencing this into the Code attribute of the first method (invariably `<init>`, the constructor). This is surrounded by bytecode which
frustrates some analysis tools if the classfile version if low enough that stack frame maps aren't mandatory on branches.

The exceptions table in `Code` is offset to accomodate the new bytecode, as are the `LocalVariableTable` and `LocalVariableTypeTable` attributes,
if present.

If a stack frame map is present, this method returns true, to indicate to `infectJar` that the file needs to be reverted, as it'll fail verification
without appropriate amendments to the stack frame map. Constructors aren't usually complex enough to require stack frame maps, so it's probably
not worthwhile to add this functionality.

### transformSelf

This changes ActualScorpion references to refer to the randomly generated name selected in `infectJar`, and applies obfuscation throughout.
The constant pool is loaded into a linked hashmap and written out at the end, in order to allow manipulation of strings informed by later
contents of the classfile.

## Obfuscation overview

### injectInvoke
* A second `Code` string is inserted into the constant pool, which `<init>` references. A tool which expects only the first instance of `Code`
in the constant pool to be used for a method's `Code` attribute will treat the constructor as an empty method.
* Constructor access flags are forced to synthetic, a flag intended to indicate members not present in the source. Some tools
omit synthetic members from their output

If the classfile version permits, invokestatic is supplemented with branches. Some tools misinterpret wide branches, which can be exploited by
using a wide jump into what would otherwise be dead code. The tools regard it as such, and optimise it out.

```
; A7 00 09 (+9)
goto first

second:

; B8 00 ?? ??
; The hidden invocation!
invokestatic ActualScorpion run ()V

; A7 00 08 (+8)
goto third

first:

; C8 FF FF FF FA (-6)
goto_w second

third:
```

### transformSelf
* The SourceFile string is set to a zalgoified string, containing `\033(0 \033[42;5;35m`, an escape sequence which puts a terminal into
graphical mode, with magenta text on a green background. This will be displayed on the terminal if an exception occurs anywhere in this file.
* All permitted access flags are set. This serves two functions: Making life harder for tools, and causing confusion to a human using them, given
the relative obscurity of some of these keywords
    - Class: final, synthetic
    - Field: volatile, transient, synthetic
    - Method: final, synchronized, bridge, strictfp, synthetic
* Method and field names (aside from `run`, `<init>` and `<clinit>`) are set to 65280 randomly selected mostly unprintable characters (`'\x01'`..`'-'`).
Some tools won't escape these, but most will escape unprintable characters in the `\uXXXX` form, a potential sixfold increase in member name length,
every time the member is referenced.
* Method `LocalVariableTable` and `LocalVariableTypeTable` attributes are changed to offer no useful hints as to the names of variables
* Method `LineNumberTable` attributes, which correspond bytecode to line numbers for debugging and analysis, are made less accurate
