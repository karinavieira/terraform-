


#variable "vpc_id" {
#  type = string
#}

data "aws_ami" "slacko-app"  {
    most_recent = true
    owners = ["amazon"]

    filter {
        name = "name"
        values = ["amazon*"]
    }
    
    filter {
        name = "architecture"
        values = ["x86_64"]
    }
}

data "aws_subnet" "subnet_public" {
    cidr_block = "10.0.102.0/24"    
}



#variable "vpc_id" {}


#https://www.youtube.com/watch?v=9cDDZzl7zow


data "aws_vpc" "my-vpc" {
    filter {
      name ="tag:Name"
      values = ["my-vpc"]

    } 
}

#https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/vpc

#data "aws_vpc" "my-vpc" {
#  id = "${var.vpc_id}"
#}


#variable "vpc_id" {} - não precisa de variável!

#data "aws_vpc" "selected" {
#  id = var.vpc_id
#}

#resource "aws_subnet" "example" {
#  vpc_id            = data.aws_vpc.selected.id
#
#}





resource "aws_key_pair" "slacko-sshkey"{
    key_name = "slacko-app-key"
    #Colar todo conteudo do slacko.pub  entre as aspas 
    public_key = file("slacko.pub")
    

}

resource "aws_instance" "slacko-app" {
    ami = data.aws_ami.slacko-app.id
    instance_type = "t2.micro"
    subnet_id = data.aws_subnet.subnet_public.id
    associate_public_ip_address = true

    tags = {
        Name = "slacko-app"
    }

    key_name = aws_key_pair.slacko-sshkey.id
    # arquivo de bootstrap
    user_data = file("ec2.sh")
}

resource "aws_instance" "mongodb" {
    ami = data.aws_ami.slacko-app.id
    instance_type = "t2.small"
    subnet_id = data.aws_subnet.subnet_public.id

    tags = {
        Name = "mongodb"
    }

    key_name = aws_key_pair.slacko-sshkey.id
    # arquivo de bootstrap
    user_data = file("mongodb.sh")
}    

resource "aws_security_group" "allow-slacko" {
    name = "allow_ssh_http"
    description = "allow ssh and http port"
    # entrar na AWS e em VPC na segunda coluna tem a VPC ID, colar no campo abaixo https://console.aws.amazon.com/vpc/home?region=us-east-1#VpcDetails:VpcId=vpc-0291422ce6448f173
   # vpc_id = "vpc-"
    
    #teste vpc dinâmico
    vpc_id = data.aws_vpc.my-vpc.id
    
    #"${data.aws_vpc.my-vpc.id}" 

    ingress = [
        {
        description = "Allow SSH"
        from_port = 22
        to_port = 22
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        ipv6_cidr_blocks = []
        prefix_list_ids = []
        security_groups = []
        self = null

    }
    ,{
        description = "Allow HTTP"
        from_port = 80
        to_port = 80
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        ipv6_cidr_blocks = []
        prefix_list_ids = []
        security_groups = []
        self = null
    }
    ]

    egress = [
        {
        description = "Allow all"
        from_port = 0
        to_port = 0
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        ipv6_cidr_blocks = []
        prefix_list_ids = []
        security_groups = []
        self = null
        }
    ]

    tags = {
        Name = "allow_ssh_http"
    }
}

resource "aws_security_group" "allow-mongodb" {
    name = "allow_mongodb"
    description = "allow Mongodb"
    # entrar na AWS e em VPC na segunda coluna tem a VPC ID, colar no campo abaixo
    vpc_id = data.aws_vpc.my-vpc.id

    ingress = [
        {
        description = "allow mongodb"
        from_port = 27017
        to_port = 27017
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        ipv6_cidr_blocks = []
        prefix_list_ids = []
        security_groups = []
        self = null
        }
    ]

    egress = [
        {
        description = "Allow all"
        from_port = 0
        to_port = 0
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        ipv6_cidr_blocks = []
        prefix_list_ids = []
        security_groups = []
        self = null
        }
    ]
    
    tags = {
        Name = "allow_mongodb"
    }
}

resource "aws_network_interface_sg_attachment" "mongodb-sg" {
    security_group_id = aws_security_group.allow-mongodb.id
    network_interface_id = aws_instance.mongodb.primary_network_interface_id
}

resource "aws_network_interface_sg_attachment" "slacko-sg" {
    security_group_id = aws_security_group.allow-mongodb.id
    network_interface_id = aws_instance.slacko-app.primary_network_interface_id
}

resource "aws_route53_zone" "slack_zone"{
    name = "iaac0506.com.br"

    vpc {
        # entrar na AWS e em VPC na segunda coluna tem a VPC ID, colar no campo abaixo
        vpc_id = data.aws_vpc.my-vpc.id
    }
}

resource "aws_route53_record" "mongodb"{
    zone_id = aws_route53_zone.slack_zone.id
    name = "mongodb.iaac0506.com.br"
    type = "A"
    ttl = "300"
    records = [aws_instance.mongodb.private_ip]
}


output "mongodb-host" {
  value = aws_instance.mongodb.private_ip
}

output "slackapphost" {
  value = aws_instance.slacko-app.public_ip
}

# para acessar http://(ip)/docs
