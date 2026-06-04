# DevOps Tech Challenge — Node.js ECS Deployment

## Objective
This challenge involves deploying a React frontend and Express backend application to AWS using ECS with Fargate. All application infrastructure is provisioned using Terraform, and deployments are automated through a Jenkins CI/CD pipeline. The frontend is publicly accessible over the internet and connects to the backend service running as containers on ECS.

---

## Phase 1: Local Setup & Application Validation

### Step 1.1: Clone the repository
```
git clone https://github.com/TayoLusi19/devops-code-challenge1.git
```

### Step 1.2: Update and refresh packages
```
sudo apt update && sudo apt upgrade -y
```

### Step 1.3: Install Node.js
```
sudo apt install nodejs
```

### Step 1.4: Install Terraform
```
# Add HashiCorp GPG key and repo
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform -y

# Verify
terraform -v
```

### Step 1.5: Install Docker
```
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to docker group so you don't need sudo
sudo usermod -aG docker $USER
```

### Step 1.6: Log out and back in for group change to take effect

### Step 1.7: Initialize the backend
```
cd backend
npm ci
npm start
```
![Backend](/screenshots/back_end.png)

### Step 1.8: Navigate to localhost:8080 to verify the backend is running

### Step 1.9: Initialize the frontend
```
cd frontend
npm ci
npm start
```
![Frontend](/screenshots/front_end.png)

### Step 1.10: Navigate to localhost:3000 to view the frontend web page

---

## Phase 2: Containerization & Local Testing

### Step 2.1: Create Dockerfiles
```
touch ~/projects/aws_coding_challenge/backend/Dockerfile
touch ~/projects/aws_coding_challenge/frontend/Dockerfile
```

### Step 2.2: Add the following to the backend Dockerfile
```
FROM node:16
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 8080
CMD ["npm", "start"]
```

### Step 2.3: Add the following to the frontend Dockerfile
```
FROM node:16
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "build"]
```

### Step 2.4: Update frontend config for local development
```
export const API_URL = 'http://localhost:8080/'
export default API_URL
```

### Step 2.5: Update the useEffect() hook
```javascript
useEffect(() => {
    const getId = async () => {
      try {
        const resp = await fetch(API_URL)
        const data = await resp.json()
        setSuccessMessage(data.message)
      }
      catch(e) {
        setFailureMessage(e.message)
      }
    }
    getId()
  })
```

### Step 2.6: Build both Docker containers
```
# Build backend
cd ~/projects/aws_coding_challenge/backend
sudo docker build -t backend-app .

# Build frontend
cd ~/projects/aws_coding_challenge/frontend
sudo docker build -t frontend-app .
```

### Step 2.7: Run both containers
```
sudo docker run -d --name backend -p 8080:8080 backend-app
sudo docker run -d --name frontend -p 3000:3000 frontend-app
```

### Step 2.8: Update frontend config to use container name
```
export const API_URL = 'backend:8080/'
export default API_URL
```

---

## Phase 3: Infrastructure Provisioning (Terraform)

### Step 3.1: Configure AWS credentials
```
aws configure
```
![AWS Configure](/screenshots/aws_configure.png)

### Step 3.2: Create Terraform directory and files
```
cd ~/projects/aws_coding_challenge
mkdir terraform
touch main.tf outputs.tf provider.tf variables.tf vpc.tf ecr.tf ecs.tf sg.tf lb.tf fargate.tf jenkins_server.tf
```

### Step 3.3: Add provider block to provider.tf
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.0.0-beta2"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

### Step 3.4: Add VPC and networking resources to vpc.tf
```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.project_name}-vpc"
    Environment = var.environment
  }
}

# Public subnets
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = count.index == 0 ? "us-east-2a" : "us-east-2b"
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.project_name}-public-subnet-${count.index + 1}"
    Environment = var.environment
  }
}

# Private subnets
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 2)
  availability_zone = count.index == 0 ? "us-east-2a" : "us-east-2b"

  tags = {
    Name        = "${var.project_name}-private-subnet-${count.index + 1}"
    Environment = var.environment
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "${var.project_name}-igw"
    Environment = var.environment
  }
}

# NAT Gateway
resource "aws_eip" "nat" {
  domain = "vpc"
  tags = {
    Name        = "${var.project_name}-nat-eip"
    Environment = var.environment
  }
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name        = "${var.project_name}-nat-gateway"
    Environment = var.environment
  }

  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name        = "${var.project_name}-public-rt"
    Environment = var.environment
  }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = {
    Name        = "${var.project_name}-private-rt"
    Environment = var.environment
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

### Step 3.5: Add ECS cluster and service resources to ecs.tf
```hcl
resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Name        = "${var.project_name}-cluster"
    Environment = var.environment
  }
}

