```mermaid
graph LR
A["Server"]
B[Service]
C[Engine]
D[Host]
E[Context]
F[Wrapper]

A --> |"1对1(包含关系)"| B
B --> |"1对1(包含关系)"| C
C --> |"child关系"| D
D --> |"child关系"| E
E --> |"child关系"| F

```

