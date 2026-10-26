PROYECTO ANSIBLE - LAMP / Samba / FTP / PostgreSQL
Sistema operativo destino: openSUSE 16.0

b) PASOS RAPIDOS PARA EJECUTAR EL PROYECTO

1. Tener dos maquinas openSUSE 16.0 en la misma red (maestra y destino).
2. En la maestra: sudo zypper install ansible git
3. ansible-galaxy collection install ansible.posix community.mysql community.postgresql
4. ssh-keygen -t ed25519
   ssh-copy-id publica@IP_DESTINO
5. git clone git@github.com:LitziMM-rgb/git-Aso.git
   cd git-Aso
6. Poner la IP de la maquina destino en inventory/hosts.yml, bajo servidor.hosts.
7. Tener la contrasena del vault (se entrega por separado).
8. ansible-playbook playbook.yml --ask-become-pass --ask-vault-pass
   - BECOME password: contrasena de sudo del usuario en la maquina destino
   - Vault password: contrasena del vault

c) COMANDOS USADOS EN EL PROYECTO

c.1) Red y maquinas virtuales (VMware)
- Las dos VM usan red NAT: VM > Settings > Network Adapter > NAT
- Reiniciar la conexion de red y ver la IP:
    nmcli device disconnect ens160
    nmcli device connect ens160
    ip a
- Probar internet y conexion entre las VM:
    ping -c 3 google.com
    ping -c 3 IP_DE_LA_OTRA_VM
- Cambiar el nombre de la maquina (necesita sudo):
    sudo hostnamectl set-hostname clon16
- Maestra: 192.168.71.128 (suse16)    Destino: 192.168.71.131 (clon16)

c.2) Git y GitHub
- Instalar y configurar git:
    sudo zypper install git
    git config --global user.name "Nombre"
    git config --global user.email "correo@ejemplo.com"
- Clave SSH para GitHub:
    ssh-keygen -t ed25519 -C "publica@suse16"
    cat ~/.ssh/id_ed25519.pub
  (pegar la clave en GitHub: Settings > SSH and GPG keys > New SSH key)
- Probar la conexion con GitHub:
    ssh -T git@github.com
- Clonar el repositorio:
    git clone git@github.com:LitziMM-rgb/git-Aso.git
    cd git-Aso

c.3) Ansible y acceso SSH sin contrasena
- Instalar Ansible y ver la version:
    sudo zypper install ansible
    ansible --version
- Ver las colecciones instaladas:
    ansible-galaxy collection list | grep -i posix
    ansible-galaxy collection list | grep -i mysql
    ansible-galaxy collection list | grep -i postgresql
- Copiar la clave a la maquina destino y probarla:
    ssh-copy-id publica@192.168.71.131
    ssh publica@192.168.71.131 hostname
- Probar que Ansible controla la maquina destino:
    ansible servidor -m ping

c.4) Archivos base
- inventory/hosts.yml (solo la IP, sin usuario ni contrasena):
    all:
      children:
        servidor:
          hosts:
            servidor:
              ansible_host: 192.168.71.131
- ansible.cfg (usuario y mejora del tiempo de respuesta):
    [defaults]
    inventory = inventory/hosts.yml
    roles_path = roles
    remote_user = publica
    host_key_checking = False
    forks = 10
    [ssh_connection]
    pipelining = True
    ssh_args = -o ControlMaster=auto -o ControlPersist=60s

c.5) Estructura del proyecto
- inventory/group_vars/all/vars.yml : variables globales sin contrasenas
- inventory/group_vars/all/vault*.yml: credenciales cifradas con Ansible Vault
- roles/<rol>/tasks/main.yml : tareas (instalar, copiar configuracion, activar servicio)
- roles/<rol>/handlers/main.yml : acciones que se ejecutan si algo cambia
- roles/<rol>/meta/main.yml : metadatos del rol
- roles/<rol>/templates/ : archivos de configuracion (vsftpd.conf.j2, smb.conf.j2, etc.)
- playbook.yml:
    - name: Instalar y configurar los servicios
      hosts: servidor
      become: yes
      roles:
        - lamp
        - mariadb
        - samba
        - ftp
        - postgresql

c.6) Contrasenas cifradas con ansible-vault
- Crear un archivo con la contrasena en texto y cifrarlo:
    mkdir -p inventory/group_vars/all
    echo 'vault_samba_password: "CLAVE"' > inventory/group_vars/all/vault.yml
    ansible-vault encrypt inventory/group_vars/all/vault.yml
- Archivos de vault del proyecto (dentro de inventory/group_vars/all):
    vault.yml     -> vault_samba_password
    vault_db.yml  -> vault_mariadb_root_password, vault_db_password
    vault_pg.yml  -> vault_pg_postgres_password, vault_pg_db_password
- Todos se cifran con la MISMA contrasena de vault.
- Ver el contenido de un archivo cifrado:
    ansible-vault view inventory/group_vars/all/vault.yml
- Un archivo cifrado empieza con: $ANSIBLE_VAULT;1.1;AES256

c.7) Comprobaciones del despliegue
- Primero, desde la carpeta del proyecto en la maquina maestra, comprobar
  inventario y conexion:
    ansible-inventory --graph
    ansible servidor -m ping
- Comprobar la sintaxis del playbook (solicitara la contrasena de Vault):
    ansible-playbook playbook.yml --syntax-check --ask-vault-pass
- Despues de desplegar, comprobar los servicios desde la maestra:
    ssh publica@192.168.71.131 "sudo systemctl is-active apache2 mariadb smb vsftpd postgresql"
