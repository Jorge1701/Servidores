# Configurar Filtro de Correo

Crear archivo */var/mail/filtro.sh* con el siguiente contenido
``` bash
#!/bin/sh

# Simple shell-based filter. It is meant to be invoked as follows:
#       /path/to/script -f sender recipients...

# Localize these. The -G option does nothing before Postfix 2.3.
INSPECT_DIR=/var/spool/filtro
SENDMAIL="/usr/sbin/sendmail -G -i" # NEVER NEVER NEVER use "-t" here.

# Exit codes from <sysexits.h>
EX_TEMPFAIL=75
EX_UNAVAILABLE=69

# Clean up when done or when aborting.
trap "rm -f in.$$" 0 1 2 3 15

# Start processing.
cd $INSPECT_DIR || {
    echo $INSPECT_DIR does not exist; exit $EX_TEMPFAIL; }

cat >in.$$ || { 
    echo Cannot save mail to file; exit $EX_TEMPFAIL; }

./filtro.sh in.$$

$SENDMAIL "$@" <in.$$

exit $?
```

Crear un archivo */var/spool/filtro/filtro.sh* con el contenido
``` bash
#!/bin/bash

rm mail.txt

while read -r line
do
   echo "$line" >> mail.txt

done < <(cat "$@")

subjectOK=""
bodyOK=""
espacio=""
from=""
to=""

while read linea
do
        #Si existe un linea vacia se chequea el body
        if [[ $espacio == "yes" ]]
        then
                if [[ $linea == *"Tecnologo en Informatica"* ]]
                then
                        bodyOK="yes"
                        break
                fi
        fi

        #Obtener el asunto
        if [ "${linea:0:7}" == "Subject" ]
        then
                subject="$(echo $linea | cut -f2 -d':')"
                if [[ $subject == *"TIP 2017"* ]]
                then
                        subjectOK="yes"
                fi
        fi

        #Obtener el remitente
        if [ "${linea:0:4}" == "From" ]
        then
            from="$(echo $linea | cut -f2 -d' ')"
        fi

        #Obtener el destino
        if [ "${linea:0:3}" == "for" ]
        then
            toTemp="$(echo $linea | cut -f2 -d' ')"
            chars="$(echo $toTemp | fold -w1)"
            for char in $chars
            do
                if [[ $char != "<" && $char != ">" && $char != ";" ]]
                then
                        to=$to"$char"
                fi
            done
        fi

        #Si hay una linea vacia se se coloca yes en espacio para indicar
        #que comienza el body
        if [ "${linea:0}" == "" ]
        then
                espacio="yes"
        fi

done < mail.txt

fecha=$(date "+%d_%m_%Y-%H_%M_%S")

if [[ $subjectOK == "yes" && $bodyOK == "yes" ]]
then
        mv mail.txt /var/mail/data_loggin/$fecha-$from-$to.txt
else
        rm mail.txt
fi
```

Editar el archivo */etc/postfix/master.cf* (*NOTA: la segunda linea ya esta, falta agregarle el content_filter*).
```
filtro unix - n n - 10 pipe flags=Rq user=filtro null_sender= argv=/var/mail/filtro.sh -f ${sender} -- ${recipient}

smtp      inet  n       -       n       -       -       smtpd -o content_filter=filtro:filtro
```

Luego ejecutar
```
# groupadd filtro
# useradd -g filtro filtro
# chown -R filtro:filtro /var/spool/filtro
# chown -R filtro:filtro /var/mail/filtro.sh
# chown -R filtro:filtro /var/mail/data_loggin
# chmod 777 /var/spool/filtro/filtro.sh
# chmod 777 /var/mail/filtro.sh
# chmod 777 /var/mail/data_loggin/
```

### Aplicar cambios
```
# postfix reload
# dovecot reload
```