resource "aws_cloudwatch_log_group" "frontend" {
  name              = "/ecs/${var.project_name}-frontend"
  retention_in_days = 7

  tags = {
    Name        = "${var.project_name}-frontend-logs"
    Environment = var.environment
  }
}

resource "aws_cloudwatch_log_group" "backend" {
  name              = "/ecs/${var.project_name}-backend"
  retention_in_days = 7

  tags = {
    Name        = "${var.project_name}-backend-logs"
    Environment = var.environment
  }
}

resource "aws_ecs_task_definition" "frontend" {
  family                   = "${var.project_name}-frontend"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.frontend_cpu
  memory                   = var.frontend_memory
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    {
      name  = "frontend"
      image = var.frontend_image != "" ? var.frontend_image : "${aws_ecr_repository.frontend.repository_url}:latest"
      portMappings = [{ containerPort = var.frontend_port, protocol = "tcp" }]
      environment = [{ name = "REACT_APP_BACKEND_URL", value = "http://${aws_lb.main.dns_name}" }]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.frontend.name
          awslogs-region        = "us-east-2"
          awslogs-stream-prefix = "ecs"
        }
      }
      essential = true
    }
  ])

  tags = {
    Name        = "${var.project_name}-frontend-task"
    Environment = var.environment
  }
}

resource "aws_ecs_task_definition" "backend" {
  family                   = "${var.project_name}-backend"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.backend_cpu
  memory                   = var.backend_memory
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    {
      name  = "backend"
      image = var.backend_image != "" ? var.backend_image : "${aws_ecr_repository.backend.repository_url}:latest"
      portMappings = [{ containerPort = var.backend_port, protocol = "tcp" }]
      environment = [{ name = "CORS_ORIGIN", value = "http://${aws_lb.main.dns_name}" }]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.backend.name
          awslogs-region        = "us-east-2"
          awslogs-stream-prefix = "ecs"
        }
      }
      essential = true
    }
  ])

  tags = {
    Name        = "${var.project_name}-backend-task"
    Environment = var.environment
  }
}

resource "aws_ecs_service" "frontend" {
  name            = "${var.project_name}-frontend-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.frontend.arn
  desired_count   = var.desired_tasks
  launch_type     = "FARGATE"

  network_configuration {
    security_groups  = [aws_security_group.frontend.id]
    subnets          = aws_subnet.private[*].id
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.frontend.arn
    container_name   = "frontend"
    container_port   = var.frontend_port
  }

  depends_on = [aws_lb_listener.frontend]

  tags = {
    Name        = "${var.project_name}-frontend-service"
    Environment = var.environment
  }
}

resource "aws_ecs_service" "backend" {
  name            = "${var.project_name}-backend-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.backend.arn
  desired_count   = var.desired_tasks
  launch_type     = "FARGATE"

  network_configuration {
    security_groups  = [aws_security_group.backend.id]
    subnets          = aws_subnet.private[*].id
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.backend.arn
    container_name   = "backend"
    container_port   = var.backend_port
  }

  tags = {
    Name        = "${var.project_name}-backend-service"
    Environment = var.environment
  }
}
```

### Step 3.6: Add ECR repositories to ecr.tf
```hcl
resource "aws_ecr_repository" "frontend" {
  name                 = "${var.project_name}-frontend"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  tags = {
    Name        = "${var.project_name}-frontend-repo"
    Environment = var.environment
  }
}

resource "aws_ecr_repository" "backend" {
  name                 = "${var.project_name}-backend"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  tags = {
    Name        = "${var.project_name}-backend-repo"
    Environment = var.environment
  }
}

resource "aws_ecr_lifecycle_policy" "frontend" {
  repository = aws_ecr_repository.frontend.name

  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 5 images"
      selection = {
        tagStatus     = "tagged"
        tagPrefixList = ["v"]
        countType     = "imageCountMoreThan"
        countNumber   = 5
      }
      action = { type = "expire" }
    }]
  })
}

resource "aws_ecr_lifecycle_policy" "backend" {
  repository = aws_ecr_repository.backend.name

  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 5 images"
      selection = {
        tagStatus     = "tagged"
        tagPrefixList = ["v"]
        countType     = "imageCountMoreThan"
        countNumber   = 5
      }
      action = { type = "expire" }
    }]
  })
}
```

### Step 3.7: Add security groups and IAM roles to sg.tf
```hcl
resource "aws_security_group" "alb" {
  name        = "${var.project_name}-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-alb-sg"
    Environment = var.environment
  }
}

