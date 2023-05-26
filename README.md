# **Configuration d'apache2 avec un script YML**

## Etape1. Configuration d'ansible

- Verifier qu'ansible est bien installée, 
ensuite:

- Creer un nouveau repertoire:
```sh
mkdir ansible-apache
```

- Déplacez-vous dans le nouveau répertoire:
```sh
cd ~/ansible-apache/
```

- Creez un nouveau fichier ansible.cfg et copier ce qui suit:
```sh
nano ansible.cfg
```

```sh
[defaults]
hostfile = hosts
```

Pour commencer, faisons quelque chose de simple:

Creez un fichier hosts
```sh
nano hosts
```

Copier ce qui suit:
```sh
[apache]
secondary_server_ip ansible_ssh_user=username
```

Pour verifier qu'il peut se connecter à chaque hote, executer un commande de base ansible:
```sh
ansible apache -m ping
```

```sh
111.111.111.111 | success >> {
    "changed": false,
    "ping": "pong"
}
```

## Étape 2 - Création d'un livre de lecture
- Creez un nouveau fichier `apache.yml` et copiez ce qui suit:
```sh
---
- hosts: apache
  tasks:
    - name: run echo command
      command: /bin/echo hello sammy
```
Il suffit ensuite d'executer le playbook:
`ansible-playbook apache.yml`

## Étape 3 - Installation d'Apache
name : le nom du package à installer, soit un nom de package unique, soit une liste de packages.
state : Accepte soit latest, absent ou present. Latest s'assure que la dernière version est installée, present vérifie simplement qu'elle est installée et absent la supprime si elle est installée.
update_cache : met à jour le cache (via apt-get update) si activé, pour s'assurer qu'il est à jour.

- Mets à jour le playbook `apache.yml`
`nano apache.yml`

- Supprimez le texte qui s'y trouve et copiez ce qui suit:
```sh
---
- hosts: apache
  sudo: yes
  tasks:
    - name: install apache2
      apt: name=apache2 update_cache=yes state=latest
```
`Apt` installe le package apacha2 et s'assure que tu as mis à jour le cache (`update_cache=yes`)

- Executer maintenant le playbook:
`ansible-playbook apache.yml --ask-sudo-pass`

`--ask-sudo-pass` demandera les mots de passe du root car c'est nécessaire pour les privilèges root.

## Étape 4 - Configuration des modules Apache
- Ouvrez `apache-yml` pour le modifier
`nano apache.yml`

- Redemarrez ensuite apache2 mais pour eviter de faire cela à chaque fois, nous allons utiliser un gestionnaire de tache.
Pour ce faire, nous devons ajouter l'option `notify` dans la tâche `apache2_module`, puis nous pouvons utiliser le module de service pour redémarrer `apache2` dans un gestionnaire .

Cela se traduit par ceci:
```sh
---
- hosts: apache
  sudo: yes
  tasks:
    - name: install apache2
      apt: name=apache2 update_cache=yes state=latest

    - name: enabled mod_rewrite
      apache2_module: name=rewrite state=present
      notify:
        - restart apache2

  handlers:
    - name: restart apache2
      service: name=apache2 state=restarted
```

- Relancez le playbook:
`ansible-playbook apache.yml --ask-sudo-pass`

## Étape 5 - Configuration des options Apache
Par défaut, Apache écoute sur le port 80 tout le trafic HTTP et en supposant qu'apache ecoute sur le port 8081.

- Avec la configuration Apache par défaut sur Ubuntu 14.04 x64, deux fichiers doivent être mis à jour :
```sh
/etc/apache2/ports.conf
    Listen 80

/etc/apache2/sites-available/000-default.conf
    <VirtualHost *:80>
```

- Pour ce faire, nous pouvons utiliser le module lineinfile. Pour cet exemple nous utiliserons le module suivante:
    dest - Le fichier à mettre à jour dans le cadre de la commande.
    regexp - Expression régulière à utiliser pour correspondre à une ligne existante à remplacer.
    line - La ligne à insérer dans le fichier, soit en remplacement de la ligne regexp, soit en tant que nouvelle ligne à la fin.
    état : soit présent, soit absent.

Ce que nous devons faire pour mettre à jour le port de 80 à 8081 est de rechercher les lignes existantes qui définissent le port 80 et de les modifier pour définir port 8081.

