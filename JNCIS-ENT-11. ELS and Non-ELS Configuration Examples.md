### ELS - Enhanced Link-Layer Switch vs Non-ELS
- Differenze nella gerarchia a cui vengono memorizzati alcuni parametri di carattere generale per lo switch

  - NonELS `{master:0} [edit ethernet-switching-options]`
  - ELS `{master:0} [edit switch-options]`

- Logica L3-Interfaces
  
  - NonELS usano RVI Routed VLAN Interfaces e sono identificati come "vlan" all'interno delle interface configuration
    ` show interfaces vlan terse`
  - ELS switch isano invece IRB Integrated Routing and Bridging interface
    `show interfaces irb terse`