resource "aws_security_group" "frontend" {
  name        = "${var.project_name}-frontend-sg"
  description = "Security group for frontend ECS tasks"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "HTTP from ALB"
    from_port       = var.frontend_port
    to_port         = var.frontend_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-frontend-sg"
    Environment = var.environment
  }
}

resource "aws_security_group" "backend" {
  name        = "${var.project_name}-backend-sg"
  description = "Security group for backend ECS tasks"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "HTTP from frontend"
    from_port       = var.backend_port
    to_port         = var.backend_port
    protocol        = "tcp"
    security_groups = [aws_security_group.frontend.id]
  }

  ingress {
    description     = "HTTP from ALB"
    from_port       = var.backend_port
    to_port         = var.backend_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-backend-sg"
    Environment = var.environment
  }
}

resource "aws_security_group" "jenkins" {
  name        = "${var.project_name}-jenkins-sg"
  description = "Security group for Jenkins server"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Jenkins Web UI"
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-jenkins-sg"
    Environment = var.environment
  }
}

resource "aws_iam_role" "ecs_task_execution_role" {
  name = "${var.project_name}-ecs-task-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_role_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role" "ecs_task_role" {
  name = "${var.project_name}-ecs-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_policy" "ecs_task_logs" {
  name        = "${var.project_name}-ecs-task-logs"
  description = "Allow ECS tasks to write to CloudWatch Logs"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"]
      Resource = "*"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_logs" {
  role       = aws_iam_role.ecs_task_role.name
  policy_arn = aws_iam_policy.ecs_task_logs.arn
}
```

### Step 3.8: Add load balancer resources to lb.tf
```hcl
resource "aws_lb" "main" {
  name               = "${var.project_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = false

  tags = {
    Name        = "${var.project_name}-alb"
    Environment = var.environment
  }
}

resource "aws_lb_target_group" "frontend" {
  name        = "${var.project_name}-frontend-tg"
  port        = var.frontend_port
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }

  tags = {
    Name        = "${var.project_name}-frontend-tg"
    Environment = var.environment
  }
}

resource "aws_lb_target_group" "backend" {
  name        = "${var.project_name}-backend-tg"
  port        = var.backend_port
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }

  tags = {
    Name        = "${var.project_name}-backend-tg"
    Environment = var.environment
  }
}

resource "aws_lb_listener" "frontend" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.frontend.arn
  }
}

resource "aws_lb_listener_rule" "backend" {
  listener_arn = aws_lb_listener.frontend.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.backend.arn
  }

  condition {
    path_pattern {
      values = ["/api", "/api/*"]
    }
  }
}
```

### Step 3.9: Add auto scaling resources to fargate.tf
```hcl
resource "aws_appautoscaling_target" "frontend" {
  max_capacity       = var.max_tasks
  min_capacity       = var.min_tasks
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.frontend.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "frontend_cpu" {
  name               = "${var.project_name}-frontend-cpu-policy"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.frontend.resource_id
  scalable_dimension = aws_appautoscaling_target.frontend.scalable_dimension
  service_namespace  = aws_appautoscaling_target.frontend.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = var.cpu_threshold
  }
}

resource "aws_appautoscaling_target" "backend" {
  max_capacity       = var.max_tasks
  min_capacity       = var.min_tasks
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.backend.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "backend_cpu" {
  name               = "${var.project_name}-backend-cpu-policy"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.backend.resource_id
  scalable_dimension = aws_appautoscaling_target.backend.scalable_dimension
  service_namespace  = aws_appautoscaling_target.backend.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = var.cpu_threshold
  }
}
```

### Step 3.10: Add Jenkins EC2 instance to jenkins_server.tf
```hcl
data "aws_ami" "amazon_linux_2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "jenkins_master" {
  ami           = data.aws_ami.amazon_linux_2023.id
  instance_type = "t3.small"
  key_name      = "1PU"

  vpc_security_group_ids = [aws_security_group.jenkins.id]
  subnet_id              = aws_subnet.public[0].id

  user_data = <<-EOF
    #!/bin/bash
    dnf install wget java-21-amazon-corretto -y
    wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
    rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    dnf install jenkins -y
    systemctl enable jenkins
    systemctl start jenkins
  EOF

  root_block_device {
    volume_size = 30
    volume_type = "gp3"
    encrypted   = true
  }

  tags = {
    Name        = "Jenkins Master"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_eip" "jenkins_master" {
  instance = aws_instance.jenkins_master.id
  domain   = "vpc"

  tags = {
    Name        = "${var.project_name}-jenkins-master-eip"
    Environment = var.environment
  }
}
```

### Step 3.11: Add variables to variables.tf
```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-2"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "devops-challenge"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "frontend_port" {
  description = "Port for frontend service"
  type        = number
  default     = 3000
}

