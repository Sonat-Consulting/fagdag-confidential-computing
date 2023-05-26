# fagdag-confidential-computing

## Forberedelse 1 - Start din egen ubuntu confidential maskin i Azure med Intel SGX

* Logg inn i azure med Sonat konto.
* Lag en ressurs gruppe
* Lag en Azure Confidential Computing VM i ressurs gruppen. Velg Ubuntu 20.04, SSH nøkkel, Og sett et navn du husker.
* Åpne for ssh (port 22)
* Last ned ssh nøkkel og sett riktige rettigheter, feks 0700. ```chmod 0700 <key>```

```
ssh -i <key> <user>@<ip>
```

### Forberedelse 2 - Installer git, ego, gdb, go
Nå er vi i OSet på maskinen.
```
sudo apt-get update
sudo apt install build-essential libssl-dev
sudo apt install golang-go
sudo apt-get install gdb
sudo snap install ego-dev --classic
```

### Oppgave 1 - Lag et go program med en verdi vi vil finne i minnet
Målet her er å lage et go program som holder en verdi som vi vil finne i minnet.
Et eksempel:
```
package main

import (
        "fmt"
        "os"
        "os/signal"
        "syscall"
)

type Data struct{num int64}

func main() {
        var b = new(Data)
        b.num = 2864434397
        fmt.Println("hello")
        sigs := make(chan os.Signal, 1)
        signal.Notify(sigs, syscall.SIGINT)
        _ = <-sigs //Vent på interrupt
        fmt.Println(b.num)
}
```
Nå kan vi bygge og kjøre programmet.
Lagret som hello_world.go så kan vi bygge slik:
```
go build hello_world.go
```
Og kjøre programmet i bakgrunnen slik:
```
./hello_world &
```
Nå har vi programmet kjørende i bakgrunnen, og vi vil hente ut minnet fra programmet.

Her er et praktisk lite script som gjør det med gdb.
Scripter tar PID som input, PID ble printet tidligere, men en enkel ```ps``` finner fort den:
```
#!/bin/bash
grep rw-p /proc/$1/maps \
| sed -n 's/^\([0-9a-f]*\)-\([0-9a-f]*\) .*$/\1 \2/p' \
| while read start stop; do \
    gdb --batch --pid $1 -ex \
        "dump memory $1-$start-$stop.dump 0x$start 0x$stop"; \
done
cat *.dump > $1.memory
rm -rf *.dump
```
Dersom scripter over er lagret som dumpmem.sh
Så kan det kjøres slik.
```
sudo sh dumpmem.sh <pid>
```

Praktisk nok så er 2864434397 i hex aabbccdd, her tolker vi filen som little-endian og søker etter hex verdien.
```
xxd -e <pid>.memory | grep 'aabbccdd'
```

### Oppgave 2
* Klon ego repo ``` git clone https://github.com/edgelesssys/ego.git ```
* Sjekk at det funger med å gå til ```cd samples/helloworld``` og gjøre:
```
ego-go build helloworld.go
ego sign helloworld
sudo ego run helloworld
```
* modifiser enclave.json, og sett ```debug=false
* Modifiser ```samples/hello_world/helloworld.go``` slik at den også holder et tall du kan lete etter i minnet.
* Prøv å finne tallet i minnet som over.
