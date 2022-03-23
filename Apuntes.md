# **COMANDOS Y CONFIGURACIÓN BÁSICA**
-----------------------------------------------------------------------------------------------------------------------------------------------------

/etc/ansible/hosts (Archivo de configuración en Ubuntu )
    - Para que haya conexión de manera local se puede añadir abajo del todo un línea
      localhost <ansible_connetion=local>
    - Para añadir una conexión a un servidor por ejemplo SSH solo hay que añadir la dirección del servidor
    - Para conectarse con un usuario predeterminado podemos usar el parámetro ansible_user=<usuario> después de la dirección del servidor 
ansible <servidor(es)|grupo(s)> -m ping (Prueba de conexión con los servidores)
ansible <servidor(es)|grupo(s)> -a "comando" (Ejecuta un comando en el servidor)
  - Ejemplo conexión servidor con usuario vagrant (ansible 192.168.1.30 -u vagrant -m ping) 
  - Ejemplo conexión a todos los servidores con usuario vagrant (ansible all -u vagrant -m ping) 
Si necesitamos hacer uso de los permisos de root podemos añadir un --become
  - Ejemplo uso become
      - ansible 192.168.1.10 -a "adduser Cisco" --> ERROR
      - ansible 192.168.1.10 -a "adduser Cisco" --become --> Succesfully

# **CREACIÓN Y CONFIGURACIÓN DE GRUPOS**
-----------------------------------------------------------------------------------------------------------------------------------------------------

En el archivo /etc/ansible/hosts creamos el grupo entre corchetes
  - Ejemplo de creación de grupos
    #Archivo /hosts
    [ubuntu]
    192.168.1.10
    [debian]
    192.168.1.20

    $ ansible debian -m ping --> sería lo mismo que si yo usase el comando ansible 192.168.1.20 -m ping
    $ ansible ubuntu -m ping --> sería lo mismo que si yo usase el comando ansible 192.168.1.10 -m ping

Un servidor puedes estar en 1 o más grupos

Un grupo puede tener sub-grupos, estos se definen 
  - [S.O:children]
    ubuntu 
    debian

Un grupo puede tener variables 
  - [S.O:vars]
    ansible_become=TRUE
Ahora todas las veces que ejecutemos ansible con el grupo S.O se ejecutará la variable sudo
  $ ansible S.O -a whoami


Se puede separar las variables en ficheros
  - /etc/ansible/group_vars/<grupo>
  - /etc/ansible/group_vars/<servidor> 

# **EJEMEPLOS DE PARÁMETROS**
-----------------------------------------------------------------------------------------------------------------------------------------------------
ansible_connection | ssh    -->   | ansible_host
                   | local        | ansible_port
                                  | ansible_user
                                  | ansible_ssh_private_key_file

ansible_become | true    -->   | ansible_become_method -->  | su
               | false         |ansible_become_user         | sudo


Es pobile utilizar un inventario dinámico a partir de:
  - Proveedor cloud
    - AWS
    - GCP
    - Digitasl Ocean
  - Entornos propios:
    - Openstack
    - Ovist
    - Openshit
    - Zabbix
URL Scripts: https://github.com/ansible/ansible --> contrib/inventory

# **OPCIONES MODO ADHOC**
ansible [opciones] servidores|grupos|all [-m módulo] [-a argumentos]
          |  |
          |  |__| --limit srv1,srv2,srv3,...
          |     | --user
          |     | --become
          |     | -f (nº de conexiones simultaneas)
          |__| all --list-hosts (Lista todos los gosts)
             | -C (Comprueba si la tarea se realizará de manera satisfactoria)
             | -v (Más información, cuantas más v más información)

ansible [opciones] servidores|grupos|all [-m módulo] [-a argumentos]
                                              |
                                              |__| setup
                                                 | copy --> "src=/etc/hosts dest=/etc/ansible/hosts"
                                                 | yum/apt --> "name=vim state=present|latests|absent"

# **ADMINISTRAR SERVIDORES WINDOWS**

Para administrar servidores Windows necesitaremos la librería pywinrm, este se instala:
  - Descargando el instalador pip --> $ sudo apt install pip -y
  - Descargando la librería mediante este nuevo instalador --> $ sudo pip install pywinrm
  - Habilitar puerto 5486
  - Powershell 3.0 o superior
  - Habilitar control remoto (ConfigureRemotingForAnsible.ps1)
