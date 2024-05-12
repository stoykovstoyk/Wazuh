# Detecting Reverse Shells with Wazuh

## Wazuh Manager Configuration

### Edit the Wazuh agent configuration file for the desired group using either of the following methods:


1. **Web Interface**:

- Navigate to the Wazuh Manager's web interface.

- Go to Management -> Groups.

- Click the pencil icon to edit the desired group's configuration file.

2. **Command Line**:

In command line example here we edit the **default group** 

```bash
nano /var/ossec/etc/shared/default/agent.conf
```

### Add the following configuration for the ps-list module:

The following XML snippet defines a configuration for a custom module in the Wazuh agent, allowing you to remotely execute the ps command to list running processes on the monitored system.

```xml
<wodle name="command">
    <disabled>no</disabled>
    <tag>ps-list</tag>
    <command>ps -eo user,pid,cmd</command>
    <interval>5s</interval>
    <ignore_output>no</ignore_output>
    <run_on_start>yes</run_on_start>
    <timeout>3</timeout>
</wodle>
```

Explanation:

- `wodle name="command">`: Defines a Wazuh module named `command`. The name can be customized to reflect the purpose of the module, such as `ps-module`, `process-monitor`, or any other descriptive name.

- `<disabled>no</disabled>`: Specifies that the module is not disabled, meaning it's enabled and will be executed.

- `<tag>ps-list</tag>`: Assigns a tag `ps-list` to the module. Tags are used for categorization and identification purposes. You can customize this tag to match your naming convention or to reflect the functionality of the module.

- `<command>ps -eo user,pid,cmd</command>`: Specifies the command to be executed by the module. In this case, it's the `ps` command with options `-eo user,pid,cmd` to list the user, process ID, and command of each running process.

- `<interval>5s</interval>`: Sets the interval for executing the command to every 5 seconds. This means the `ps` command will be executed and its output collected every 5 seconds.

- `<ignore_output>no</ignore_output>`: Specifies that the output of the command should not be ignored. This allows Wazuh to process and analyze the output for security monitoring purposes.

- `<run_on_start>yes</run_on_start>`: Indicates that the command should be executed when the Wazuh agent starts. This ensures that the command is run even if the agent is restarted.

- `<timeout>3</timeout>` : Sets a timeout of 3 seconds for the command execution. If the command takes longer than 3 seconds to execute, it will be terminated to prevent resource hogging and potential system instability.

This configuration enables the Wazuh agent to periodically execute the `ps` command to retrieve information about running processes on the monitored system. The collected data can then be analyzed for security monitoring and detection purposes.


### Edit the local rules file using either of the following methods:

1. **Web Interface**:

- Access the Wazuh Manager's web interface.

- Navigate to Management -> Rules -> Custom Rules.

- Click Manage Rule Files and select local_rules.xml.

2. **Command Line**:
```bash
nano /var/ossec/etc/rules/local_rules.xml
```
### Add the following rules to detect reverse shells.

Ensure that the text inside the `<location>` tag specifies the location, which relates to the `wodle` command and follows the naming convention: the name of the `wodle` followed by an underscore `_` and then the tag.

```xml
<group name="commands,">
    <!-- Rule 1: List of Running Processes -->
    <rule id="100001" level="0">
        <!-- Location: command_ps-list -->
        <location>command_ps-list</location>
        <description>List of running processes.</description>
        <group>process_monitor,</group>
    </rule>
    
    <!-- Rule 2: Detect Reverse Shell -->
    <rule id="100002" level="10">
        <if_sid>100001</if_sid>
        <!-- Matching string: eval(base64_decode -->
        <match>eval(base64_decode</match>
        <description>Reverse shell detected.</description>
        <group>process_monitor,attacks</group>
    </rule>
</group>
```

Explanation of the rules:

#### Rule 1 (ID: 100001): List of Running Processes ####

- **Location:** This specifies where the rule should be applied. In this case, it's named `command_ps-list`, corresponding to the `wodle` command used to list running processes.

- **Description:** This rule provides a list of currently running processes.

- **Group:** The rule is assigned to the `process_monitor` group, indicating it's monitoring processes.

#### Rule 2 (ID: 100002): Detect Reverse Shell ####

- **Condition:** This rule is triggered if the condition of Rule **1** is met (indicated by `<if_sid>100001</if_sid>`), meaning it's based on the output of Rule **1**.

- **Match:** It looks for the pattern `eval(base64_decode` within the process dump.

- **Description:** When this pattern is found, it triggers an alert for a potential reverse shell attack.

- **Group:** It's assigned to both `process_monitor` and `attacks` groups, indicating it's monitoring processes and detecting attacks.

- **Relation:** Rule **2** is a child rule of Rule **1**, meaning it's dependent on the output of Rule **1** to trigger. This dependency is established through the condition specified by `<if_sid>100001</if_sid>`, which triggers Rule **2** based on the output of Rule 1.

### Restart the Wazuh manager service:

You can restart the Wazuh manager service using either of the following methods:

1. **Web Interface**:

After making changes to the configuration files through the web interface:

- Click the "Save" button to apply the changes.
- Click the "Restart" button to restart the Wazuh manager service.

2. **Command Line**:

```bash
systemctl restart wazuh-manager
```

This command will restart the Wazuh manager service from the command line interface.

## Wazuh Agent Configuration

### Append remote command configuration to the local internal options file:

```bash
echo -e "wazuh_command.remote_commands=1" >> /var/ossec/etc/local_internal_options.conf
```

### Restart the Wazuh agent service:

```bash
service wazuh-agent restart
```