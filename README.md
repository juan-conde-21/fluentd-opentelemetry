# Fluentd - Opentelemetry - Instana (Logs) - Centos 7

![image](https://github.com/juan-conde-21/fluentd-opentelemetry/assets/13276404/3f63e53c-1395-4778-8a3d-d18633c54e8d)


# Instalar opentelemetry collector Contrib 

(Soporta fluentforward para recepcion de logs desde fluentd, no confundir con collector generico)

1. Descargar y ejecutar la instalación del collector

       yum install wget -y
       wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.99.0/otelcol-contrib_0.99.0_linux_amd64.rpm
       rpm -ivh otelcol-contrib_0.99.0_linux_amd64.rpm
       systemctl status otelcol-contrib

1. Respaldar el archivo de configuración

       cd /etc/otelcol-contrib/
       cp -p config.yaml config.yaml.orig

2. Modificar el archivo de configuracion para recibir logs de fluentd y habilitar el procesador de logs.

       vi config.yaml

    Reemplazar el contenido con la siguiente configuracion:

       receivers:
         fluentforward:
           endpoint: 0.0.0.0:8006
        
       processors:
         batch:
         transform:
           error_mode: ignore
           log_statements:
             - context: log
               statements:
                 - set(severity_number, SEVERITY_NUMBER_ERROR) where IsString(body) and IsMatch(body, ".*Fallida.*")
        
       exporters:
         debug:
           verbosity: detailed
        
         otlp:
           endpoint: localhost:4317
           tls:
             insecure: true
        
       service:
        
         pipelines:
        
           #traces:
           #  receivers: [otlp, opencensus, jaeger, zipkin]
           #  processors: [batch]
           #  exporters: [debug]
        
           #metrics:
           #  receivers: [otlp, opencensus, prometheus]
           #  processors: [batch]
           #  exporters: [debug]
        
           logs:
             receivers: [fluentforward]
             processors: [batch, transform]
             exporters: [debug, otlp]

3. Reiniciar servicios del colector

       systemctl restart otelcol-contrib
       systemctl status otelcol-contrib

# Instalar agente Instana

1. Descargar el instalador, reemplazar los valors de agent_key y tenant_location por los correspondientes para su tenant.

       curl -o setup_agent.sh https://setup.instana.io/agent && chmod 700 ./setup_agent.sh && sudo ./setup_agent.sh -a {agent_key} -d {agent_key} -t dynamic -e {tenant_location}.instana.io:443 -s -y

2. Editar el archivo de configuracion y habilitar el plugin de opentelemetry.

       vi /opt/instana/agent/etc/instana/configuration.yaml

   A nivel del archivo ubicar la seccion de opentelemtry y modificar de acuerdo como se muestra.

       com.instana.plugin.opentelemetry:
         grpc:
           enabled: true
         http:
           enabled: true

3. Reiniciar los servicios del agente.

       systemctl restart instana-agent


# Instalar fluentd

1. Descargar y ejecutar instalador

       curl -fsSL https://toolbelt.treasuredata.com/sh/install-redhat-fluent-package5-lts.sh | sh
    
2. Generar carpeta de logs y habilitar permisos.

       mkdir -p /var/log/fluentd
       chmod -R 775 /var/log/fluentd
       chown -R fluentd.fluentd /var/log/fluentd/
 
3. Backup y modificacion del archivo de configuracion

       cp -p /etc/fluent/fluentd.conf /etc/fluent/fluentd.conf.orig
       vi /etc/fluent/fluentd.conf

   Colocar la siguiente configuración, puede variar dependiendo de las rutas donde se ubiquen los archivos.


       <source>
         @type tail
         path /opt/app/application.log
         pos_file /var/log/fluentd/application.log.pos
         tag application.log
         <parse>
           @type none
         </parse>
       </source>
       
       <match application.log>
         @type forward
         <server>
           host localhost
           port 8006
         </server>
         <buffer>
           flush_interval 10s
         </buffer>
       </match>
        
       <source>
         @type tail
         path /opt/app/transaction_logs.log
         pos_file /var/log/fluentd/transaction.log.pos
         tag transaction.log
         read_from_head true
         <parse>
           @type multiline
           format_firstline /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) INFO TransactionService - Inicio de transaccion: ID=(?<id>\d+), Estado=(?<estado>.+)$/
           format1 /^(?<message>.+)$/
           multiline_flush_interval 5
         </parse>
       </source>
        
       <match transaction.log>
         @type forward
         <server>
           host localhost
           port 8006
         </server>
         <buffer>
           flush_interval 10s
         </buffer>
       </match>


  *Guardar las modificaciones

4. Reinciar servicio de fluentd

       systemctl restart fluentd
       systemctl status fluentd

# Instalar el plugin para el soporte de logs multilinea

