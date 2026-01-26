---
title: Specification of the UMB Format
date: 01/26/2026
description: Overall specification of the unified Markov binary format.
published: true
---
# Specification of the UMB Format

Models specified in the UMB ("unified Markov binary") format consist of a folder structure containing a set of files with well-defined names (containing characters `[a-z][0-9][-_]` plus a single `.` only) in well-defined locations. For transporting models, that folder structure is bundled in a tar file that is optionally compressed using gzip or xz. The file extension for both compressed and uncompressed tar files of this format is `.umb`. Tools can look at the magic bytes at the beginning of the file to detect the format: big-endian `75 73 74 61 72` at offset 257 for POSIX tar, `1F 8B 08` for gzip or `FD 37 7A 58 5A 00` for xz.

A detailed description of the individual files follows below. Overall, there is one central `index.json` file, which provides all metadata relevant for the model, e.g., its type, statistics regarding its size and what other information is attached (such as rewards, atomic propositions or state valuations). The file also acts an index for the binary files that make up the rest of the tar file.

---

## Index (JSON)

`/index.json` (mandatory)

This file should be the first entry in the tar file.  It is a UTF-8 encoded JSON file whose contents follow the `IndexFile` schema, specified in a format similar to [js-schema](https://github.com/molnarg/js-schema) here:

```javascript
var IndexFile = schema({  
  "format-version": Number.min(0).max(2^32-1).step(1), // major version of format, change breaks compatibility
  "format-revision": Number.min(0).max(2^32-1).step(1), // minor version of format, nothing that breaks compatibility (additions and such)

  // Model metadata  
  "?model-data": { // information about the model that this file represents  
    "?name": String, // the (short) name of the model  
    "?version": String, // information about the version of this model (e.g. the date when it was last modified)  
    "?authors": Array.of(String), // information about the creator of the model      
    "?description": String, // a description of the model  
    "?comment": String, // additional comments  
    "?doi": String, // the DOI of the paper where this model was introduced/used/described   
    "?url": String, // a URL pointing to more information about the model  
  },

  // File metadata  
  "?file-data": { // information about how this file was created (for reproducibility purposes)  
    "?tool": String, // the tool used to create this model (strongly encouraged)  
    "?tool-version": String, // the tool's version  
    "?creation-date": Number.min(0).max(2^64-1).step(1), // export date ([Unix timestamp](https://en.wikipedia.org/wiki/Unix_time))  
    "?parameters": Any // the tool parameters (e.g. string or list of command-line arguments) used  
  },

  "transition-system": {  
    // LTS:   discrete-time   1-player  branch probability type absent  
    // DTMC:  discrete-time   0-player  
    // MDP:   discrete-time   1-player  
    // CTMC:  stochastic-time 0-player  
    // MA:    urgent-st.-time 1-player  
    // CTMDP: stochastic-time 1-player  
    // POMDP: discrete-time   1-player  #observations > 0  
    // TG:    discrete-time   n-player  branch probability type absent  
    // TSG:   discrete-time   n-player  
    // Add "I" in front of acronym to make probabilities=interval-probabilistic  
    // Excluded for being symbolic: TA, HA, PTA, PHA, STA, SHA  
    // Excluded for having complicated actions: CSG  
    // Both of the above: We know how to add  
    "time": [ "discrete", "stochastic", "urgent-stochastic" ],  
    "#players": Number.min(0).max(2^32-1).step(1), // 0: DTMC or CTMC, 1: LTS, MDP or MA, more: TSG  
    "#states": Number.min(1).max(2^64-1).step(1), // number of states  
    "#initial-states": Number.min(0).max(2^64-1).step(1), // number of initial states  
    "#choices": Number.min(0).max(2^64-1).step(1), // number of choices  
    "#choice-actions": Number.min(0).max(2^32-1).step(1), // number of choice actions (not all have to be used)
    "#branches": Number.min(0).max(2^64-1).step(1), // number of branches  
    "#branch-actions": Number.min(0).max(2^32-1).step(1), // number of branch actions (not all have to be used)  
    "#observations": Number.min(0).max(2^64-1).step(1), // number of observation indices (0 means no observations)  
    "?observations-apply-to": [ "states", "branches" ] // must be present iff #observations is non-zero

    // The types in branch-probability-type, exit-rate-type, and observation-probability-type must be continuous numeric types, and must be of the default size (given explicitly as the value for size or implicitly by omitting size) except for types "rational" and "rational-interval", for which the size must be a positive multiple of 128 and 256, respectively  
    "?branch-probability-type": Type, // must be present iff system has probabilities (e.g. DTMC, MDP)  
    "?exit-rate-type": Type, // must be present iff system has exit rates (e.g. CTMC, MA)  
    "?observation-probability-type": Type, // must be present iff the system has observations (i.e. #observations is not 0) and they are not deterministic

    "?player-names": Array.of(string) // must be of length #players if present; can be omitted if #players is less than 2; names must be unique  
  },

  "?annotations": schema(Map.of(  
    [ // Keys:  
      "rewards", // rewards (must have continuous numeric type), stored in /annotations/rewards/<identifier>/  
      "aps", // atomic propositions (must apply to states only and be of type bool), stored in /annotations/aps/<identifier>/  
      /[0-9a-z_-]+/ // stored in /annotations/<key>/<identifier>/  
    ],  
    AnnotationMap // values  
  )),

  "?valuations": { // for each annotated entity, we specify a list of valuation classes  
    "?states": ValuationDescription,  
    "?choices": ValuationDescription,  
    "?branches": ValuationDescription,  
    "?observations": ValuationDescription,  
    "?players": ValuationDescription,  
  }

} // end of IndexFile

var Type = schema({  
  "type": [  
    "bool", // Boolean: 0 is false, anything else is true; default size is 1  
    "int", // discrete numeric; signed integer (two's complement representation); default size is 64  
    "uint", // discrete numeric; unsigned integer; default size is 64  
    "int-interval", // discrete numeric, interval; size must be a multiple of 2 (for a size/2-bit int lower bound followed by a size/2-bit int upper bound); default size is 128  
    "uint-interval", // discrete numeric, interval; size must be a multiple of 2 (for a size/2-bit uint lower bound followed by a size/2-bit uint upper bound); default size is 128  
    "double", // continuous numeric; 64-bit IEEE 754 floating-point number; size is/must always be 64  
    "double-interval", // continuous numeric, interval; size is/must always be 128 (for a 64-bit double lower bound followed by a 64-bit double upper bound)  
    "rational", // continuous numeric; size must be a multiple of 2 (for a size/2-bit int numerator followed by a size/2-bit uint denominator); default size is 128  
    "rational-interval" // continuous numeric, interval; size must be a multiple of 4 (for a size/2-bit rational lower bound followed by a size/2-bit rational upper bound); default size is 256  
    "string", // size is/must always be 64 (for a uint64 index into a string mapping)  
  ],  
  "?size": Number.min(1).max(2^32-1).step(1) // the size, in bits, of the type; if absent, use the default size of the type  
});

var AnnotationMap = schema(Map.of(/[0-9a-z_-]+/, Annotation)); // maps annotation identifiers to annotations

var Annotation = schema({  
  "?alias": String, // for referencing in external property files etc.; can be any utf-8 string, must be unique among the aliases within the same AnnotationMap element  
  "?description": String, // for pretty-printing/user information/etc.; can be any utf-8 string, not necessarily unique  
  "applies-to": List.of([ "states", "choices", "branches" ]), // must be non-empty  
  "type": Type, // must be of the default size (given explicitly as the value for size or implicitly by omitting size) except for types "rational" and "rational-interval", for which the size must be a positive multiple of 128 and 256, respectively  
  "?lower": Number.min(-2^63).max(2^63-1).step(1), // lower bound for the values of the annotation, for numeric types, omit if the bound does not fit into int64  
  "?upper": Number.min(-2^63).max(2^63-1).step(1), // upper bound for the values of the annotation, for numeric types, omit if the bound does not fit into int64  
  "?#strings": Number.min(1).max(2^32-1).step(1), // must be present iff the type of the annotation is "string" and indicates the number of strings used  
  "?probability-type": Type, // must be present iff the annotation stores distributions over values (as used in stochastic observations), with probabilities provided using this type, which must be continuous numeric and of the default size (given explicitly as the value for size or implicitly by omitting size) except for types "rational" and "rational-interval", for which the size must be a positive multiple of 128 and 256, respectively
  "?#probabilities": Number.min(1).max(2^63-1).step(1) // must be present if probability-type is present, and gives the sum of the supports of all distributions
};

var ValuationDescription = schema({  
  "unique": Boolean, // if true, then the valuations are guaranteed to be unique (e.g. no two states are mapped to the same valuation, with equality defined on the bits for the variables, ignoring padding bits)  
  "?#strings": Number.min(1).max(2^32-1).step(1), // present iff the type of any variables in this valuation are “string” and indicates the total number of strings used  
  "classes": Array.of(ValuationClassDescription)  
});

var ValuationClassDescription = schema({ // the size of a state in bits is the sum of the padding and type size values of the elements plus 1 for each element where is-optional is true; this size must be a multiple of 8 (which can be ensured by adding a padding entry as necessary)  
  "variables": Array.of([  
    {  
      "padding": Number.min(1).step(1), // number of bits of padding at this position  
    },  
    {  
      "name": String,  
      "?is-optional": Boolean, // marks the type as an option type, using one Boolean bit preceding the value (which does not count towards size); if absent, the type is not an option type  
      "type": Type, // must not be an interval type  
      "?lower": Number.min(-2^63).max(2^63-1).step(1), // lower bound for the actual values of the variable, for numeric types, omit if the bound does not fit into int64  
      "?upper": Number.min(-2^63).max(2^63-1).step(1), // upper bound for the actual values of the variable, for numeric types, omit if the bound does not fit into int64  
      "?offset": Number.min(-2^63).max(2^63-1).step(1) // offset to add to the encoded value of the variable to obtain its actual value, for uint and int types (default 0); note that a negative offset with uint turns the type of the resulting values into int  
    }  
  ])  
});
```

---

## Binary File Types

Signed integers are stored in two's-complement representation. All multi-byte values are stored in little-endian byte order. We write `file[i]` to refer to the `(i+1)`-th byte in the file, and `file[i]_t` (where `t` is some positive number `b` or some type of size `b` bytes) to refer to the value represented by the sequence of bytes `file[i]`, ..., `file[i+b-1]` in little-endian byte order.

The UMB format uses 3 types of binary files:

### 1\. CSR Mappings

**Concept:**  A CSR ("compressed sparse row") file maps each index `i=0,...,n-1` to an interval `[l_i,u_i]` of target indices. The target index intervals must be sequential, with the first starting at zero, i.e. `l_0=0` and `u_(i-1)+1 = l_i` for all `i=1,...,n-1`.

**Representation:** A file consisting of `n+1` sequential `uint64` values, with `file[i]_8 = l_i` for `i<n` and `file[n]_8 = u_(n-1) + 1`.

**Default:** If a specified CSR file is absent: Every applicable index `i` is mapped to `[i,i]`, unless noted otherwise.

**Example:** Suppose we map state indices to choice indices. If state 0 has 2 choices, state 1 has 3 choices, and state 2 has 4 choices, we store `0 2 5 9`, corresponding to intervals `[0,1]`, `[2,3,4]`, `[5,6,7,8]` of indices to the total list of 9 choices.

**Notes:** A CSR file stores the beginning of each interval (and the beginning of the `(n+1)`-th interval", which indicates the end of the list). We store `n+1` elements instead of omitting the leading 0 to simplify access. In particular, the boundaries of the interval for index `i` can always be found at `file[i]_8` and `file[i+1]_8` without special treatment of the file beginning or end. The target indices are always stored as `uint64`. They denote an index in the target file w.r.t. the element type of the target file. For example, if the target file stores elements of size 128 bits, an index of 3 refers to an offset of `3 * 16` bytes.

### 2\. TO1 Mappings

**Concept:**  A TO1 file maps each index `i=0,...,n-1` to a target value `v_i` of a specified fixed-size element type `t`.

**Representation:**  A file consisting of `n` sequential values of type `t`, with  
`file[i]t \= v_i`.

**Special case:** `t = bool`: The file stores a single `n`-bit binary number (in little-endian byte order), with the `(i+1)`-th least significant bit being 1 if `v_i` else 0. The file consists of `ceil(n / 64⌉) * 8` bytes, with the `(n+1)`-th least significant bit and all more significant bits being 0 (padding).

**Default:** If a specified TO1 file is absent:  Every applicable index `i` is mapped to the equivalent of 0 in `t`, or to the least value of `t` if no equivalent of 0 exists, unless noted otherwise. If no equivalent of 0 or least value of `t` exists, then the value to assume if the file is absent must be explicitly specified.

**Notes:**  The case of `t` \= bool means we store a bitset, where the truth value for index `i` is stored at bit `i`, with bit 0 being the least significant bit. For example, consider the following representations of state sets: 
* 16 states, state 0 is in the set:
```
Binary number: 00000000.00000001
In memory:     00000001.00000000
```
* 14 states, all in the set:
```
Binary number: __111111.11111111
In memory:     11111111.00111111
```
This representation means that, no matter which numeric type `uintx` is used, the truth value for index `i` is always, in C-like notation, `(file[floor(i/x)]_x >> (i mod x)) & 1`.

### 3\. SEQ Sequences

**Concept:** A SEQ file contains a sequence of `n` values `v_0,...,v_(n-1)` of type `t` at indices `i_0,...,i_(n-1)`. Different values of type `t` can have a different size (e.g. strings). We write `sizeof(v)` for the size, in bytes, of value `v`. Let `b` be the greatest common divisor of the sizes of all values that `t` can take. Then the indices are expressed in multiples of `b`, and `ij = sizeof(v_0)+...+sizeof(v_(j-1))`.

**Representation:**  A file consisting of `i_n-1 + sizeof(v_n-1)` bytes, with  
`file[ij * b], ..., file[ij \+ sizeof(v_j) * b] = v_j`.

**Default:** If a specified SEQ file is absent: The meaning of the absence of a certain SEQ file must be explicitly specified.

**Notes:** SEQ files are indexed into by CSR and TO1 files, using the multiples-of-`b` indices specified above. SEQ files are in particular used to store values for variable-width types such as strings (e.g. action label names) and arbitrary-precision rationals (e.g. rational probabilities). A rational of length `c` bits, where `c` is a multiple of 128, consists of two `c/2`-bit integers: first the numerator, then the denominator. The numerator is signed while the denominator is unsigned and must not be 0\. Intervals of rationals work the same way just with 4 equal-size rational values (lower numerator, lower denominator, upper numerator, upper denominator). Rationals are indexed in terms of multiples of `b = 128`.

---

## Binary Files

The UMB format prescribes the binary files specified in this section.

**Filenames:** Files are named according to the following conventions:

* CSR mapping files are named `source-to-targets.bin` (with plural `targets`)..
  _Exception: CSR mappings into byte ranges for strings use "string-mapping" for targets._
  Examples: `state-to-choices.bin`, `string-mapping.bin`
  
* TO1 mapping files are named source-to-target.bin (with singular target).  
    Exception: TO1 mappings to `bool`, which are named `source-is-property.bin`.
    Examples: `state-to-exit-rate.bin`, `state-is-initial.bin`
    
* SEQ files are named after what they store, in plural form.  
  Example: `strings.bin`

If a file is in a source-specific subfolder, the mapping source is omitted in the file name, and the main TO1 mapping file is named `values.bin`. Overall, this scheme results in systematic and consistent file naming, and groups files related to the same entities when sorting by file names.

### 1\. States

`/state-to-choices.bin`  
CSR: state \-\> interval(choice)  
_Can be omitted if every state has exactly one choice (due to CSR default), e.g., in DTMC and CTMC (where `#players = 0`). If present, file size must be `(#states + 1) * 8` bytes._

`/state-to-player.bin`  
TO1: state \-\> player (type `uint32`)  
_Does not apply and must be omitted if `#players = 0`\. Can be omitted if every state is controlled by player 0 (e.g. when `#players = 1`\) (due to TO1 default). If present, file size must be `#states  * 4` bytes._

`/state-is-initial.bin`  
TO1: state \-\> is initial? (type `bool`)  
_If omitted, no state is initial. If present, file size must be `ceil((#states / 64)) * 8` bytes._

`/state-is-markovian.bin`  
TO1: state \-\> is Markovian? (type `bool`)  
_Does not apply and must be omitted unless `"time"` is `"urgent-stochastic"`. If omitted and `"time"` is `"urgent-stochastic"`, every state is mapped to false. If not Markovian, a state is probabilistic. Each Markovian state must have at most 1 choice. If present, file size must be ⌈(`#states` / 64)⌉ \* 8 bytes._

`/state-to-exit-rate.bin`  
TO1: state \-\> exit rate (type defined in `index.json`)  
_Does not apply and must be omitted if `"time"` is `"discrete"`. If `"time"` is `"urgent-stochastic"`, then the exit rates for non-Markovian states can be anything and must be ignored._

### 2\. Choices (nondeterminism)

`/choice-to-branches.bin`  
CSR: choice \-\> interval(branch)  
Note: Typically absent in LTS or deterministic games.

### 3\. Branches (probabilities)

`/branch-to-target.bin`  
TO1 mapping: branch \-\> state (`uint64`)  
_Note: This allows the same state-action-target triple to occur multiple times, which is intended._

`/branch-to-probability.bin`  
TO1 mapping: branch \-\> probability (type defined in `index.json`)  
_Must be absent if `"branch-values"` is `"none"` (e.g. an LTS)._

### 4\. Action labels

Actions can be assigned to choices and branches. The choice actions are `0, ...,n-1` where `n` is the value of property `#choice-actions` in `index.json`. Each choice action can optionally be given a string label. This applies for branch actions analogously. Storage, described below, is as for a standard string annotation to choices/branches, except that the strings themselves can optionally be omitted.

Choice actions are in the following files in folder `/actions/choices`:

`values.bin`  
TO1 mapping: choice \-\> choice action (`uint32`)  
_Must be omitted if `#choice-actions = 0`; otherwise, if omitted, each choice is mapped to choice action 0\._

`string-mapping.bin`  
CSR mapping: choice action \-\> interval(bytes)  
_Maps into `strings.bin`. The interval must make up a valid UTF-8 string. Must be omitted if `#choice-actions = 0`; otherwise, if omitted, no string labels are provided for choice actions._

`strings.bin`  
SEQ: sequence of UTF-8 strings whose bytes `string-mapping.bin` maps into  
_Must be present if and only if `string-mapping.bin` is present._

Branch actions are in the following files in folder `/actions/branches`, following the same pattern as choice actions:

`values.bin`  
TO1 mapping: branch \-\> branch action (uint32)  
_Must be omitted if `#branch-actions = 0`; otherwise, if omitted, each branch is mapped to branch action 0\._

`string-mapping.bin`  
CSR mapping: branch action \-\> interval(bytes)  
_Maps into `strings.bin`. The interval must make up a valid UTF-8 string. Must be omitted if `#branch-actions = 0`; otherwise, if omitted, no strings are provided for branch actions._

`strings.bin`  
SEQ: sequence of UTF-8 strings whose bytes `string-mapping.bin` maps into  
_Must be present if and only if `string-mapping.bin` is present._

### 5\. Observations

**Observations** (e.g. for POMDPs) can be attached either to states (when they depend only on the target state of a transition) or to branches (when they depend on both the action taken and the target state, and optionally also the source state). They are stored in the folder `/observations` using the same notation as annotations (see below), assuming a value type of observation (uint64). For example, deterministic state-based observations would be stored as:

`/observations/states/values.bin`  
TO1 mapping: state \-\> observation (`uint64`)

### 6\. Annotations

**Annotations** provide mappings from model entities (states, choices, branches, observations, players) to values of a specified type. Annotations are arranged in **groups** and each annotation has an **ID**, which is unique within its group. There are two pre-defined groups: `rewards` and `aps`, which are used to store a model’s rewards (continuous numeric type) and atomic propositions (Boolean type), respectively.

An annotation is stored in folder `/annotations/<group>/<id>` and contains one or more subfolders `states`, `choices`, `branches`, `observations` or `players`, indicating what is annotated. Annotation folders can contain the following files:

`values.bin`  
TO1: entity \-\> annotation value (type defined in `index.json`)

`string-mapping.bin`  
CSR mapping: string index \-\> interval of bytes in `strings.bin`.  
_The interval must make up a valid UTF-8 string. Must be present if and only if the annotation's type is `string`._

`strings.bin`  
SEQ: sequence of UTF-8 strings  
_Must be present if and only if `string-mapping.bin` is present._

`distribution-mapping.bin`  
CSR mapping: entity index (`uint32` or `uint64`, depending on the object) \-\> annotation distribution range (`uint64`)  
Only relevant for stochastic annotations. Maps each entity to an interval of indices. These indices map into both `values.bin` and `probabilities.bin`, which should be of equal size (equal to the sum of the size of the supports for all distributions). If omitted, 1:1 mapping, so each entity corresponds to a Dirac distribution, i.e., a single value.

`probabilities.bin`  
TO1 mapping: annotation distribution index (`uint64`) \-\> probability  
If omitted, everything maps to 1\. Should be omitted if `distributions.bin` is not present.

### 7\. Valuations

**Valuations** provide optional, more detailed representations for model entities (states, choices, branches, observations or players). A valuation consists of the values for a set of **variables**, with the values of each variable being from the domain of some specified type. A set of such variables and their types is called a **valuation class**. A valuation can use multiple valuation classes, which are defined in `ValuationClassDescription` elements in `index.json`; for each kind of entity (states, choices, ...), a list of valuation classes is given, and each entity's valuation can conform to a different valuation class. Each valuation of the same class is of the same size, which can be derived from the corresponding `ValuationClassDescription` element in `index.json`.

Let `<entity>` be the entity kind: `states`, `choices`, `branches`, `observations` or `players`. The valuations for one entity kind are stored in folder `/valuations/<entity>`, containing the following files:

`valuations.bin`
SEQ: sequence of valuations  
_There must be one valuation for each entity._

`valuation-to-class.bin`  
TO1: entity \-\> index of valuation class (`uint32`)  
_Maps every entity to the index of the valuation class of that one entity's valuation. Note that different classes have different sizes, so if this file is present, tools may need to compute the offsets of the entries in valuations.bin in a linear pass over this file. If absent, then every entity of this kind is mapped to the first valuation description of the corresponding element of `valuations` in `index.json` and `valuations.bin` is in fact a TO1 mapping._

`string-mapping.bin`  
CSR mapping: string index \-\> interval of bytes in strings.bin  
_Must make up a valid UTF-8 string. Must be omitted if there are no string-valued variables in state valuations._

`strings.bin`  
SEQ: sequence of UTF-8 strings  
_Must be present iff string-mapping.bin is present._

#### Encoding of valuations and variable values

File `valuations.bin` contains one state valuation per entity, each of which internally consists of a sequence of variable values and padding bits, bit-packed into a large binary number in the order of the entries in the "variables" element (i.e. the first variable value/padding occupies the least significant bits) and written to file in little-endian byte order. The internal structure is described by `valuations` in `index.json`.

**Example 1:** Four variables `v0`, `v1`, `v2` and `v3`, bit-packed, with `v1` being optional, with types as given in the valuation class definition below, and the string  list  `["xyz","ab","c"` (used by variable `v2`).
```javascript
   {  
     "variables":  
     [   
       { "name": "v0", "type": "bool", "size": 2 },  
       { "name": "v1", "type": "uint", "size": 6, "is-optional": true },  
       { "padding": 5 },  
       { "name": "v2", "type": "string", "size": 64 },  
       { "name": "v3", "type": "bool", "size": 3 },  
       { "padding": 7 }  
     ]  
   }
```
Total encoding size: 88 bits \= 11 bytes
Example values: `v0 = true`, `v1 = Some(27)`, `v2 = "ab"`, `v3 = true`
Binary-encoded values: `v0 = 01`, `v1 = 1011011`, `v2 = 00...001`, `v3 = 001`
Binary number, including padding, marked as `_`:
```
_______.0 01.000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 01._____.1 011011.01
```
File `valuations.bin:` in little-endian byte order:
```
01101101 01000001 00000000 00000000 00000000 00000000 00000000 00000000 00000000 01000000 00000000
```
File `string-mapping.bin`: `[0 3 5 6]`
File `strings.bin`: `xyzabc`

**Example 2:** C-like byte-packed `struct { uint8 sv0, int16 sv1 }`  with offset for `sv0` that allows values \-128 to 127 to be represented by `sv0`
```javascript
{  
     "variables":  
     [   
       { "name": "sv0", "type": "uint", "size": 8, "offset": \-128 },  
       { "name": "sv1", "type": "int", "size": 16 }  
     ]  
   }
```
Total encoding size: 24 bits \= 3 bytes
Example values: `sv0 = 56` (encoded as 56 \- (-128) \= 184), `sv1 = -16897`
Binary-encoded values: `sv0 = 10111000`, `sv1 = 1011110111111111`
Binary number:
```
10111101 11111111.10111000
```
File `valuations.bin:` in little-endian byte order:
```
10111000 11111111 10111101 
```
Note that the in-file representation matches the typical little-endian in-memory representation of the struct in e.g. C here.  
