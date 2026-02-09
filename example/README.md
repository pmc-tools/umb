# UMB Example

To give a feel for how UMB files encode models,
below is a textual view of the binary file [mdp.umb](mdp.umb),
which stores in UMB format a simple 4-state Markov decision process,
with atomic proposition `g`, reward `r` and state variables `x` and `b`.

Described in the [PRISM](https://www.prismmodelchecker.org/) language, the model is:

```
mdp

module M

	x : [0..2];
	b : bool;
	
	[a] x=0 -> 0.5:(x'=1) + 0.5:(x'=2)&(b'=false);
	[b] x=0 -> 0.2:(x'=2) + 0.8:(x'=2)&(b'=true);
	[a] x=1 -> 0.9:(x'=0) + 0.1:(x'=1);
	[a] x=2 -> (x'=2);

endmodule

label "g" = x=2 & b;

rewards "r" [b] true:1; endrewards
```

A textual view of the UMB file is as follow,
comprising a JSON metadata file and multiple binary data arrays
(see the [spec](../spec.md) for details).

```javascript
/index.json:
{
  "format-version": 1,
  "format-revision": 0,
  "model-data": {},
  "file-data": {
    "tool": "PRISM",
    "tool-version": "4.10",
    "creation-date": 1769360247
  },
  "transition-system": {
    "time": "discrete",
    "#players": 1,
    "#states": 4,
    "#initial-states": 1,
    "#choices": 5,
    "#choice-actions": 2,
    "#branches": 8,
    "#branch-actions": 0,
    "#observations": 0,
    "branch-probability-type": {
      "type": "double",
      "size": 64
    }
  },
  "annotations": {
    "aps": {
      "g": {
        "alias": "g",
        "applies-to": [
          "states"
        ],
        "type": {
          "type": "bool",
          "size": 1
        }
      }
    },
    "rewards": {
      "r": {
        "alias": "r",
        "applies-to": [
          "choices"
        ],
        "type": {
          "type": "double",
          "size": 64
        }
      }
    }
  },
  ...
\end{lstlisting}
\begin{lstlisting}[basicstyle=\ttfamily\scriptsize]
  ...
  "valuations": {
    "states": {
      "unique": true,
      "classes": [
        {
          "variables": [
            {
              "name": "x",
              "type": {
                "type": "uint",
                "size": 2
              }
            },
            {
              "name": "b",
              "type": {
                "type": "bool",
                "size": 1
              }
            },
            {
              "padding": 5
            }
          ]
        }
      ]
    }
  }
}
/state-to-choices.bin:
[0,2,3,4,5]
/choice-to-branches.bin:
[0,2,4,6,7,8]
/branch-to-probability.bin:
[0.5,0.5,0.2,0.8,0.9,0.1,1.0,1.0]
/branch-to-target.bin:
[1,2,2,3,0,1,2,3]
/state-is-initial.bin:
[1000]
/actions/choices/string-mapping.bin:
[0,1,2]
/actions/choices/strings.bin:
[a,b]
/actions/choices/values.bin:
[0,1,0,0,0]
/annotations/aps/g/states/values.bin:
[0001]
/annotations/rewards/r/choices/values.bin:
[0.0,1.0,0.0,0.0,0.0]
/valuations/states/valuations.bin:
[.....|0|00,.....|0|01,.....|0|10,.....|1|10]
```