variable "backend_port" {
  description = "Port for backend service"
  type        = number
  default     = 8080
}

variable "frontend_image" {
  description = "ECR image URI for frontend"
  type        = string
  default     = ""
}

variable "backend_image" {
  description = "ECR image URI for backend"
  type        = string
  default     = ""
}

variable "frontend_cpu" {
  description = "CPU units for frontend task"
  type        = number
  default     = 512
}

variable "frontend_memory" {
  description = "Memory for frontend task (MB)"
  type        = number
  default     = 1024
}

variable "backend_cpu" {
  description = "CPU units for backend task"
  type        = number
  default     = 512
}

variable "backend_memory" {
  description = "Memory for backend task (MB)"
  type        = number
  default     = 1024
}

variable "min_tasks" {
  description = "Minimum number of tasks"
  type        = number
  default     = 1
}

variable "desired_tasks" {
  description = "Desired number of tasks"
  type        = number
  default     = 1
}

variable "max_tasks" {
  description = "Maximum number of tasks"
  type        = number
  default     = 4
}

variable "cpu_threshold" {
  description = "CPU utilization threshold for auto scaling"
  type        = number
  default     = 50
}
```

### Step 3.12: Add outputs to outputs.tf (optional)
```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "frontend_repository_url" {
  description = "URL of the frontend ECR repository"
  value       = aws_ecr_repository.frontend.repository_url
}

output "backend_repository_url" {
  description = "URL of the backend ECR repository"
  value       = aws_ecr_repository.backend.repository_url
}

output "alb_dns_name" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.main.dns_name
}

output "ecs_cluster_name" {
  description = "Name of the ECS cluster"
  value       = aws_ecs_cluster.main.name
}

