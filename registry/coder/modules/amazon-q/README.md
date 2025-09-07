---
display_name: Amazon Q
description: Run Amazon Q in your workspace to access Amazon's AI coding assistant with MCP integration and task reporting.
icon: ../../../../.icons/amazon-q.svg
verified: true
tags: [agent, ai, aws, amazon-q, mcp, agentapi]
---

# Amazon Q

Run [Amazon Q](https://aws.amazon.com/q/) in your workspace to access Amazon's AI coding assistant. This module provides a complete integration with Coder workspaces, including automatic installation, MCP (Model Context Protocol) integration for task reporting, and support for custom pre/post install scripts.

```tf
module "amazon-q" {
  source   = "registry.coder.com/coder/amazon-q/coder"
  version  = "2.0.0"
  agent_id = coder_agent.example.id

  # Required: Authentication tarball (see below for generation)
  auth_tarball = <<-EOF
base64encoded-tarball
EOF
}
```

![Amazon-Q in action](../../.images/amazon-q-new.png)

## Prerequisites

- **zstd** - Required for compressing the authentication tarball
  - **Ubuntu/Debian**: `sudo apt-get install zstd`
  - **RHEL/CentOS/Fedora**: `sudo yum install zstd` or `sudo dnf install zstd`
  - **macOS**: `brew install zstd`
  - **Windows**: Download from [zstd releases](https://github.com/facebook/zstd/releases)
- **auth_tarball** - Required if running tasks or agentapi.

### Authentication Tarball (Required for tasks)

You must generate an authenticated Amazon Q tarball on another machine where you have successfully logged in:

```bash
# 1. Install Amazon Q and login on your local machine
q login

# 2. Generate the authentication tarball
cd ~/.local/share/amazon-q
tar -c . | zstd | base64 -w 0
```

Copy the output and use it as the `auth_tarball` variable.

<details>
<summary><strong>Detailed Authentication Setup</strong></summary>

**Step 1: Install Amazon Q locally**

- Download from [AWS Amazon Q Developer](https://aws.amazon.com/q/developer/)
- Follow the installation instructions for your platform

**Step 2: Authenticate**

```bash
q login
```

Complete the authentication process in your browser.

**Step 3: Generate tarball**

```bash
cd ~/.local/share/amazon-q
tar -c . | zstd | base64 -w 0 > /tmp/amazon-q-auth.txt
```

**Step 4: Use in Terraform**

```tf
variable "amazon_q_auth_tarball" {
  type      = string
  sensitive = true
  default   = "PASTE_YOUR_TARBALL_HERE"
}
```

**Important Notes:**

- Regenerate the tarball if you logout or re-authenticate
- Each user needs their own authentication tarball
- Keep the tarball secure as it contains authentication credentials

</details>

### AI Task Integration (Required for coder_ai_task)

When using the `coder_ai_task` resource in your Coder template, you must define a `coder_parameter` named **'AI Prompt'** to enable task integration:

```tf
data "coder_parameter" "ai_prompt" {
  name         = "AI Prompt"
  display_name = "AI Prompt"
  description  = "Prompt for the AI task to execute"
  type         = "string"
  mutable      = true
  default      = ""
}

resource "coder_ai_task" "example" {
  agent_id = coder_agent.example.id
  prompt   = data.coder_parameter.ai_prompt.value
}

module "amazon-q" {
  source       = "registry.coder.com/coder/amazon-q/coder"
  version      = "2.0.0"
  agent_id     = coder_agent.example.id
  auth_tarball = var.amazon_q_auth_tarball
  ai_prompt    = data.coder_parameter.ai_prompt.value
}
```

**Important Notes:**

- The parameter name must be exactly **'AI Prompt'** (case-sensitive)
- This parameter enables the AI task workflow integration
- The parameter value is passed to the Amazon Q module via the `ai_prompt` variable
- Without this parameter, `coder_ai_task` resources will not function properly

### Default System Prompt

The module includes a simple system prompt that instructs Amazon Q:

```
You are a helpful Coding assistant. Aim to autonomously investigate
and solve issues the user gives you and test your work, whenever possible.
Avoid shortcuts like mocking tests. When you get stuck, you can ask the user
but opt for autonomy.
```

### System Prompt Features:

- **Autonomous Operation:** Encourages Amazon Q to work independently and test solutions
- **Quality Focus:** Avoids shortcuts like mocking tests, promotes thorough testing
- **User Interaction:** Clear guidelines on when to ask for user input
- **Coding Focus:** Specifically designed for coding and development tasks

You can customize this behavior by providing your own system prompt via the `system_prompt` variable.

### Default Coder MCP Instructions

The module includes specific instructions for the Coder MCP server integration that are separate from the system prompt:

```
YOU MUST REPORT ALL TASKS TO CODER.
When reporting tasks you MUST follow these EXACT instructions:
- IMMEDIATELY report status after receiving ANY user message
- Be granular If you are investigating with multiple steps report each step to coder.

Task state MUST be one of the following:
- Use "state": "working" when actively processing WITHOUT needing additional user input
- Use "state": "complete" only when finished with a task
- Use "state": "failure" when you need ANY user input lack sufficient details or encounter blockers.

Task summaries MUST:
- Include specifics about what you're doing
- Include clear and actionable steps for the user
- Be less than 160 characters in length
```

You can customize these instructions by providing your own via the `coder_mcp_instructions` variable.

## Default Agent Configuration

The module includes a default agent configuration template that provides a comprehensive setup for Amazon Q integration:

```json
{
  "name": "agent",
  "description": "This is an default agent config",
  "prompt": "${system_prompt}",
  "mcpServers": {},
  "tools": [
    "fs_read",
    "fs_write",
    "execute_bash",
    "use_aws",
    "@coder",
    "knowledge"
  ],
  "toolAliases": {},
  "allowedTools": ["fs_read", "@coder"],
  "resources": [
    "file://AmazonQ.md",
    "file://README.md",
    "file://.amazonq/rules/**/*.md"
  ],
  "hooks": {},
  "toolsSettings": {},
  "useLegacyMcpJson": true
}
```

### Configuration Details:

- **Tools Available:** File operations, bash execution, AWS CLI, Coder MCP integration, and knowledge base access
- **@coder Tool:** Enables Coder MCP integration for task reporting (`coder_report_task` and related tools)
- **Allowed Tools:** By default, only `fs_read` and `@coder` are allowed (can be customized for security)
- **Resources:** Access to documentation and rule files in the workspace
- **MCP Servers:** Empty by default, can be configured via `agent_config` variable
- **System Prompt:** Dynamically populated from the `system_prompt` variable
- **Legacy MCP:** Uses legacy MCP JSON format for compatibility

You can override this configuration by providing your own JSON via the `agent_config` variable.

### Agent Name Configuration

The module automatically extracts the agent name from the `"name"` field in the `agent_config` JSON and uses it for:

- **Configuration File:** Saves the agent config as `~/.aws/amazonq/cli-agents/{agent_name}.json`
- **Default Agent:** Sets the agent as the default using `q settings chat.defaultAgent {agent_name}`
- **MCP Integration:** Associates the Coder MCP server with the specified agent name

If no custom `agent_config` is provided, the default agent name "agent" is used.

## Usage Examples

### Basic Usage

```tf
module "amazon-q" {
  source       = "registry.coder.com/coder/amazon-q/coder"
  version      = "2.0.0"
  agent_id     = coder_agent.example.id
  auth_tarball = var.amazon_q_auth_tarball
}
```

### With Custom AI Prompt

```tf
module "amazon-q" {
  source       = "registry.coder.com/coder/amazon-q/coder"
  version      = "2.0.0"
  agent_id     = coder_agent.example.id
  auth_tarball = var.amazon_q_auth_tarball
  ai_prompt    = "Help me set up a Python FastAPI project with proper testing structure"
}
```

### With Custom Pre/Post Install Scripts

```tf
module "amazon-q" {
  source       = "registry.coder.com/coder/amazon-q/coder"
  version      = "2.0.0"
  agent_id     = coder_agent.example.id
  auth_tarball = var.amazon_q_auth_tarball

  pre_install_script = <<-EOT
    #!/bin/bash
    echo "Setting up custom environment..."
    # Install additional dependencies
    sudo apt-get update && sudo apt-get install -y zstd
  EOT

  post_install_script = <<-EOT
    #!/bin/bash
    echo "Configuring Amazon Q settings..."
    # Custom configuration commands
    q settings chat.model claude-3-sonnet
  EOT
}
```

### Specific Version Installation

```tf
module "amazon-q" {
  source           = "registry.coder.com/coder/amazon-q/coder"
  version          = "2.0.0"
  agent_id         = coder_agent.example.id
  auth_tarball     = var.amazon_q_auth_tarball
  amazon_q_version = "1.14.0" # Specific version
  install_amazon_q = true
}
```

### Custom Agent Configuration

```tf
module "amazon-q" {
  source       = "registry.coder.com/coder/amazon-q/coder"
  version      = "2.0.0"
  agent_id     = coder_agent.example.id
  auth_tarball = var.amazon_q_auth_tarball

  agent_config = <<-EOT
    {
      "name": "custom-agent",
      "description": "Custom Amazon Q agent for my workspace",
      "prompt": "You are a specialized DevOps assistant...",
      "tools": ["fs_read", "fs_write", "execute_bash", "use_aws"]
    }
  EOT
}
```

### With Task Reporting and Custom Working Directory

```tf
module "amazon-q" {
  source       = "registry.coder.com/coder/amazon-q/coder"
  version      = "2.0.0"
  agent_id     = coder_agent.example.id
  auth_tarball = var.amazon_q_auth_tarball

  # Task reporting configuration
  report_tasks = true
  workdir      = "/workspace/project"

  # Enable CLI app alongside web app
  cli_app              = true
  web_app_display_name = "Amazon Q Assistant"
  cli_app_display_name = "Q CLI"
}
```

### With Custom AgentAPI Configuration

```tf
module "amazon-q" {
  source       = "registry.coder.com/coder/amazon-q/coder"
  version      = "2.0.0"
  agent_id     = coder_agent.example.id
  auth_tarball = var.amazon_q_auth_tarball

  # AgentAPI configuration for environments without wildcard access
  agentapi_chat_based_path = true
  agentapi_version         = "v0.6.1"
}
```

### UI Customization

```tf
module "amazon-q" {
  source       = "registry.coder.com/coder/amazon-q/coder"
  version      = "2.0.0"
  agent_id     = coder_agent.example.id
  auth_tarball = var.amazon_q_auth_tarball

  # UI configuration
  order = 1
  group = "AI Tools"
  icon  = "/icon/custom-amazon-q.svg"
}
```

### Air-Gapped Installation

For environments without direct internet access, you can host Amazon Q installation files internally and configure the module to use your internal repository:

```tf
module "amazon-q" {
  source       = "registry.coder.com/coder/amazon-q/coder"
  version      = "2.0.0"
  agent_id     = coder_agent.example.id
  auth_tarball = var.amazon_q_auth_tarball

  # Point to internal artifact repository
  q_install_url = "https://artifacts.internal.corp/amazon-q-releases"

  # Use specific version available in your repository
  amazon_q_version = "1.14.1"
}
```

**Prerequisites for Air-Gapped Setup:**

1. Download Amazon Q installation files from AWS and host them internally
2. Maintain the same directory structure: `{base_url}/{version}/q-{arch}-linux.zip`
3. Ensure both architectures are available:
   - `q-x86_64-linux.zip` for Intel/AMD systems
   - `q-aarch64-linux.zip` for ARM systems
4. Configure network access from Coder workspaces to your internal repository

## Troubleshooting

### Common Issues

**Amazon Q not found after installation:**

```bash
# Check if Amazon Q is in PATH
which q
# If not found, add to PATH
export PATH="$PATH:$HOME/.local/bin"
```

**Authentication issues:**

- Regenerate the auth tarball on your local machine
- Ensure the tarball is properly base64 encoded
- Check that the original authentication is still valid

**MCP integration not working:**

- Verify that AgentAPI is installed (`install_agentapi = true`)
- Check that the Coder agent is properly configured
- Review the system prompt configuration

### Debug Mode

Enable verbose logging by setting environment variables:

```bash
export DEBUG=1
export VERBOSE=1
```

## Security Considerations

- **Authentication Tarball**: Contains sensitive authentication data - mark as sensitive in Terraform
- **Tool Trust**: By default, all tools are trusted - review for security requirements
- **Pre/Post Scripts**: Custom scripts run with user permissions - validate content
- **Network Access**: Amazon Q requires internet access for AI model communication
