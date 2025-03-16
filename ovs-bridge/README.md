# Configurando uma bridge Open vSwitch (OVS) no Fedora 41 para conectar máquinas virtuais(VMs) usando Vlans

## 1 Instalando os pacotes necessários

### 1.0 Primeiro, instale o OpenvSwitch e o NetworkManager

```bash
sudo dnf install -y openvswitch NetworkManager-ovs
```

### 1.1 Inicie e habilite o serviço do OpenvSwitch e NetworkManager:

```bash
sudo systemctl enable --now openvswitch NetworkManager
```

### 1.2 Reinicie o NM

```bash
sudo systemctl restart NetworkManager
```

## 2 Criar uma bridge OVS

### 2.0 Agora crie a bridge OVS manualmente usando o comando ovs-vsctl:

```bash
sudo ovs-vsctl add-br ovs-br0
```

<!-- ### 2.1 Configure a porta da bridge como trunk, isso nos permitira criar interfaces no host em diferentes vlans usando essa porta
```bash
sudo ovs-vsctl set port ovs-br0 trunks=1-4094
``` -->

### 2.2 Verifique se a bridge foi criada corretamente:

```bash
sudo ovs-vsctl show
```

# 3 Configurar o host

### 3.1 Criando a interface vlan25 no host para fazer testes de comunicacao entre o host e a VM usando as portas trunk e access na bridge

```bash
nmcli connection add type vlan con-name vlan25 ifname vlan25 dev ovs-br0 id 25 ipv4.addresses 192.168.25.1/24 ipv4.method manual
```

### 3.2 Ativar a interface vlan25 no host

```bash
nmcli connection up vlan25
```

## 3.3 Criar a rede no libvirt.

* Isso pode ser feito pela interface do virt-manager se voce estiver usando esse software.

### 3.4 Crie o arquivo XML

```bash
cat << eof > $PWD/libvirt-ovs-bridge-config.xml
<network>
  <name>ovs-br0</name>
  <forward mode='bridge'/>
  <bridge name='ovs-br0'/>
  <virtualport type='openvswitch'/>
</network>
eof
```

### 3.5 Defina a rede no libvirt com o comando virsh

```bash
sudo virsh net-define libvirt-ovs-bridge-config.xml
sudo virsh net-autostart ovs-br0
sudo virsh net-start ovs-br0
```

### 3.6 Verifique se a rede foi corretamente criada.

```bash
sudo virsh net-list --all
```

## Neste ponto, Crie uma VM normalmente e adicione nela uma interface na rede previamente criada.

## 4 Configurar VM para ter acesso a vlan25(NO TRUNK)

### 4.0 Configurar uma conexao de rede na VM

```bash
nmcli con add type ethernet con-name eth1-access-vlan25 ifname eth1 ipv4.method manual ipv4.addresses 192.168.25.2/24
nmcli con up eth1-access-vlan25
```

### 4.1 Identifique a vnet usada pela VM e configure a porta vnet na ovs-br0 como portaccess vlan-id 25 para que haja comunicacao entre o host e a VM

```bash
sudo ovs-vsctl show
sudo ovs-vsctl set port vnet35 tag=25
sudo ovs-vsctl list port vnet35
```

* Perceba que a porta esta configurada para parmitir tag 25

### 4.2 Abre um terminal no host e execute um ping contra a VM e se tudo estiver funcionando corretamente havera resposta.
```bash
ping 192.168.25.2
```

### 4.3 Abra um novo terminal e execute o comando tcpdump para verificar se os pacotes estao fluindo normalmente, perceba o vlanid 25 no output do comando

```bash
sudo tcpdump -i ovs-br0 -e vlan
```

* Exemplo output:

```bash
20:27:24.714147 72:6e:cd:8b:cc:40 (oui Unknown) > 52:54:00:12:20:22 (oui Unknown), ethertype 802.1Q (0x8100), length 102: **vlan 25**, p 0, ethertype IPv4 (0x0800), ppacific-fedora > 192.168.25.2: ICMP echo request, id 65, seq 5, length 64
20:27:24.714366 52:54:00:12:20:22 (oui Unknown) > 72:6e:cd:8b:cc:40 (oui Unknown), ethertype 802.1Q (0x8100), length 102: **vlan 25**, p 0, ethertype IPv4 (0x0800), 192.168.25.2 > ppacific-fedora: ICMP echo reply, id 65, seq 5, length 64
```

* Para que seja possivel definir a porta da VM na bridge como trunk(o que faremos no proximo passo) devemos remover a configurao da porta vnet que taggea pacotes vindos dessa porta como vlan25 e configurar-la como trunk, para isso use o comando abaixo:

```bash
sudo ovs-vsctl clear port vnet35 tag
```

## 5 Configurar VM para receber o trafego via trunk

### 5.0 Por padrao as portas do ovs sao configuradas como trunk e permitem a passagem de todos os tags mas podemos configurar uma porta como native-untagged ou especificar vlans no trunk para nossos testes.

```bash
sudo ovs-vsctl set port vnet35 trunks=1 vlan_mode=native-untagged other-config:vlan=0
```

* Apos a cofiguracao com o comando acima o trunk permite pacotes tagged com a vlan1 e pacotes untagged serao colocados na vlan nativa, 0. Como os pacotes vindos da VM nao sao tagged eles vao entrar pela vlan nativa impedindo a comunicacao entre a VM e o host que esta configurado na vlan25

### 5.1 Vamos configurar uma interface vlan na VM para receber o trafego da vlan25 usando o device eth1(que sera a porta trunk na VM)

* o nome da interface da sua VM pode ser diferente, no meu caso era eth1 pois eu tinha duas interfaces.

```bash
nmcli con add type vlan con-name vlan25 ifname vlan25 dev eth1 id 25 ipv4.method manual ipv4.addresses 192.168.25.2/24
nmcli con up vlan25
```

* Neste ponto o comando ping lancado na VM em direcao ao host(192.168.25.1) nao funcionara pois o trunk que configuramos na porta vnet na bridge esta limitando a passagem de pacotes tagged somente para a vlan 1

### Configure a porta vnet da VM na bridge para receber no trunk pacotes tagged da vlanid 25, no meu caso essa porta era vnet35, verifique qual a porta da sua VM

```bash
sudo ovs-vsctl set port vnet35 trunks=1,25
```

* Agora a comunicacao flui normalmente pois a porta vnet do switch a qual a nossa VM esta conectada esta configurada como trunk e permite a passagem de pacotes tagged das vlan 1 e 25, veja a saida do comando:

```bash
sudo ovs-vsctl show 
873056d6-dffe-4171-a355-cd548f2d6587
    Bridge ovs-br0
        Port vnet4
            trunks: [1, 25]
            Interface vnet4
        Port ovs-br0
            Interface ovs-br0
                type: internal
    ovs_version: "3.4.0-2.fc41"
```

* Abaixo deixo o comando para remover a configuracao da porta da bridge. Isso vai ser necessario se voce quiser voltar a usar a porta de acceso na vlan25 novamente.

```bash
sudo ovs-vsctl clear port vnet35 trunks tag other-config vlan-mode
```

* Comando para remover a bridge

```bash
sudo ovs-vsctl del-br ovs-br0
```