- FTP
    ssh publica@192.168.71.131 "rpm -q vsftpd"
- samba (usuario del sistema, usuario Samba, directorio y create mask)
    ssh publica@192.168.71.131 "grep -E 'create mask|directory mask' /etc/samba/smb.conf"
    ssh publica@192.168.71.131 "ls -ld /srv/samba/compartido"
    ssh -t publica@192.168.71.131 "sudo pdbedit -L"
- LAMP (Apache, PHP, librerias para MariaDB y PostgreSQL, phpMyAdmin)
    ssh publica@192.168.71.131 "rpm -q php8 apache2-mod_php8 php8-mysql php8-pgsql phpMyAdmin"
    curl -I http://192.168.71.131/
    curl -I http://192.168.71.131/phpMyAdmin/
- MariaDB (base de datos appdb y usuario appuser)
    ssh -t publica@192.168.71.131 "mariadb -u root -p -e 'SHOW DATABASES;'"
- PostgreSQL (usuario pguser, base appdb_pg, acceso remoto cifrado)
  Ejecutar desde otra maquina de la red con psql instalado. La IP de origen
  debe estar dentro de pg_allowed_network en inventory/group_vars/all/vars.yml:
    psql "host=192.168.71.131 port=5432 dbname=appdb_pg user=pguser sslmode=require" -W -c "SELECT ssl, version, cipher FROM pg_stat_ssl WHERE pid = pg_backend_pid();"
  La columna ssl debe mostrar t. La contrasena se solicita de forma interactiva.
- Puertos abiertos en el firewall de la maquina destino:
    ssh -t publica@192.168.71.131 "sudo firewall-cmd --list-services"
  (debe incluir ftp, http, samba, postgresql y ssh)

c.8) Ejecucion del playbook
    ansible-playbook playbook.yml --ask-become-pass --ask-vault-pass
- Resultado esperado al final (PLAY RECAP): failed=0
- ok = ya estaba asi; changed = hizo un cambio; failed = error
- PostgreSQL inicializa su cluster cuando aun no existe, genera un certificado
  autofirmado y configura el acceso remoto solo mediante conexiones SSL.
- Si una comprobacion falla, revisar el estado y los registros del servicio en
  el destino, por ejemplo:
    sudo systemctl status postgresql
    sudo journalctl -u postgresql -n 50 --no-pager

c.9) Flujo de git (uno por cada avance)
    git add archivo_o_carpeta
    git status
    git commit -m "Mensaje que describe el cambio"
    git push
- git status debe terminar en: nothing to commit, working tree clean

d) ROLES INCLUIDOS
lamp        Apache, PHP, librerias MariaDB/PostgreSQL y phpMyAdmin
mariadb     MariaDB (contrasena de root, base de datos y usuario)
samba       Recurso compartido (usuario del sistema, usuario Samba, directorio, create mask)
ftp         Servidor FTP (vsftpd)
postgresql  PostgreSQL (contrasena, usuario, base de datos, conexion remota con SSL)

e) PLANTILLAS
roles/ftp/templates/vsftpd.conf.j2
roles/samba/templates/smb.conf.j2
roles/lamp/templates/phpmyadmin.conf.j2
roles/lamp/templates/phpinfo.php.j2
roles/postgresql/templates/pg_hba.conf.j2

f) MEJORAS
- inventario sin usuario ni contrasena visibles (el usuario SSH esta en ansible.cfg)
- ansible.cfg con pipelining y ControlPersist (mejor tiempo de respuesta)
- Contrasenas en archivos cifrados con ansible-vault
- Uso de roles de Ansible para LAMP, MariaDB, Samba, FTP y PostgreSQL
- Puertos del firewall abiertos desde los propios roles

g) ERRORES COMUNES Y SOLUCION
- "Host key verification failed": al preguntar, escribir yes completo (no solo y).
- "Permission denied (publickey)" con GitHub: la clave no esta agregada en GitHub.
- "No route to host": la maquina destino cambio de IP o esta apagada. Revisar con ip a.
- phpMyAdmin da 404: falta la configuracion de Apache. Se soluciona con la plantilla
  roles/lamp/templates/phpmyadmin.conf.j2.
- phpMyAdmin no responde desde otra maquina: el firewall bloquea el puerto 80.
- El paquete python3-PyMySQL no existe en openSUSE 16.0: se llama python313-PyMySQL.
- mysql falla la segunda vez: root ya tiene contrasena. Se pasa login_user y
  login_password en las tareas.
- Al copiar comandos, no copiar el texto "publica@suse16:~/git-Aso>": la terminal crea
  archivos con nombres raros (cat, ls, git). Se borran con: rm cat ls git
- "tail: option used in invalid context": usar tail -n 8 en vez de tail -8.

h) CONCEPTOS CLAVE
- Inventario (hosts): lista de maquinas que Ansible va a configurar.
- Playbook: lista de roles que se aplican a un grupo de maquinas.
- Rol: carpeta con tareas, variables, plantillas y handlers de un servicio.
- Handler: accion que solo se ejecuta si hubo un cambio (por ejemplo reiniciar).
- notify: avisa a un handler que debe ejecutarse.
- become: ejecutar con permisos de administrador.
- no_log: evita mostrar contrasenas en pantalla.
- Idempotencia: ejecutar dos veces da el mismo resultado; lo ya hecho sale ok.
- ansible-vault: cifra archivos con contrasenas.
- create mask (Samba): permisos de los archivos nuevos del recurso compartido.
- hostssl (pg_hba.conf): acepta conexiones remotas solo si usan SSL.