1. Descargar instalador y ejecutar el source para las variables de entorno

       gpg2 --keyserver hkp://pgp.mit.edu --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
        
       curl -sSL https://get.rvm.io | bash -s stable 
       source /etc/profile.d/rvm.sh

       rvm install 3.0.0
       rvm use 3.0.0 --default

2. Verificar la version de Ruby instalada

       ruby --version

3. Instalar plugin multilinea

       gem install fluent-plugin-multi-format-parser


# Crear los scripts para la generacion de logs de prueba

1. Crear los directorios utilizados

       mkdir /opt/app
       mkdir /opt/Proyecto

2. Crear el script para la generacion de logs de una sola linea

       cd /opt/Proyecto 
       vi genera_simple_log.sh

3. Agregar el siguiente contenido al script.

       #!/bin/bash
      
       # Ruta del archivo de log
       LOG_FILE="/opt/app/application.log"
      
       # Función para escribir logs
       log_message() {
           local log_type=$1
           local log_msg=$2
           echo "$(date '+%Y-%m-%d %H:%M:%S') $log_type - $log_msg" >> "$LOG_FILE"
       }
      
       # Ejemplo de escritura de logs
       log_message "INFO" "Inicio de la aplicación."
       log_message "INFO" "Procesando datos..."
       log_message "WARNING" "El archivo de configuración está desactualizado."
       log_message "ERROR" "No se pudo encontrar el archivo de datos requerido."
       log_message "INFO" "Proceso completado exitosamente."
      
       echo "Logs escritos en $LOG_FILE"

4. Guardar el archivo y agregar permisos de ejecucion.

       chmod +x genera_simple_log.sh

5. Crear el script para generacion de logs multilinea.

       vi genera_multi_log.sh

6. Agregar el siguiente contenido al script.

       #!/bin/bash
        
       # Archivo de log
       LOG_FILE="/opt/app/transaction_logs.log"
        
       # Limpiar el archivo de log anterior
       #> $LOG_FILE
        
       # Función para escribir un log
       write_log() {
           echo "$(date '+%Y-%m-%d %H:%M:%S') $1 $2 - $3, Estado=$4" >> $LOG_FILE
       }
        
       # Simular múltiples transacciones
       for i in {1..10}; do
           transaction_id=$(($RANDOM % 10000 + 1000))
           payment_amount=$(($RANDOM % 100 + 50)).00
           payment_method=$(( $RANDOM % 2 ))
        
           if [ $payment_method -eq 0 ]; then
               payment_method="Tarjeta de Credito"
           else
               payment_method="Transferencia Bancaria"
           fi
        
           # Estados posibles en cada paso
           estados=("Iniciada" "Procesando" "Advertencia" "Exito" "Completada" "Fallida")
        
           # Inicio de transacción
           write_log "INFO" "TransactionService" "Inicio de transaccion: ID=$transaction_id" "${estados[0]}"
           sleep 1
           write_log "INFO" "PaymentProcessor" "Procesando pago: Monto=$payment_amount USD, Metodo=$payment_method" "${estados[1]}"
           sleep 1
        
           # Decidir al azar si hay una advertencia
           if [ $(($RANDOM % 5)) -eq 3 ]; then
               write_log "WARN" "PaymentValidator" "Advertencia: La tarjeta esta proxima a expirar" "${estados[2]}"
               sleep 1
           fi
        
           # Resultado de la transacción
           if [ $(($RANDOM % 10)) -gt 2 ]; then
               write_log "INFO" "PaymentProcessor" "Pago procesado correctamente: ID Transaccion=$transaction_id" "${estados[3]}"
               final_state="${estados[4]}"
           else
               final_state="${estados[5]}"
           fi
           sleep 1
           write_log "INFO" "TransactionService" "Transaccion ${final_state}: ID=$transaction_id" "$final_state"
           sleep 1
       done
        
       echo "Logs generados en $LOG_FILE"

7. Guardar el archivo y agregar permisos de ejecucion.

       chmod +x genera_multi_log.sh


# Ejecutar scripts de prueba y verificacion en Instana

1. Ejecutar las pruebas con el script de log simple

       /opt/Proyecto/genera_simple_log.sh

2. Verificar en Instana, para ello ingresar a la seccion de Analytics y seleccionar logs. Se deberan mostrar los logs identificadors por opentelemetry-stream, tambien podremos filtrar por la zona del agente o contenido del log.

   ![image](https://github.com/juan-conde-21/fluentd-opentelemetry/assets/13276404/46373729-f151-46a3-a177-af056b757153)

3. Ejecutar las pruebas con el script de log multilinea.

       /opt/Proyecto/genera_multi_log.sh

4. Verificar en Instana, para ello ingresar a la seccion de Analytics y seleccionar logs. Se deberan mostrar los logs identificadors por opentelemetry-stream, tambien podremos filtrar por la zona del agente o contenido del log.

   ![image](https://github.com/juan-conde-21/fluentd-opentelemetry/assets/13276404/d771e918-4811-4598-bc93-04bea8d9de1c)