output "jenkins_url" {
  description = "URL to access Jenkins"
  value       = "http://${aws_eip.jenkins_master.public_ip}:8080"
}
```

### Step 3.13: Create a key pair for the EC2 instance
```
aws ec2 create-key-pair --key-name 1PU --query 'KeyMaterial' --output text > 1PU.pem
chmod 400 1PU.pem
cp 1PU.pem ~/.ssh/
```

### Step 3.14: Add items to .gitignore
```
node_modules
.pem
terraform/.terraform/
terraform/.terraform.lock.hcl
terraform/*.pem
terraform/*.tfstate
terraform/*.tfstate.backup
```
![Git Ignore File](/screenshots/git_ignore.png)

### Step 3.15: Apply Terraform
```
cd ~/projects/aws_coding_challenge/terraform
terraform init
terraform apply
```

### Step 3.16: Validate AWS resources on the AWS Console
Check ECS cluster, ALB DNS, VPC, and related services.

![EC2](/screenshots/ec2.png)
![ECS](/screenshots/ecs.png)
![VPC](/screenshots/vpc.png)

### Step 3.17: Push Terraform code to GitHub
```
git add terraform
git commit -m "add terraform files"
git push
```

---

## Phase 4: Jenkins Setup on AWS

### Step 4.1: SSH into the Jenkins EC2 instance
```
ssh -i ~/.ssh/1PU.pem ec2-user@<Jenkins-EC2-Public-IP>
```
![SSH](/screenshots/ssh.png)

### Step 4.2: Update the system
```
sudo dnf update && sudo dnf upgrade
```

### Step 4.3: Install Docker
```
sudo dnf install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

### Step 4.4: Add Jenkins to the Docker group
```
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

> **Note:** Jenkins and Java 21 are pre-installed on boot via the `user_data` script in `jenkins_server.tf`. No manual installation is required.

### Step 4.5: Verify Jenkins is running
Navigate to:
```
http://<Jenkins-EC2-Public-IP>:8080
```
![Jenkins](/screenshots/jenkins.png)

### Step 4.6: Get the initial admin password
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Step 4.7: Install suggested plugins
Click "Install suggested plugins" when prompted.

![Plugins](/screenshots/plugins.png)

### Step 4.8: (Optional) Create an admin user
![Admin](/screenshots/admin.png)

### Step 4.9: Install additional plugins
Navigate to **Manage Jenkins → Plugins → Available Plugins** and install:
- Docker
- Amazon EC2
- Amazon Elastic Container Service (ECS) / Fargate

### Step 4.10: Install AWS CLI on the Jenkins server
```
sudo dnf install awscli -y
aws --version
```

### Step 4.11: Configure Jenkins credentials
Navigate to **Manage Jenkins → Credentials → System → Global → Add Credentials**

**GitHub PAT Token:**
1. Kind: Username with password
2. Username: your GitHub username
3. Password: your GitHub personal access token
4. ID: `github_pat`

To create a GitHub PAT:
1. Click your profile icon → Settings
2. Scroll to Developer Settings → Personal access tokens → Tokens (classic)
3. Click Generate new token (classic)
4. Select `repo` and `admin:repo_hook` checkboxes
5. Click Generate Token and copy it immediately

**AWS Credentials:**
1. Kind: AWS Credentials
2. ID: `aws_keys`
3. Access Key ID and Secret Access Key from your IAM user

To retrieve AWS credentials:
1. Go to IAM → Users → your user
2. Security Credentials tab → Create access key
3. Select CLI use case and copy the keys immediately

---

## Phase 5: CI/CD Pipeline with Jenkins

### Step 5.1: Create a Jenkinsfile in the project root
```groovy
pipeline {
    agent any

    environment {
        AWS_REGION    = 'us-east-2'
        FRONTEND_REPO = '<your-account-id>.dkr.ecr.us-east-2.amazonaws.com/devops-challenge-frontend'
        BACKEND_REPO  = '<your-account-id>.dkr.ecr.us-east-2.amazonaws.com/devops-challenge-backend'
    }

    stages {
        stage('Checkout code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker images') {
            steps {
                script {
                    sh 'docker build -t frontend:latest ./frontend'
                    sh 'docker build -t backend:latest ./backend'
                }
            }
        }

        stage('Authenticate to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws_keys']]) {
                    script {
                        sh '''
                            aws --version
                            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $FRONTEND_REPO
                            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $BACKEND_REPO
                        '''
                    }
                }
            }
        }

        stage('Tag and Push images to ECR') {
            steps {
                script {
                    sh '''
                        docker tag frontend:latest $FRONTEND_REPO:latest
                        docker tag backend:latest $BACKEND_REPO:latest
                        docker push $FRONTEND_REPO:latest
                        docker push $BACKEND_REPO:latest
                    '''
                }
            }
        }

        stage('Update ECS services') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws_keys']]) {
                    script {
                        sh '''
                            aws ecs update-service --cluster devops-challenge-cluster --service devops-challenge-frontend-service --force-new-deployment --region $AWS_REGION
                            aws ecs update-service --cluster devops-challenge-cluster --service devops-challenge-backend-service --force-new-deployment --region $AWS_REGION
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
```

### Step 5.2: Create a Jenkins Pipeline job
1. Go to Jenkins home → Create a job
2. Name the pipeline and select Pipeline → OK
3. Under Triggers, select **GitHub hook trigger for GITScm polling**
4. Under Pipeline, set Definition to **Pipeline from SCM**
5. SCM: Git
6. Repository URL: `https://github.com/<your-username>/aws_coding_challenge.git`
7. Credentials: select your GitHub PAT
8. Branch: `*/main`
9. Click Save

### Step 5.3: Create a GitHub Webhook
1. Go to your GitHub repo → Settings → Webhooks → Add webhook
2. Payload URL: `http://<your-ec2-public-ip>:8080/github-webhook/`
3. Content Type: `application/json`
4. Click Add webhook

### Step 5.4: Run the initial build
Click **Build Now** and troubleshoot any issues.

![Run Build](/screenshots/run_build.png)

---

## Phase 6: Deploy Application and Validate

### Step 6.1: Verify successful build and deployment
![Successful Build](/screenshots/success.png)

### Step 6.2: Check ECR repositories
Confirm images are updated.
![ECR](/screenshots/ecr.png)

### Step 6.3: Check ECS services
Verify tasks are running the latest images.
![ECS](/screenshots/ecs_proof.png)

### Step 6.4: Update backend config.js to use the ALB DNS
```javascript
module.exports = {
  CORS_ORIGIN: 'http://<ALB-DNS>'
};
```

### Step 6.5: Update frontend config.js to use the ALB DNS
```javascript
export const API_URL = 'http://<ALB-DNS>/api/'
export default API_URL
```