- Ouvrez le fichier `apache.yml` pour le modifier.
`nano apache.yml`
Modifier les lignes supplementaires qui suit:
```sh
---
- hosts: apache
  sudo: yes
  tasks:
    - name: install apache2
      apt: name=apache2 update_cache=yes state=latest

    - name: enabled mod_rewrite
      apache2_module: name=rewrite state=present
      notify:
        - restart apache2

    - name: apache2 listen on port 8081
      lineinfile: dest=/etc/apache2/ports.conf regexp="^Listen 80" line="Listen 8081" state=present
      notify:
        - restart apache2

    - name: apache2 virtualhost on port 8081
      lineinfile: dest=/etc/apache2/sites-available/000-default.conf regexp="^<VirtualHost \*:80>" line="<VirtualHost *:8081>" state=present
      notify:
        - restart apache2

  handlers:
    - name: restart apache2
      service: name=apache2 state=restarted
```
Bien sur, dans ce processus nous devons redemarrer `apache2`

- Exécutez maintenant le playbook:
```sh
ansible-playbook apache.yml --ask-sudo-pass
```

## Étape 6 - Configuration des hôtes virtuels
- Créer une configuration d'hôte virtuel

La première étape consiste à créer une nouvelle configuration d'hôte virtuel. Nous allons créer le fichier de configuration de l'hôte virtuel sur le Droplet principal et le télécharger sur le Droplet secondaire à l'aide d'Ansible.

Voici un exemple de configuration d'hôte virtuel de base que nous pouvons utiliser comme point de départ pour notre propre configuration. Notez que le numéro de port et le nom de domaine, mis en évidence ci-dessous, sont codés en dur dans la configuration.

```sh
<VirtualHost *:8081>
    ServerAdmin webmaster@example.com
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example.com
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Créez un nouveau fichier appelé `virtualhost.conf`.
```sh
nano virtualhost.conf
```

```sh
<VirtualHost *:{{ http_port }}>
    ServerAdmin webmaster@{{ domain }}
    ServerName {{ domain }}
    ServerAlias www.{{ domain }}
    DocumentRoot /var/www/{{ domain }}
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- Utiliser des variables de modèle

La première étape consiste à ajouter une section dans le playbook pour les variables.
Il s'appelle `vars` et va au même niveau que `hosts`, `sudo`, `tasks` et `handlers`.
```sh
---
- hosts: apache
  sudo: yes
  vars:
    http_port: 80
    domain: example.com
  tasks:
    - name: install apache2
...
```

La variable doit être ajoutée dans la ligne et l'option regexp doit être mise à jour afin qu'elle ne recherche pas un port spécifique. Les changements ressembleront à ceci :
```sh
lineinfile: dest=/etc/apache2/ports.conf regexp="^Listen " line="Listen {{ http_port }}" state=present
lineinfile: dest=/etc/apache2/sites-available/000-default.conf regexp="^<VirtualHost \*:" line="<VirtualHost *:{{ http_port }}>"
```

- Ajouter un module de modèle
L'étape suivante consiste à ajouter le module de modèle pour pousser le fichier de configuration sur l'hôte. Nous utiliserons ces options pour y arriver :
    dest – Le chemin du fichier de destination pour enregistrer le modèle mis à jour sur le ou les hôtes, c'est-à-dire /etc/apache2/sites-available/{{ domain }}.conf.
    src - Le fichier de modèle source, c'est-à-dire virtualhost.conf.
```sh
- name: a2ensite {{ domain }}
  command: a2ensite {{ domain }}
  notify:
  - restart apache2
```

- Activer l'hôte virtuel
Executer la commande:
```sh
sudo a2ensite example.com
```
ou en liant manuellement le fichier de configuration dans
```sh
/etc/apache2/sites-enabled/
```
Pour cela, le module `command` est à nouveau utilisé.
```sh
- name: a2ensite {{ domain }}
  command: a2ensite {{ domain }}
  notify:
  - restart apache2
```

- Empêcher le travail supplémentaire
Dans notre cas, il ne doit être exécuté que si le fichier `.conf` n'a pas encore été créé sur l'hôte.
Les changements ressembleront à ceci:
```sh
- name: a2ensite {{ domain }}
  command: a2ensite {{ domain }}
  args:
    creates: /etc/apache2/sites-enabled/{{ domain }}.conf
  notify:
  - restart apache2
```
Il est important de noter l'utilisation de la section args dans la tâche. Il s'agit d'une manière facultative de lister les options de module, et dans ce cas supprime toute confusion entre ce qu'est une option de module et ce qu'est la commande elle-même.

