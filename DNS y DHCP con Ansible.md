# DNS y DHCP con Ansible
## Tecnologias y plataformas
- Ansible
- Terraform
- Docker
- AWS (S3, DyanmoDB, EC2 y VPC)
## Banco de prueba
Creación de banco de prueba de **AWX** en **AWS** desplegado con **Terraform** en contenedores **Docker**. Con Terraform se depliega dos instancias (***t2.medium*** y ***t2.micro***) con sus respectivos ***Security Groups*** y haciendo uso de una clave SSH en común. Previamente, se creara un almacenamiento ***S3*** y una tabla en ***DynamoDB*** con las que se permitira guardar el estado de los despliegues de ***Terraform*** usando el modulo ***backend***.

### backend.tf
```json
provider "aws" {
  region     = "eu-west-1"
  profile = "default"

}

resource "aws_s3_bucket" "states-awx" {
  bucket   = "states-awx"

  versioning {
    enabled = true
  }

  lifecycle {
    prevent_destroy = true
  }

  tags {
    Name = "states-awx"
  }
}

resource "aws_dynamodb_table" "states-awx" {
  name           = "states-awx"
  hash_key       = "LockID"
  read_capacity  = 20
  write_capacity = 20

  attribute {
    name = "LockID"
    type = "S"
  }

  tags {
    Name = "states-awx"
  }
}
```
### main.tf
```json
terraform {
  required_version = ">=0.11.13"

  backend "s3" {
    bucket         = "states-awx"
    region         = "eu-west-1"
    key            = "states-tfstate"
    dynamodb_table = "states-awx"
    profile        = "default"
  }
}

provider "aws" {
  region  = "${var.region}"
  profile = "default"
}

resource "aws_key_pair" "awx" {
  public_key = "${file("${var.file-ssh-pubkey}")}"
  key_name   = "awx"
}

resource "aws_security_group" "sgdns" {
  name = "sgdns"
  egress {
    from_port = 0
    protocol = "-1"
    to_port = 0
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port = 22
    protocol = "tcp"
    to_port = 22
    cidr_blocks = ["${var.ip-emergya}"]
  }
  ingress {
    from_port = 0
    protocol = "-1"
    to_port = 0
    cidr_blocks = ["172.31.16.10/32"]
  }
  tags {
    Name = "sgdns"
  }
}

resource "aws_instance" "awx-dns-server" {
  ami = "${var.ami}"
  instance_type = "t2.micro"
  key_name = "${aws_key_pair.awx.key_name}"
  security_groups = ["${aws_security_group.sgdns.name}"]
  availability_zone = "eu-west-1a"
  private_ip = "172.31.16.11"

  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt upgrade -y",
      "sudo apt install apt-transport-https ca-certificates curl gnupg-agent software-properties-common -y",
      "curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -",
      "sudo add-apt-repository 'deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable' -y",
      "sudo apt update",
      "sudo apt install docker-ce python3 python -y",
      "sudo systemctl stop systemd-resolved",
      "sudo systemctl disable systemd-resolved",
      "sudo systemctl mask systemd-resolved",
      "sudo rm /etc/resolv.conf",
      "echo 'nameserver 127.0.0.1' | sudo tee /etc/resolv.conf",
      "echo 'nameserver 8.8.8.8' | sudo tee -a /etc/resolv.conf",
      "sudo adduser ubuntu docker",
      "sudo rm -rf /root/.ssh",
      "sudo cp -r /home/ubuntu/.ssh /root"
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = "${file("${var.file-ssh-key}")}"
    }
  }
  tags {
    Name = "awx-dns-server"
  }
}

resource "aws_security_group" "sgawx" {
  name = "sgawx"

  ingress {
    from_port   = 22
    protocol    = "tcp"
    to_port     = 22
    cidr_blocks = ["${var.ip-emergya}","172.31.16.11/32"]
  }

  ingress {
    from_port   = 443
    protocol    = "tcp"
    to_port     = 443
    cidr_blocks = ["${var.ip-emergya}","172.31.16.11/32"]
  }
  egress {
    from_port   = 0
    protocol    = "-1"
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags {
    Name = "sgawx"
  }
}

resource "aws_instance" "awx" {
  ami             = "${var.ami}"
  instance_type   = "${var.instance-type}"
  security_groups = ["${aws_security_group.sgawx.name}"]
  key_name        = "${aws_key_pair.awx.key_name}"
  availability_zone = "eu-west-1a"
  private_ip = "172.31.16.10"

  root_block_device {
    volume_size = 20
  }

  provisioner "file" {
    destination = "/home/ubuntu/docker-compose.yml"
    source      = "docker-compose.yml"

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = "${file("${var.file-ssh-key}")}"
    }
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt upgrade -y",
      "sudo apt install apt-transport-https ca-certificates curl gnupg-agent software-properties-common -y",
      "curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -",
      "sudo add-apt-repository 'deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable' -y",
      "sudo apt update",
      "sudo apt install docker-ce python3 python -y",
      "sudo curl -L https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose",
      "sudo chmod +x /usr/local/bin/docker-compose",
      "sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose",
      "sudo adduser ubuntu docker"
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = "${file("${var.file-ssh-key}")}"
    }
  }

  provisioner "remote-exec" {
    inline = ["docker-compose -f /home/ubuntu/docker-compose.yml up -d",
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = "${file("${var.file-ssh-key}")}"
    }
  }

  tags {
    Name = "awx-controller"
  }
}

output "instance" {
  value = "${aws_instance.awx.public_dns}"
}
output "instance-priv" {
  value = "${aws_instance.awx.private_ip}"
}
output "instance-dns" {
  value = "${aws_instance.awx-dns-server.public_dns}"
}
output "instance-dns-priv" {
  value = "${aws_instance.awx-dns-server.private_ip}"
}
```
### variables.tf
```json
variable "region" {
  type    = "string"
  default = "eu-west-1"
}

variable "ami" {
  type    = "string"
  default = "ami-0727f3c2d4b0226d5"
}

variable "instance-type" {
  type    = "string"
  default = "t2.medium"
}

variable "file-ssh-pubkey" {
  type    = "string"
  default = "~/.ssh/awx.pub"
}

variable "file-ssh-key" {
  type    = "string"
  default = "~/.ssh/awx"
}

variable "ip-emergya" {
  type    = "list"
  default = ["89.7.187.142/32", "79.146.59.158/32", "79.146.65.242/32","89.140.125.66/32"]
}
```
### emergya-variables.tf
```json
variable "emergya_docker_registry_uri" {
  type    = "string"
  default = "docker-registry.emergya.com:443"
}

variable "emergya_docker_registry_user" {
  type    = "string"
  default = "XXXXXXXXXXXXXXXX"
}

variable "emergya_docker_registry_pass" {
  type    = "string"
  default = "XXXXXXXXXXXXXXXX"
}

variable "profile" {
  type    = "string"
  default = "default"
}
```
### docker-compose.yml
```yaml
version: '2'
services:

  web:
    image: ansuan/awx-web-ssl:latest
    depends_on:
      - rabbitmq
      - memcached
      - postgres
    ports:
      - "443:8052"
    hostname: awxweb
    user: root
    restart: unless-stopped
    environment:
      http_proxy:
      https_proxy:
      no_proxy:
      SECRET_KEY: awxsecret
      DATABASE_NAME: awx
      DATABASE_USER: awx
      DATABASE_PASSWORD: awxpass
      DATABASE_PORT: 5432
      DATABASE_HOST: postgres
      RABBITMQ_USER: guest
      RABBITMQ_PASSWORD: guest
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PORT: 5672
      RABBITMQ_VHOST: awx
      MEMCACHED_HOST: memcached
      MEMCACHED_PORT: 11211
      AWX_ADMIN_USER: admin
      AWX_ADMIN_PASSWORD: emergya

  task:
    image: ansible/awx_task:3.0.1
    depends_on:
      - rabbitmq
      - memcached
      - web
      - postgres
    hostname: awx
    user: root
    restart: unless-stopped
    environment:
      http_proxy:
      https_proxy:
      no_proxy:
      SECRET_KEY: awxsecret
      DATABASE_NAME: awx
      DATABASE_USER: awx
      DATABASE_PASSWORD: awxpass
      DATABASE_HOST: postgres
      DATABASE_PORT: 5432
      RABBITMQ_USER: guest
      RABBITMQ_PASSWORD: guest
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PORT: 5672
      RABBITMQ_VHOST: awx
      MEMCACHED_HOST: memcached
      MEMCACHED_PORT: 11211
      AWX_ADMIN_USER: admin
      AWX_ADMIN_PASSWORD: emergya

  rabbitmq:
    image: ansible/awx_rabbitmq:3.7.4
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_VHOST: awx
      RABBITMQ_ERLANG_COOKIE: cookiemonster

  memcached:
    image: memcached:alpine
    restart: unless-stopped

  postgres:
    image: postgres:9.6
    restart: unless-stopped
    volumes:
      - /opt/awx/var/lib/postgresql/data:/var/lib/postgresql/data:Z
    environment:
      POSTGRES_USER: awx
      POSTGRES_PASSWORD: awxpass
      POSTGRES_DB: awx
      PGDATA: /var/lib/postgresql/data/pgdata
```