### Step 6.6: Push code to GitHub
```
git add .
git commit -m "Updated frontend and backend config.js to use ALB DNS"
git push origin main
```

### Step 6.7: Verify the Jenkins webhook triggered a new build
![Trigger Build](/screenshots/trigger_build.png)

### Step 6.8: Verify the application in the browser
Navigate to the ALB DNS in your browser and confirm the SUCCESS message is displayed.
![ALB Success](/screenshots/alb_success.png)

---

## Phase 7: Load Testing and Auto Scaling

### Step 7.1: Install siege
```
sudo dnf install gcc make wget -y
wget http://download.joedog.org/siege/siege-latest.tar.gz
tar -xzf siege-latest.tar.gz
cd siege-*/
./configure
make
sudo make install
```

### Step 7.2: Run load test
```
siege -c 250 -t 2M http://<frontend-alb-dns>
```
![Load Test](/screenshots/load_test.png)

### Step 7.3: Monitor ECS auto scaling and document results
Confirm tasks scale up and down based on CPU utilization.

![ECS Events](/screenshots/ecs_events.png)
![ECS Backend Events](/screenshots/ecs_backend_events.png)
![Results](/screenshots/results.png)

---

## Phase 8: GitOps with GitHub Actions (Bonus)

### Step 8.1: Create a gitops branch and remove the Jenkinsfile
```
git checkout -b gitops
rm Jenkinsfile
```

### Step 8.2: Create the GitHub Actions workflow directory
```
mkdir -p .github/workflows
```

### Step 8.3: Configure OIDC for GitHub Actions in AWS
1. Go to IAM → Identity providers → Add provider
2. Provider type: OpenID Connect
3. Provider URL: `https://token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. Click Add provider

### Step 8.4: Create an IAM role for GitHub Actions
1. IAM → Roles → Create role
2. Trusted entity type: Web identity
3. Identity provider: `token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. GitHub organization: your GitHub username
6. GitHub repository: `*`
7. GitHub branch: `gitops`
8. Attach policies: `AmazonEC2ContainerRegistryPowerUser` and `AmazonECS_FullAccess`
9. Name and create the role

### Step 8.5: Create .github/workflows/deploy.yml
```yaml
name: Deploy to ECS

on:
  push:
    branches:
      - gitops

env:
  AWS_REGION:    us-east-2
  ECR_FRONTEND:  <your-account-id>.dkr.ecr.us-east-2.amazonaws.com/devops-challenge-frontend
  ECR_BACKEND:   <your-account-id>.dkr.ecr.us-east-2.amazonaws.com/devops-challenge-backend
  CLUSTER_NAME:  devops-challenge-cluster
  FRONTEND_SVC:  devops-challenge-frontend-service
  BACKEND_SVC:   devops-challenge-backend-service

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      id-token: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v3
        with:
          role-to-assume: arn:aws:iam::<your-account-id>:role/<your-oidc-role-name>
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push frontend image
        run: |
          docker build -t frontend:latest ./frontend
          docker tag frontend:latest $ECR_FRONTEND:latest
          docker push $ECR_FRONTEND:latest

      - name: Build, tag, and push backend image
        run: |
          docker build -t backend:latest ./backend
          docker tag backend:latest $ECR_BACKEND:latest
          docker push $ECR_BACKEND:latest

      - name: Update ECS frontend service
        run: |
          aws ecs update-service --cluster $CLUSTER_NAME --service $FRONTEND_SVC --force-new-deployment --region $AWS_REGION

      - name: Update ECS backend service
        run: |
          aws ecs update-service --cluster $CLUSTER_NAME --service $BACKEND_SVC --force-new-deployment --region $AWS_REGION
```

### Step 8.6: Commit and push to the gitops branch
```
git add .
git commit -m "Set up GitOps workflow with GitHub Actions"
git push origin gitops
```

### Step 8.7: Verify the GitHub Actions workflow ran successfully
Check the **Actions** tab in your GitHub repo.
![GitOps](/screenshots/gitops.png)
![Actions](/screenshots/actions.png)

### Step 8.8: Verify ECS deployments
Confirm new tasks are running for both frontend and backend services.
![Frontend Verification](/screenshots/fe_verify.png)
![Backend Verification](/screenshots/be_verify.png)

### Step 8.9: Verify the application in the browser
Navigate to the ALB DNS and confirm the SUCCESS message is displayed.
![Success Verification](/screenshots/double_check.png)
