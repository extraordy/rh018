# Introduzione a KVM

<p align="center">
    <img src="images/kvm.png" alt="KVM" width="600"/>
</p>

## Cosa significa "virtualizzazione"

Volendo sintetizzare al massimo, si potrebbe dire che la virtualizzazione sia *"una tecnologia che
permette di creare risorse virtuali e di mapparle logicamente a risorse fisiche, creando un layer
di astrazione aggiuntivo separato dall'hardware fisico"*.Detto in altre parole, la virtualizzazione
permette di eseguire diversi sistemi operativi in parallelo sulla stessa macchina fisica, spesso
(non sempre, dipende dalla tecnologia usata) senza che questi siano "consapevoli" di essere eseguiti
in una macchina virtuale. I risvolti che l'utilizzo di questa tecnologia ha (e continua ad avere) nel
mondo dell'Information Technology sono enormi.

## Vantaggi della virtualizzazione

Chi approccia la virtualizzazione per la prima volta, non di rado arriva a porsi la domanda "d'accordo,
possiamo creare una macchina virtuale che viene eseguita all'interno (o "sopra") una macchina fisica,
ma quali sono i vantaggi?"

### Ho eliminato l'hardware fisico?

No ovviamente, l'hardware non può essere eliminato. La macchina host
avrà sempre bisogno di avere CPU, RAM, storage, network etc.

### Ho semplificato l'amministrazione del sistema operativo o delle applicazioni?

No, neppure.
La macchina, anche se virtuale, ha comunque bisogno che il suo sistema operativo abbia le corrette configurazioni,
e lo stesso vale per le applicazioni. Ad esempio: se devo installare un server web, il fatto che sia virtualizzato
in sè non semplifica la configurazione di Apache. Anzi, potrebbe addirittura complicarla: l'interfaccia di rete della
macchina virtuale, ad esempio, è collegata ad una Virtual Network, un layer software ulteriore che si aggiunge alla
configurazione dell'interfaccia fisica della macchina host e che va amministrato.

Ma il bilancio, se si fermasse qui, sarebbe estremamente parziale, perché la virtualizzazione può portare
enormi vantaggi rispetto ad un approccio in cui tutto è fisico. Per citarne alcuni:

* **Consolidamento delle risorse**

   Il dimensionamento di un server che deve erogare un servizio o eseguire
un'applicazione può essere complicato. E' molto difficile valutare esattamente le risorse necessarie e,
quasi sempre, un server fisico si ritrova con risorse inutilizzate per gran parte del tempo. Grazie alla
virtualizzazione, possiamo trarre vantaggio di queste risorse libere: se il server ha potenza computazionale
residua, perché non aggiungere altre macchine virtuali che possano sfruttarle ed erogare servizi aggiuntivi?
Il risparmio economico e la razionalizzazione delle risorse possono essere enormi.

* **Backup**

  La virtualizzazione permette di creare backup e snapshot di tutta la nostra infrastruttura
con estrema facilità, rendendo anche la sperimentazione di nuove soluzioni molto più agile. Creo uno snapshot,
testo alcune modifiche o una nuova configurazione e, se ne ho bisogno, posso far tornare tutto allo stato
precedente in tempi brevissimi.

* **Deployment**

  Ci sono soluzioni come Red Hat Satellite e Ansible che rendono il deployment
di macchine anche fisiche un processo perlopiù automatico, ma la virtualizzazione ne moltiplca le capacità. Diventa
possibile definire non solo le configurazioni software, ma anche quelle "hardware" in maniera automatica. Sfruttando,
ad esempio, delle API, possiamo assegnare esattamente le risorse che ci servono su ogni macchina virtuale tramite una
logica implementata a livello software.

## Tipi di virtualizzazione

Nel panorama IT esistono diversi tipi di virtualizzazione, spesso categorizzati a seconda di *quello* che si vuole
virtualizzare (Desktop Virtualization, Server Virtualization, Application Virtualization etc.). Più spesso,
viene preso in considerazione il "metodo" usato per creare macchine virtuali. Alcuni esempi di tipi di virtualizzazione
sono:

* **Partitioning**

    La CPU è divisa in più parti e ognuna lavora individualmente in maniera isolata e può
eseguire un sistem opartivo diverso (come le IBM Logical Partitions - LPARs)

* **Full Virtualization**

  L'hardware della macchina virtuale è emulato in maniera trasparente, così che
il sistema operativo *guest* non sia consapevole di essere eseguito in una macchina virtuale (e quindi,
non abbia bisogno di essere modificato rispetto a quello che viene eseguito in una macchina fisica).
La *Full Virtualization* può essere ottenuta sia via software, traducendo al volo l'instruction set dei
binari eseguiti dalla macchina *guest* (è un processo costoso in termini di risorse), sia via hardware,
se la CPU della macchina host supporta la virtualizzazione (AMD-V o Intel VT). KVM sfrutta questa tecnologia.

* **Paravirtualization**

  La macchina *guest* esegue una versione modificata del sistema operativo e dei
driver che le permettono di venire eseguita senza bisogno delle estensioni di virtualizzazione della CPU.

Ci sono poi anche approcci ibridi e i container (che permettono la virtualizzazione delle singole
applicazioni e di tutte le loro dipendenze senza però bisogno di virtualizzare anche il kernel e l'intero sistema
operativo).

## Hypervisor

La virtualizzazione ha sempre bisogno di un software che si occupi della creazione e gestione dell'hardware
virtuale, la creazione e gestione delle singole virtual machines, l'allocazione delle risorse, la mappatura
I/O etc. Questo software è un elemento critico della virtualizzazione ed è chiamato *Virtual Machine Manager
(VMM)* o **hypervisor**. In breve, l'hypervisor viene eseguito direttamente sulla macchina fisica (*host*) e
ha il compito di gestire in ogni suo aspetto le macchine virtuali (*guest*). Esistono principalemente 2 tipi
di hypervisor, distinti sulla base del livello a cui operano rispetto all'hardware fisico (anche se la linea
di demarcazione non è così netta e potrebbe essere oggetto di dibattito...):

* **Type 1 Hypervisor**

  Il software dell'hypervisor viene eseguito direttamente sulla macchina fisica (in altre
parole, *è* il sistema operativo dell'*host*) ed interagisce direttamente con l'hardware. Alcuni esempi di questo
tipo sono *VMware ESXi*, *Citrix XenServer*, *Microsoft Hyper-V* e *KVM*, che di fatto rende il kernel della macchina
*host* un hypervisor. Questo tipo di hypervisor è detto anche *bare-metal*

<p align="center">
    <img src="images/type1.png" alt="Type 1 Hypervisor" width="600"/>
</p>

* **Type 2 Hypervisor**

  La macchina *host* ha un sistema operativo sul quale viene eseguito un software di
virtualizzazione. Hypervisor di questo tipo sono *Oracle VirtualBox* e *VMware Workstation/Player*

<p align="center">
    <img src="images/type2.png" alt="Type 2 Hypervisor" width="600"/>
</p>

## KVM vs QEMU vs LIBVIRT
