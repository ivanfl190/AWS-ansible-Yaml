
### Passo a passo de como criar uma instancia na Aws com YAML utilizando o ansible.

**<p>Configuração dos credenciais da Amazon Web Services:</p>**
 Primeiramente deve conter instalado o CLI: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
 em seguida, ao termino da instalação do  CLI ,deve ser colocado as informações de seu token da aws.
 Utilizando o comando  vim ~/.aws/credentials, assim substituindo ou adicionando o conteúdo por seu token da CLI da AWS.                                  
  Com a configuração acima feita, deve se abrir o o terminal onde o arquivo .Yaml se encontra e utilizar o comando:
 ansible-playbook -i hosts nome_do_arquivo.yaml para que seja feito o processo. 
 
 **<p>Sobre o código:</p>**
 a estrutura do código e basicamente a mesa estrutura de uma linguagem yaml, seguindo os mesmos padrões e espaçamentos.
 

A parte inicial do código yaml, onde foi definido o host, e o gather_facts é para entregar as informações do host para o sistema remoto.

    - name: Iniciando
     hosts: localhost
     gather_facts: no
   
     
Definiçao de qual usuario será, qual grupo  e qual  regiao do AWS a maquina será hospedada.
onde o profile que se refere, é o profile do CLI do AWS.


   vars:  
       - profile: Default
       - user: ansible
       - group: ansible
       - regiao: us-east-1


Criando uma vpc na AWS , onde irá adicionar a faixa de ip acima, e adicionando o profile que foi definido no escopo vars. O register ,e uma variavel que armazena tudo o que foi criado nas linhas anteriores.


 tasks:
       - name: Criando VPC em AWS
         ec2_vpc_net:
           name: ansiblelab
           cidr_block: 10.10.0.0/16
           profile: "{{ profile }}"
           region: "{{ regiao }}"
         register: vpcl



Em seguida é criado a  subnet, que será registrado como subwb e terá a tag de Webserver subnet, para identificar. Em vpc_id, será adicionando o register anterior.
 
 - name: Criando Subnet
         ec2_vpc_subnet:
           profile: "{{ profile }}"
           region: "{{ regiao }}"
           vpc_id: "{{ vpcl.vpc.id }}"
           cidr: 10.10.1.0/24
           tags:
             Name: Webserver Subnet
         register: subwb

Ao criar o security group , ele será registrado pra porta 22 , o qual será liberado pra SSH, com qualquer IP. 

- name: Criando security group
         ec2_group: 
           name: SG-SSH
           description: Acess SSH
           profile: "{{ profile }}"
           vpc_id: "{{ vpcl.vpc.id }}"
           region: "{{ regiao }}"
           rules:
           - proto: tcp
             from_port: 22
             to_port: 22
             cidr_ip: 0.0.0.0/0
             rule_desc: SSH
           rules_egress: 
           - proto: all
             cidr_ip: 0.0.0.0/0
         register: secgr

Criando a chave de acesso, com isso o conteúdo do copy irá criar a chave e move la para a pasta onde o playbook esta sendo executado.


 - name: Criando chave
         ec2_key:
           name: hostsweb
           region: "{{ regiao }}"
           profile: "{{ profile }}"
           copy: content="{{ key_pair_hostsweb.key.private_key }}" dest="./{{ key_pair_hostsweb.key.name }}.pem" mode=0400
         register: kpair






Aqui começará o processo de provisionamento do ec2, ciando o internet gateway e adicionando  a routing table.

  - name: Preparando provisionando ec2 
         ec2_vpc_igw:             
           profile: "{{ profile }}"
           region: "{{ regiao }}"
           vpc_id: "{{ vpcl.vpc.id }}"
         register: vpcigw 

       - name: Adicionando Routing table 
         ec2_vpc_route_table: 
           vpc_id: "{{ vpcl.vpc.id }}"
           region: "{{ regiao }}"
           profile: "{{ profile }}"
           subnets:
             - "{{ subwb.subnet.id }}"
           routes: 
             - dest: 0.0.0.0/0
               gateway_id:  "{{ vpcigw.gateway_id }}"
         register: rtbl



e por fim o provisionamento do ec2 , assim adicionando alguns registros que foram criados anteriormente. Como o keypair, security group e a subnet, mas também adicionando o image ID, o qual foi retirado do próprio AWS, a image ID cujo sistema operacional é Ubuntu Server 18.04 LTS e adicionando o tipo da instancia, nesse cado  t2.micro.

- name: Provisionamento EC2
         ec2_instance: 
           name: ansible_ec2
           key_name: "{{ kpair.key.name }}"
           profile: "{{ profile }}"
           region: "{{ regiao }}"
           
           vpc_subnet_id: "{{ subwb.subnet.id }}"
           instance_type: t2.micro
           image_id: ami-085925f297f89fce1
           security_group: "{{ secgr.group_id }}"
           network: 
             assign_public_ip: true
           wait: true 
           wait_timeout: 500
           tags:
             name: webserver
         register: preec2 
