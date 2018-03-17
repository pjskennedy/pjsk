---
layout: post
title:  "Terraform Single Use SSH Keys"
date:   2018-03-16T19:00:00-08:00
description: "A pattern for generating single use SSH keys in a terraform project"
categories:
---

Currently I've been getting into [Terraform](https://terraform.io/) for some projects. It's incredible to reliably and repeatedly spin up and down infrastructure on demand in no time. However, I was annoyed with SSH access on all of these EC2 instances, particularly with the smaller and shorter-lasting infrastructure. Is the correct solution to generate new SSH keys for every project? Or is it to pass around your public key for a certain SSH key on every iteration?

Both those solutions didn't seem very ... Terraform-y.

#### Here's a pattern to generate SSH keys for your AWS project from within Terraform itself as a resource.
Please note that this pattern applies (currently) to AWS projects. I'm sure it can easily be modified to work with other providers.

---

Specify local variables for where the private and public key are going to exist, this is useful since they will be saved in the projects directory. Then generate the key pair to be used in our project.

```
locals {
  public_key_filename  = "${path.root}/keys/id_rsa.pub"
  private_key_filename = "${path.root}/keys/id_rsa"
}

# Generate an RSA key to be used
resource "tls_private_key" "generated" {
  algorithm = "RSA"
}

# Generate the local SSH Key pair in the directory specified
resource "local_file" "public_key_openssh" {
  content  = "${tls_private_key.generated.public_key_openssh}"
  filename = "${local.public_key_filename}"
}
resource "local_file" "private_key_pem" {
  content  = "${tls_private_key.generated.private_key_pem}"
  filename = "${local.private_key_filename}"
}
```

Now that we have our keys generated locally via Terraform, we need to record it in AWS to access in new EC2 instances.

```
resource "aws_key_pair" "generated" {
  key_name   = "pjsk-sshtest-${uuid()}"
  public_key = "${tls_private_key.generated.public_key_openssh}"

  lifecycle {
    ignore_changes = ["key_name"]
  }
}
```

At this point, we should be able to provision EC2 machines and connect to them with our newly generated key.

```
resource "aws_instance" "ssh_test" {
  ami           = "${data.aws_ami.ubuntu.id}"
  instance_type = "t2.micro"

  key_name = "${aws_key_pair.generated.key_name}"

  connection {
    user        = "ubuntu"
    private_key = "${tls_private_key.generated.private_key_pem}"
    timeout = "2m"
  }
}
```

<img src="/assets/images/terraform-ssh-keys/ssh.png" style="max-width: 100%;">

Now, you're probably thinking it's annoying to remember the path for the key and IP every-time you want to connect to this machine. Luckily Terraform outputs can get us out of this.

```
output "ssh_command" {
  description = "Command to use to SSH into the instance."
  value = "ssh -i ${local.private_key_filename} ubuntu@${aws_instance.ssh_test.public_ip}"
}
```

Now when terraform finishes we see:

```
aws_instance.ssh_test: Still creating... (1m10s elapsed)
aws_instance.ssh_test: Creation complete after 1m11s (ID: i-0d6278c668086e45c)

Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

ssh_command = ssh -i /Users/peter/Documents/sshtest/keys/id_rsa ubuntu@54.187.12.254
```

While, this probably isn't a secure or scalable pattern for terraform projects, it is incredibly handy for those terraform projects you bring up for a few minutes or hours.
