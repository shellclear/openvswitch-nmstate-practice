# Este repositorio guarda exemplos de configuracoes para:
* OVS(OpenVSwitch)
* NMState

A ideia e conseguir criar bridges no host onde estamos executando um CRC(CodeReady Containers) e configurar as NICs adicionadas ao CRC com diferentes opcoes de configuracao como por exemplo: bond, bridge, trunks e vlans. Em seguida conseguir utilizar o Multus para adicionar redes adicionais aos pods que executam as VMIs do OCP-V(kubevirt) para que as VMs tenham acesso a redes externas.
