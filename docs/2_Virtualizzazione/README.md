# Setup oVirt Lab

Spesso si necessita di avere un piccolo lab di virtualizzazione, un modo facile per ottenerlo è creare due (o più) nodi virtualizzati in modalità self hosted.

## Preparazione

Prima di iniziare.

Iniziamo a preparare quello che ci servirà:

- 2 virtual machine con le seguenti caratteristiche (ciascuna):
    - 2 vCPU (raccomandate 4 vCPU)
    - 6 GiB di memoria (4 per il manager, 2 per l’host)
    - 1 vNIC di tipo VirtIO
    - 1 vDisk da 64 GB
- 2 vDisk di tipo shared per simulare la SAN FIbreChannel:
    - 1 vDisk di tipo shared da 75 GB (servirà per il manager)
    - 1 vDisk di tipo shared da 10 GB (come storage per le VM)
- L’immagine ISO di RHVH (solo se in possesso di una sottoscrizione) o di oVirt Node:
    - RHVH → scaricare l’ultima [Hypervisor Image for RHV 4.4.z](https://access.redhat.com/downloads/content/328/ver=4.4/rhel---8/4.4/x86_64/product-software)
    - oVirt Node → scaricare l'ultima [ovirt-node-ng-installer-4.5.???.el8.iso](https://resources.ovirt.org/pub/ovirt-4.5/iso/ovirt-node-ng-installer/)

> N.B.: al momento della redazione di questo documento, solo le immagini di oVirt Node basate su EL8 sono supportate.

Dato che l'obiettivo è avere questa infrastruttura virtualizzata, è fondamentale abilitare la virtualizzazione nested nel nostro host.

## Installazione di libvirt/KVM
Per installare la virtualizzazione su RHEL 8 / CentOS 8, attenersi alla documentazione ufficiale → [Configuring and managing virtualization](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_virtualization/getting-started-with-virtualization-in-rhel-8_configuring-and-managing-virtualization#enabling-virtualization-in-rhel8_virt-getting-started) (paragrafo Enabling virtualization).

Per installare la virtualizzazione su Fedora, attenersi alla documentazione della community → https://docs.fedoraproject.org/en-US/quick-docs/getting-started-with-virtualization/

### Preparazione della rete di management.

Un esempio di rete di management può essere quello definito dal seguente xml:
```xml
<network>
  <name>ovirt-mgt</name>
  <forward mode="nat" />
  <domain name="ovirt" />
  <dns>
    <host ip="192.168.200.2">
      <hostname>engine.ovirt</hostname>
    </host>
    <host ip="192.168.200.3">
      <hostname>node1.ovirt</hostname>
    </host>
    <host ip="192.168.200.4">
      <hostname>node2.ovirt</hostname>
    </host>
  </dns>
  <ip address="192.168.200.1" netmask="255.255.255.0">
  </ip>
</network>
```


> N.B.: È fondamentale che nella definizione della rete siano presenti le risoluzioni sia dell’oVirt Engine, sia dei nodi hypervisor. L’installazione di oVirt non andrà a buon fine se saranno presenti delle anomalie DNS come, ad esempio, un nome host che risolve più di un indirizzo IP.

Per importare questa definizione di connessione in libvirt, possiamo eseguire i seguenti comandi:

```bash
virsh net-define --file /path/to/xml
virsh net-autostart ovirt-mgt
virsh net-start ovirt-mgt
```

### Preparazione dei dischi shared (per simulare uno storage FC).

Per lo storage delle VM, andremo a simulare una Storage Area Network di tipo FIbreChannel. Per farlo, useremo i dischi shared di libvirt.

Per prima cosa, creiamo un disco necessariamente di tipo raw (essenziale per la condivisione):

```bash
virsh vol-create-as default HE_store.img 75G --format raw
```

Sostituite *`default`* con il nome del pool dove desiderate creare il volume.
	
Quindi creiamo il secondo disco:

```bash
virsh vol-create-as default VM_store.img 10G --format raw
```

### Abilitazione nested virtualization

Per abilitare la nested virtualization ci basta eseguire i seguenti comandi:

```bash
## Intel processor
modprobe -r kvm_intel
echo "options kvm_intel nested=1" > /etc/modprobe.d/kvm.conf
modprobe kvm_intel

## AMD processor
modprobe -r kvm_amd
echo "options kvm_amd nested=1" > /etc/modprobe.d/kvm.conf
modprobe kvm_amd
```

Se tutto è filato liscio,  ora la nested virtualization dovrebbe risultata abilitata:

```bash
cat /sys/module/kvm_intel/parameters/nested
Y # --> abilitata
```

### Creazione della VM

Per definire le vm possiamo creare la macchina virtuale eseguendo il comando:

```bash
virt-install --name node1.ovirt \
--cdrom /path/to/ovirt-node-ng-installer-4.5.3.2-2022102813.el8.iso \
--vcpus 4 --memory 6144 \
--network network=ovirt-mgt \
--os-variant rhel8.6 \
--cpu host-passthrough \
--controller type=scsi,model=virtio-scsi \
--disk size=64,pool=default \
--disk vol=default/HE_store.img,bus=scsi,format=raw,shareable=on \
--disk vol=default/VM_store.img,bus=scsi,format=raw,shareable=on
```

Sostituite *default* (ogni volta che compare nel comando) con il nome del pool dove avete precedentemente creato i due dischi shared.

Per vedere quali sono tutte e os-variant supporate sul proprio sistema operativo, eseguire il comando:

```bash
osinfo-query os
```

Il comando andrà a creare il disco della macchina virtuale nel dominio di default (/var/lib/libvirt/images) e aprire una finestra in cui è sarà possibile eseguire l’installazione della macchina virtuale.

> N.B.: è necessario eseguire questo passaggio una seconda volta per creare anche la macchina node2.ovirt.


## Installazione

Questa parte verrà spiegata durante il corso.

### Installazione dell’ RHVH ( o dell’oVirt Node)

A seconda che si sia scelto di installare RHV od oVirt, attenersi alla relativa documentazione:

- RHVH → [Installing the Self-hosted Engine Deployment Host](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.4/html/installing_red_hat_virtualization_as_a_self-hosted_engine_using_the_command_line/installing_the_self-hosted_engine_deployment_host_she_cli_deploy)
- oVirt Node → [Installing the Self-hosted Engine Deployment Host](https://www.ovirt.org/documentation/installing_ovirt_as_a_self-hosted_engine_using_the_command_line/index.html#Installing_the_self-hosted_engine_deployment_host_SHE_cli_deploy)

> N.B.: è necessario prestare molta attenzione alla scelta del disco di installazione. Va infatti selezionato solo il disco VirtIO da 64 GB, lasciando deselezionati i due dischi SCSI, che verranno invece utilizzati in seguito.

### Download dell’ RHV-M Appliance (o dell’Engine Appliance)

Questo step è del tutto opzionale. Può tornare utile nel caso si stia effettuando un'installazione *disconnessa*.

In caso si stia effettuando un’installazione con connettività Internet, procedere pure alla fase successiva.

A seconda che si sia scelto di installare RHV od oVirt, scaricare l’appliance corrispondente e installarla, attenendosi alla relativa documentazione:

- RHVM → [Manually installing the RHV-M Appliance](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.4/html/installing_red_hat_virtualization_as_a_self-hosted_engine_using_the_command_line/installing_the_red_hat_virtualization_manager_she_cli_deploy#proc_Manually_installing_the_appliance_install_RHVM)
- oVirt Engine → [Manually installing the Engine Appliance](https://www.ovirt.org/documentation/installing_ovirt_as_a_self-hosted_engine_using_the_command_line/index.html#proc_Manually_installing_the_appliance_install_RHVM)

Quindi procedere alla fase successiva.

### Installazione dell’ RHVM (o dell’oVirt Engine)

A seconda che si sia scelto di installare RHV od oVirt, attenersi alla relativa documentazione:

- RHVM → [Deploying the self-hosted engine using the command line](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.4/html/installing_red_hat_virtualization_as_a_self-hosted_engine_using_the_command_line/installing_the_red_hat_virtualization_manager_she_cli_deploy#Deploying_the_Self-Hosted_Engine_Using_the_CLI_install_RHVM)
- oVirt Engine → [Deploying the self-hosted engine using the command line](https://www.ovirt.org/documentation/installing_ovirt_as_a_self-hosted_engine_using_the_command_line/index.html#Deploying_the_Self-Hosted_Engine_Using_the_CLI_install_RHVM)

Rispetto a quanto riportato nella documentazione, è importante apportare le seguenti configurazioni durante il setup dell’Hosted Engine:

1. alla domanda
   ``` Please specify the memory size of the VM in MB. The default is the maximum available [6824]: ```
   rispondere con il valore **4096**.

2. alla domanda
   ``` Please specify the storage you would like to use (glusterfs, iscsi, fc, nfs)[nfs]: ```
   rispondere con **fc**.

3. alla schermata

   ```
   The following luns have been found on the requested target:
   [1] drive_scsi0-0-0-0   75GiB   QEMU QEMU HARDDISK
   		status: free, paths: 1 active
   
   [2] drive_scsi0-0-0-1   30GiB   QEMU QEMU HARDDISK
   		status: free, paths: 1 active
   
   Please select the destination LUN (1, 2) [1]:
   ```

   selezionare uno dei due dischi SCSI creati in precedenza (quello da 75 GiB è l’unico che rispetta la dimensione minima).

Seguire l’intero capitolo fino a tutto il paragrafo **Connecting to the Administration Portal** compreso.

## Task post-installazione

### Installazione del secondo host

A questo punto l’installazione è finita e si può procedere all’aggiunta del secondo nodo **(node2.ovirt)** che si dovreste aver già installato precedentemente.

> N.B.: sul secondo nodo non è ovviamente necessario installare l’appliance dell’engine.

Questo nodo andrà aggiunto come **Self-Hosted Engine Node**, ovvero in grado di ospitare, oltre che le normali VM, anche la VM dell’Engine.

Per aggiungere il secondo nodo al cluster, attenersi alla relativa documentazione:

- RHVM → [Adding Self-Hosted Engine Nodes to the Red Hat Virtualization Manager](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.4/html/installing_red_hat_virtualization_as_a_self-hosted_engine_using_the_command_line/installing_hosts_for_rhv_she_cli_deploy#Adding_self-hosted_engine_nodes_to_the_Manager_SHE_cli_deploy)
- oVirt Engine → [Adding Self-Hosted Engine Nodes to the oVirt Engine](https://www.ovirt.org/documentation/installing_ovirt_as_a_self-hosted_engine_using_the_command_line/index.html#Adding_self-hosted_engine_nodes_to_the_Manager_SHE_cli_deploy)

### Preparazione dello storage

Oltre alla Storage Domain **hosted_storage** creato automaticamente all'installazione, ne andremo a creare un secondo per le virtual machine.

Per aggiungere il secondo Storage Domain al datacenter, attenersi alla relativa documentazione per il Fibre Channel:

- RHVM → [Adding FCP Storage](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.4/html/installing_red_hat_virtualization_as_a_self-hosted_engine_using_the_command_line/adding_storage_domains_to_rhv_she_cli_deploy#Adding_FCP_Storage_SHE_cli_deploy)
- oVirt Engine → [Adding FCP Storage](https://www.ovirt.org/documentation/installing_ovirt_as_a_self-hosted_engine_using_the_command_line/index.html#Adding_FCP_Storage_SHE_cli_deploy)

Rispetto a quanto riportato nella documentazione, è importante apportare le seguenti configurazioni:

- Come **Storage Type** selezionare **Fibre Channel**.
- Come **LUN ID** selezioniamo un disco QEMU libero (dovrebbe essere quello da 10 GiB, se avete correttamente utilizzato quello da 75 per l’Hosted Engine).