Para conectarnos al servidor Windows y probar conexión --> $ ansible -c winrm -k (Pide contraseña)-u <usuario> -m win_ping 
Tendremos que validar o ignorar el certificado
  - Para validarlo tendremos que copiar el certificado autosigned en nuestro servidor
  - Podemos ignorarlo añadiendo ansible_winrm_server_cert_validation=ignore
De manera final una línea de ejemplo podría ser:
  $ ansible -c winrm -k -u vagrant -e ansible_winrm_server_cert_validation=ignore -m win_ping 

# **PLAYBOOKS**

Se configuran en formato YAML
  - name: Mi primer playbook
    hosts: all
    remote_user: vagrant
    tasks:
      - name: Copiar fichero de hosts
        copy: 
          src: /etc/hosts
          dest: /etc/hosts
      - name:
        service: 
Para ejecutarlo se usa el comando
  $ ansible-playbook [opciones] fichero.yml
Para comprobar que un fichero yml está escrito de manera correcta tenemos que ejecutar el comando 
  $ ansible-playbook --syntax-check fichero.yml
Para listar las tareas en el archivo yml podemos usar 
  $ ansible-playbook --list-task fichero.yml 
Con el comando --step te pregunta paso a paso si quieres ejecutarlo
  $ ansible-playbook --step fichero.yml

# **TEMPLATES**
Expresiones: {{variable}} --> {{asible_fqdn}} mostrará el nombre completo del servidor
Control: {%...%} --> Condición:
                        {%if ansiboe_distribution == "Ubuntu"%}
Comentario: {#Comentario#} 

# **FICHERO DE ROLES**
  roles/
    nombre/
      files/
      templates/
      task/main.yml
      handlers/main.yml
      vars/main.yml
      defaults/main.yml
      meta/main.yml

# **PRIORIDAD DE VARIABLES**
Defaults definidas en un rol                                                                        menos
Variables de grupo (inventario -> group_vars/all -> group_vars/<grupo>)                               |
Variables de servidor (inventario -> host_vars/<servidor>)                                            |
"Facts" del servidor                                                                                  |
Variables del play (-> vars_prompt -> vars_files)                                                     |
Variables del role (Definidas en /roles/rol/vars/main.yml)                                            |
Variables de bloque -> Variables de tareas                                                            |
Parámetros role -> include-params -> include-vars                                                     |
Set-facts / regitered_vars                                                                            |
extra vars (Las introducida por línea de comando $ ansible <servidor> -e "nombre=Cisco" )            más

# **CONDICIONES**
-name: Instalar apache2
 apt: name=apache2 state=latest
 when: ansible_distribution == "Debian" or ansible_distribution == "Ubuntu"

# **BUCLES**
Para lista y disccionarios, utilizaremos la expresión with_items
  - name: Instalar software necesario
    apt: name={{ item }} state=latest
    with_items:
      - mariadb
      - php5
      - phpmyadmin

  - name: Crear usuarios necesarios
    user: name={{ item.nombre }} groups={{ item.grupo }}
    with_items:
      - { nombre: usuario1, grupo: www-data }
      - { nombre: usuario2, grupo: www-data }

# **EXPRESIÓN REGISTER**
Nos permite guardar en el valor de una variable el resultado de la acción realizada por un módulo en una tarea
- name: Playbook1
  hosts: localhost
  tasks:
  - name: Ejecutar comando
    command: uptime
    register: salida_uptime
  - name: Mostrar variable
    debug: var=salida_uptime

# **IGNORE ERRORS**
La expresión ignore_errors permite ignorar una tarea marcada como error y continuará con el resto de tareas
- name: Comprobar si fichero existe
  command: ls /noexiste.conf
  register: existe
  ignore_errors: true
- name: Mostrar si existe
  debug: existe


# **MÓDULOS**
Ansible está compuesto por muchos módulos, estoa administran sistemas, dispositivos, etc..
Cada tarea en un playbook está asociada a un módulo, los argumentos pueden ser obligatorios u opcionales.
Se dividen en 20 categorias:
- cloud           - files         - network               - storage
- clustering      - identitiy     - notification          - system
- commands        - inventory     - packaging             - utilites
- crypto          - messaging     - remote management     - web infrastructures
- database        - monitoring    - source control        - windows
Todos estos los podemos ver con el comando $ ansible-doc -l
