%% OSPFv2 Area Types Lab - topology with interface names
graph TD
  subgraph A0["Area 0 - Backbone"]
    R1
    R2
    R3
    R4
    R5
  end
  subgraph A1["Area 1 - Stub"]
    R6
  end
  subgraph A2["Area 2 - Totally Stubby"]
    R7
  end
  subgraph A3["Area 3 - NSSA"]
    R8
    R10
  end
  subgraph A4["Area 4 - Totally NSSA"]
    R9
  end
  subgraph EG["EIGRP 100"]
    R11
  end
  R1 ---|"Gi0/0 - Gi0/0"| R2
  R1 ---|"Gi0/1 - Gi0/0"| R3
  R1 ---|"Gi0/2 - Gi0/0"| R4
  R1 ---|"Gi0/3 - Gi0/0"| R5
  R2 ---|"Gi0/1 - Gi0/0"| R6
  R3 ---|"Gi0/1 - Gi0/0"| R7
  R4 ---|"Gi0/1 - Gi0/0"| R8
  R8 ---|"Gi0/1 - Gi0/0"| R10
  R5 ---|"Gi0/1 - Gi0/0"| R9
  R8 ---|"Gi0/2 - Gi0/0"| R11
  R9 ---|"Gi0/1 - Gi0/0"| Net((Net / Internet))
