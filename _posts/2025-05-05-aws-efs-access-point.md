---
layout: post
title: AWS EFS Access Points
date: 2025-05-05 11:03:14 +0200
categories: aws efs
tags: aws efs vespa docker filesystem permissions
---
Today I struggled with file permission problems, when running the [vespa](https://vespa.ai/) docker image in AWS ECS. The reason was that by default,[EFS volumes are mounted as user `root` and other users don't have write permissions](https://docs.aws.amazon.com/efs/latest/ug/accessing-fs.html), but [vespa runs as user `vespa`](https://docs.vespa.ai/en/operations-selfhosted/docker-containers.html#start-vespa-container-with-vespa-user) (uid,gid=1000).
One solution would have been to wrap the vespa docker image using a custom Dockerfile and change the permissions of the volume mountpoint before starting vespa.
But as that would require building custom images for every new vespa release, **[EFS Access Points](https://docs.aws.amazon.com/efs/latest/ug/enforce-root-directory-access-point.html)** seemed like the correct approach.
Using these, it is possible to specify root directories inside the EFS volume and define owner uid & gid's for these.

In Terraform, the final ECS Task definition contains volumes and mountpoints something like this:

```terraform

resource "aws_efs_file_system" "vespa-sn" {
  encrypted = true
  tags = {
    Name = "ECS-EFS-DOCCHAT-VESPA-SN"
  }
}

resource "aws_efs_access_point" "vespa-sn-var" {
  file_system_id = aws_efs_file_system.vespa-sn.id
  root_directory {
    path = "/var"
    creation_info {
      owner_gid   = 1000
      owner_uid   = 1000
      permissions = 0755
    }
  }
}

resource "aws_ecs_task_definition" "vespa-sn" {

	container_definitions = jsonencode([{
		...
		mountPoints = [
			{
			containerPath = "/opt/vespa/var",
			sourceVolume  = "efs-vespa-sn-var"
			},
			{
			containerPath = "/opt/vespa/logs",
			sourceVolume  = "efs-vespa-sn-logs"
			}
		]
		...
	}])
	
	volume {
		name = "efs-vespa-sn-var"
		efs_volume_configuration {
			file_system_id     = aws_efs_file_system.vespa-sn.id
			transit_encryption = "ENABLED"
			authorization_config {
				access_point_id = aws_efs_access_point.vespa-sn-var.id
			}
		}
	}
	volume {
		name = "efs-vespa-sn-logs"
		efs_volume_configuration {
			file_system_id     = aws_efs_file_system.vespa-sn.id
			transit_encryption = "ENABLED"
			authorization_config {
				access_point_id = aws_efs_access_point.vespa-sn-logs.id
			}
		}
  }
```

Links:
- [https://docs.aws.amazon.com/efs/latest/ug/accessing-fs.html]()
- [https://docs.aws.amazon.com/efs/latest/ug/enforce-root-directory-access-point.html]()
- [https://docs.vespa.ai/en/operations-selfhosted/docker-containers.html#start-vespa-container-with-vespa-user]()
- [https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/efs_access_point]()