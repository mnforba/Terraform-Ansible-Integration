# Terraform-Ansible-Integration
Creating AWS EC2 instance using terraform
To begin with the setup we use terraform code to create the infrastructure on AWS. We create an AWS EC2 instance and an EBS volume attached to it.
#aws instance creation:
resource "aws_instance" "os1" {
  ami           = "instance-image-id"
  instance_type = "t2.micro"
  security_groups =  [ "secret-group-name" ]
   key_name = "key-name-used-to-create-instance"
  tags = {
    Name = "TerraformOS"
  }
}
#ebs volume created
resource "aws_ebs_volume" "ebs"{
  availability_zone =  aws_instance.os1.availability_zone
  size              = 1
  tags = {
    Name = "myterraebs"
  }
}