Playbook apache.yml final
Ouvrez apache.yml:
```sh
nano apache.yml
```

modifier votre playbook `apache.yml` avec ce qui suit:
```sh
---
- hosts: apache
  sudo: yes
  vars:
    http_port: 80
    domain: example.com
  tasks:
    - name: install apache2
      apt: name=apache2 update_cache=yes state=latest

    - name: enabled mod_rewrite
      apache2_module: name=rewrite state=present
      notify:
        - restart apache2

    - name: apache2 listen on port {{ http_port }}
      lineinfile: dest=/etc/apache2/ports.conf regexp="^Listen " line="Listen {{ http_port }}" state=present
      notify:
        - restart apache2

    - name: apache2 virtualhost on port {{ http_port }}
      lineinfile: dest=/etc/apache2/sites-available/000-default.conf regexp="^<VirtualHost \*:" line="<VirtualHost *:{{ http_port }}>"
      notify:
        - restart apache2

    - name: create virtual host file
      template: src=virtualhost.conf dest=/etc/apache2/sites-available/{{ domain }}.conf

    - name: a2ensite {{ domain }}
      command: a2ensite {{ domain }}
      args:
        creates: /etc/apache2/sites-enabled/{{ domain }}.conf
      notify:
        - restart apache2

  handlers:
    - name: restart apache2
      service: name=apache2 state=restarted
```

- Executer le playbook:
```sh
ansible-playbook apache.yml --ask-sudo-pass
```
## Étape 7 - Utilisation d'un référentiel Git pour votre site Web
Dans cet exercice, nous utiliserons Ansible pour cloner un référentiel Git afin de configurer le contenu de votre site Web.

Le module git a beaucoup d'options, celles qui sont pertinentes pour ce tutoriel étant :
    dest : le chemin sur l'hôte vers lequel le référentiel sera extrait.
    repo - L'URL du référentiel qui sera cloné. Cela doit être accessible par l'hôte.
    update – Lorsqu'il est défini sur no, cela empêche Ansible de mettre à jour le référentiel lorsqu'il existe déjà.
    accept_hostkey - Indique à SSH d'accepter toute clé d'hôte inconnue lors de la connexion via SSH. Ceci est très utile car cela évite d'avoir à se connecter via SSH pour accepter la première tentative de connexion, mais cela supprime la possibilité de vérifier manuellement la signature de l'hôte. Selon votre dépôt, vous aurez peut-être besoin de cette option.

 Dans cet esprit, la tâche git ressemblera à ceci :

```sh
name: clone basic html template
git: repo=https://github.com/do-community/ansible-apache-tutorial.git dest=/var/www/example.com update=no
```
Mais avant cela, installer le package `git` afin qu'ansible puisse pour cloner le referentiel.

```sh
name: install packages
 apt: name={{ item }} update_cache=yes state=latest
  with_items:
    - apache2
    - git
```

Ouvrez une derniere fois `apache.yml`:
```sh
nano apache.yml
```
Mets à jour le playbook avec ce qui suit
```sh
---
- hosts: apache
  sudo: yes

  vars:
    http_port: 80
    domain: example.com

  tasks:

    - name: install packages
      apt: name={{ item }} update_cache=yes state=latest
      with_items:
        - apache2
        - git

    - name: enabled mod_rewrite
      apache2_module: name=rewrite state=present
      notify:
        - restart apache2

    - name: apache2 listen on port {{ http_port }}
      lineinfile: dest=/etc/apache2/ports.conf regexp="^Listen " line="Listen {{ http_port }}" state=present
      notify:
        - restart apache2

    - name: apache2 virtualhost on port {{ http_port }}
      lineinfile: dest=/etc/apache2/sites-available/000-default.conf regexp="^<VirtualHost \*:" line="<VirtualHost *:{{ http_port }}>"
      notify:
        - restart apache2

    - name: create virtual host file
      template: src=virtualhost.conf dest=/etc/apache2/sites-available/{{ domain }}.conf

    - name: a2ensite {{ domain }}
      command: a2ensite {{ domain }}
      args:
        creates: /etc/apache2/sites-enabled/{{ domain }}.conf
      notify:
        - restart apache2

    - name: clone basic html template
      git: repo=https://github.com/do-community/ansible-apache-tutorial.git dest=/var/www/example.com update=no

  handlers:
    - name: restart apache2
      service: name=apache2 state=restarted
```

Enregistrez le fichier et exécutez le playbook.

```sh
ansible-playbook apache.yml --ask-sudo-pass
```
