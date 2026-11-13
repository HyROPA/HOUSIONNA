# Plant models (L-system rules)

The four crop species in HOUSIONNA are generated with Houdini's L-system SOP, using rules written for this work, in five growth stages (0 to 4). The same rules live inside [`houdini_files/SIONNA_HOUDINI_BATCH_01-v3.hipnc`](../houdini_files/SIONNA_HOUDINI_BATCH_01-v3.hipnc). They are listed here so they can be read without Houdini.

Formalism: P. Prusinkiewicz and A. Lindenmayer, *The Algorithmic Beauty of Plants*, Springer, 1990. Rules use L-system SOP syntax. A trailing `:p` is the probability of a stochastic rule. The `#` column is the rule slot in the SOP.

| Plant | Premise | # | Rule |
|---|---|---|---|
| Strawberry | `A` | 1 | `A=/(137.5)[&(25)B(0)&(60)J(0.1)[+(45)J(0.1)][-(45)J(0.1)]]F(0.005)A:0.5` |
| | | 2 | `A=/(137.5)[&(10)E(0)&(75)J(0.1)[+(45)J(0.1)][-(45)J(0.1)]]F(0.005)A:0.3` |
| | | 3 | `A=/(137.5)[&(35)C(0)[&(35)D(0)M(0.2)][^(35)D(0)M(0.2)]C(0)M(0.25)]F(0.005)A:0.2` |
| | | 4 | `B(n):n<6=F(0.2)TB(n+1)` |
| | | 5 | `E(n):n<8=F(0.2)TE(n+1)` |
| | | 6 | `C(n):n<3=F(0.2)TC(n+1)` |
| | | 7 | `D(n):n<2=F(0.15)TD(n+1)` |
| | | 8 | `J(s):s<0.7=J(s+0.1)` |
| | | 9 | `M(s):s<0.4=M(s+0.1)` |
| | | 10 | `M(s):s>=0.4=TTTTTTTTK(0.2)` |
| | | 16 | `K(s):s<0.35=K(s+0.05)` |
| Sweet basil | `A` | 1 | `A=F(0.05)T[&(20)TJ(0.2)]/(180)[&(20)TJ(0.2)]/(90)A:0.6` |
| | | 2 | `A=F(0.05)T[&(20)TJ(0.2)][^(-30)B]/(180)[&(20)TJ(0.2)]/(90)A:0.25` |
| | | 3 | `A=F(0.05)T[&(20)TJ(0.2)][^(-30)B]/(180)[&(20)TJ(0.2)][^(-30)B]/(90)A:0.15` |
| | | 4 | `F(l):l<0.3=F(l+0.06)` |
| | | 5 | `J(s):s<1.1=J(s+0.2)` |
| | | 6 | `&(a):a<70=&(a+10)` |
| | | 7 | `B=C` |
| | | 8 | `C=A` |
| Kropsla (butterhead lettuce) | `A` | 1 | `A=/(137.5)[&(10)J(0.1)]F(0.01)A` |
| | | 2 | `J(s):s<1=J(s+0.12)` |
| | | 3 | `&(a):a<75=&(a+8)` |
| Cherry tomato | `A(0)` | 1 | `A(n):n<2=C(0)/(137.5)[&(50)F(0.1)TE(0)][&(30)B]A(n+1)` |
| | | 2 | `A(n):n>=2=C(0)/(137.5)[&(50)F(0.1)TE(0)][&(30)B][^(40)W(0)D(0)]A(0)` |
| | | 3 | `B=A(0):0.05` |
| | | 4 | `B=B:0.95` |
| | | 5 | `C(n):n<2=F(0.12)C(n+1)` |
| | | 6 | `E(n):n<3=F(0.15)T[+(60)J(0.1)][-(60)J(0.1)]E(n+1)` |
| | | 7 | `E(n):n>=3=J(0.1)` |
| | | 8 | `D(n):n<4=F(0.08)T[&(60)F(0.05)TM(0.1)]/(180)D(n+1)` |
| | | 9 | `J(s):s<0.6=J(s+0.1)` |
| | | 10 | `M(s):s<0.2=M(s+0.1)` |
| | | 11 | `M(s):s>=0.2=TTTTTTTTK(0.15)` |
| | | 12 | `K(s):s<0.3=K(s+0.05)` |
