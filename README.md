# jenkins Cookbook

[![Cookbook Version](https://img.shields.io/cookbook/v/jenkins.svg)](https://supermarket.chef.io/cookbooks/jenkins)
[![CI State](https://github.com/sous-chefs/jenkins/workflows/ci/badge.svg)](https://github.com/sous-chefs/jenkins/actions?query=workflow%3Aci)
[![OpenCollective](https://opencollective.com/sous-chefs/backers/badge.svg)](#backers)
[![OpenCollective](https://opencollective.com/sous-chefs/sponsors/badge.svg)](#sponsors)
[![License](https://img.shields.io/badge/License-Apache%202.0-green.svg)](https://opensource.org/licenses/Apache-2.0)

Installs and configures Jenkins CI master & node slaves. Resource providers to support automation via jenkins-cli, including job create/update.

## Maintainers

This cookbook is maintained by the Sous Chefs. The Sous Chefs are a community of Chef cookbook maintainers working together to maintain important cookbooks. If you’d like to know more please visit [sous-chefs.org](https://sous-chefs.org/) or come chat with us on the Chef Community Slack in [#sous-chefs](https://chefcommunity.slack.com/messages/C2V7B88SF).

## Requirements

### Platforms

- Debian 9+
- Ubuntu 18.04+
- RHEL/CentOS 7+

### Chef

- Chef 13.0+

#### Java cookbook

This cookbook does not install, manage, or manipulate a JDK, as that is outside of the scope of Jenkins. The `package` installation method will automatically pull in a valid Java if one does not exist on Debian. RHEL jenkins packages do not depend on java as there are far too many options for a package to do the right thing. We recommend including the java cookbook on your system which allows for either openJDK or Oracle JDK installations.

## Attributes

In order to keep the README manageable and in sync with the attributes, this cookbook documents attributes inline. The usage instructions and default values for attributes can be found in the individual attribute files.

## Examples

Documentation and examples are provided inline using YARD. The tests and fixture cookbooks in `tests` and `tests/fixtures` are intended to be a further source of examples.

## Recipes

### master

The master recipe will create the required directory structure and install jenkins. There are two installation methods, controlled by the `node['jenkins']['master']['install_method']` attribute:

- `package` - Install Jenkins from the official jenkins-ci.org packages
- `war` - Download the latest version of the WAR file and configure a systemd service

## Resources

### jenkins_command

This resource executes arbitrary commands against the [Jenkins CLI](https://wiki.jenkins-ci.org/display/JENKINS/Jenkins+CLI)

#### Actions

- :execute

#### Examples

To perform a restart

```ruby
jenkins_command 'safe-restart'
```

To reload the configuration from disk:

```ruby
jenkins_command 'reload-configuration'
```

To prevent Jenkins from starting any new builds (in preparation for a shutdown):

```ruby
jenkins_command 'quiet-down'
```

**NOTE** You must add your own `not_if`/`only_if` guards to the `jenkins_command` to prevent duplicate commands from executing. Just like Chef's core `execute` resource, the `jenkins_command` resource has no way of being idempotent.

### jenkins_script

This resource executes arbitrary Java or Groovy commands against the Jenkins master. By the nature of this command, it is **not** idempotent.

#### Examples

A simple inline Groovy script

```ruby
jenkins_script 'println("This is Groovy code!")'
```

More complex inline Groovy

```ruby
jenkins_script 'add_authentication' do
  command <<-EOH.gsub(/^ {4}/, '')
    import jenkins.model.*
    import hudson.security.*
    import org.jenkinsci.plugins.*

    def instance = Jenkins.getInstance()

    def githubRealm = new GithubSecurityRealm(
      'https://github.com',
      'https://api.github.com',
      'API_KEY',
      'API_SECRET'
    )
    instance.setSecurityRealm(githubRealm)

    def strategy = new FullControlOnceLoggedInAuthorizationStrategy()
    instance.setAuthorizationStrategy(strategy)

    instance.save()
  EOH
end
```

Executing Groovy code on disk

```ruby
template ::File.join(Chef::Config[:file_cache_path], 'create_jenkins_user' + '.groovy') do
  source "create_jenkins_user.groovy.erb"
  mode '0644'
  owner 'jenkins'
  group 'jenkins'
  variables(
    users: users
  )
  notifies :execute, "jenkins_script[create_jenkins_user]", :immediately
end

jenkins_script 'create_jenkins_user' do
  groovy_path ::File.join(Chef::Config[:file_cache_path], 'create_jenkins_user' + '.groovy')
end
```

### jenkins_credentials

#### NOTES

- Install version 1.6 or higher of the [credentials plugin](https://wiki.jenkins-ci.org/display/JENKINS/Credentials+Plugin) to use the Jenkins credentials resource.

- In version `4.0.0` of this cookbook this resource was changed so that credentials are referenced by their ID instead of by their name. If you are upgrading your nodes from an earlier version of this cookbook ( <= 3.1.1 ), use the credentials resource and do not have explicit IDs assigned to credentials, you will need to go into the Jenkins UI, find the auto-generated UUIDs for your credentials, and add them to your cookbook resources.

#### Actions

- :create
- :delete

Both actions operate on the credential resources idempotently. It also supports why-run mode.

`jenkins_credentials` is a base resource that is not used directly. Instead there are resources for each specific type of credentials supported.

### Properties

Use of the credential resource requires a unique `id` property. The resource uses this ID to find the credential for future modifications, and it is an immutable resource once the resource is created within Jenkins. This ID is also how you reference the credentials in other Groovy scripts (i.e. Pipeline code).

The `username` property (also the name property) corresponds to the username of the credentials on the target node.

You may also specify a `description` which is useful in credential identification.

#### jenkins_password_credentials

Basic username + password credentials.

##### Examples

```ruby
# Create password credentials
jenkins_password_credentials 'wcoyote' do
  id          'wcoyote-password'
  description 'Wile E Coyote'
  password    'beepbeep'
end
```

```ruby
# Delete password credentials
jenkins_password_credentials 'wcoyote' do
  id     'wcoyote-password'
  action :delete
end
```

#### jenkins_private_key_credentials

Credentials that use a username + private key (optionally protected with a passphrase).

##### Examples

```ruby
# Create private key credentials
jenkins_private_key_credentials 'wcoyote' do
  id          'wcoyote-key'
  description 'Wile E Coyote'
  private_key "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQ..."
end

# Private keys with a passphrase will also work
jenkins_private_key_credentials 'wcoyote' do
  id          'wcoyote-key'
  description 'Eile E Coyote'
  private_key "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQ..."
  passphrase  'beepbeep'
end
```

```ruby
# Delete private key
jenkins_private_key_credentials 'wcoyote' do
  id     'wcoyote-key'
  action :delete
end
```

#### jenkins_secret_text_credentials

Generic secret text. Requires the the `credentials-binding` plugin.

##### Examples

```ruby
# Create secret text credentials
jenkins_secret_text_credentials 'wcoyote' do
  id          'wcoyote-secret'
  description 'Wile E Coyote Secret'
  secret      'Some secret text'
end
```

```ruby
# Delete secret text credentials
jenkins_secret_text_credentials 'wcoyote' do
  id     'wcoyote-secret'
  action :delete
end
```

### jenkins_file_credentials

Generic file credentials.

```ruby
# Create file credentials
jenkins_file_credentials 'wcoyote' do
  id          'wcoyote-file'
  description 'Wile E Coyote File'
  filename    'file.txt'
  data        'my file content'
end
```

```ruby
# Delete file credentials
jenkins_file_credentials 'wcoyote' do
  id     'wcoyote-file'
  action :delete
end
```

#### Scopes

Credentials in Jenkins can be created with 2 different "scopes" which determines where the credentials can be used:

- **GLOBAL** - This credential is available to the object on which the credential is associated and all objects that are children of that object. Typically you would use global-scoped credentials for things that are needed by jobs.
- **SYSTEM** - This credential is only available to the object on which the credential is associated. Typically you would use system-scoped credentials for things like email auth, slave connection, etc, i.e. where the Jenkins instance itself is using the credential. Unlike the global scope, this significantly restricts where the credential can be used, thereby providing a higher degree of confidentiality to the credential.

The credentials created with the `jenkins_credentials` resources are assigned a `GLOBAL` scope.

### jenkins_job

This resource manages Jenkins jobs

#### Actions

- :create
- :delete
- :disable
- :enable
- :build

The resource is fully idempotent and convergent. It also supports why-run mode.

The `:create` action requires a Jenkins job `config.xml`. This config file must exist on the target node and contain a valid Jenkins job configuration file. Because the Jenkins CLI actually reads and generates its own copy of this file, **do NOT** write this configuration inside of the Jenkins job. We recommend putting them in Chef's file cache path:

```ruby
xml = File.join(Chef::Config[:file_cache_path], 'bacon-config.xml')

# You could also use a `cookbook_file` or pure `file` resource to generate
# content at this path.
template xml do
  source 'custom-config.xml.erb'
end

# Create a jenkins job (default action is `:create`)
jenkins_job 'bacon' do
  config xml
end
```

```ruby
jenkins_job 'bacon' do
  action :delete
end
```

You can disable a Jenkins job by specifying the `:disable` option. This will disable an existing job, if and only if that job exists and is enabled. If the job does not exist, an exception is raised.

```ruby
jenkins_job 'bacon' do
  action :disable
end
```

You can enable a Jenkins job by specifying the `:enable` option. This will enable an existing job, if and only if that job exists and is disabled. If the job does not exist, an exception is raised.

```ruby
jenkins_job 'bacon' do
  action :enable
end
```

You can execute a Jenkins job by specifying the `:build` option. This will run the job, if and only if that job exists and is enabled. If the job does not exist, an exception is raised.

```ruby
jenkins_job 'my-parameterized-job' do
  parameters(
    'STRING_PARAM' => 'meeseeks',
    'BOOLEAN_PARAM' => true,
  )
  # if true will live stream the console output of the executing job  (default is true)
  stream_job_output true
  # if true will block the Chef client run until the build is completed or aborted (defaults to true)
  wait_for_completion true
  action :build
end
```

### jenkins_view

This resource manages Jenkins view

#### Actions

- :create
- :delete

The resource is fully idempotent and convergent as long as you're not using free hand code. It also supports whyrun mode.

The `:create` action requires an array of jobs:

```ruby
jenkins_view 'ham' do
  jobs [ "pig", "giraffe" ]
end
```

The `:delete` action deleted a configured view:

```ruby
jenkins_view 'ham' do
  action :delete
end
```

It is possible to pass a snippet of groovy code in order to create more sophisticated views, the idea is to override the `create_view` and `configure_view` groovy closures.

```ruby
code = <<-GROOVY
create_view = { name ->
  // Return a new view
  return new BuildPipelineView(...)
}

configure_view = { view ->
  // Configure view
  view.setCssUrl("")
}
GROOVY

jenkins_view 'pipline_view' do
  code    code
  action :create
end
```

Please note that if you pass `code`, it will always run the `:create` action as the provider cannot determine when a change has to be made and when not.

### jenkins_proxy

This resource manages Jenkins HTTP proxy information

#### Actions

- :config
- :remove

This uses the Jenkins groovy API to configure the HTTP proxy information, that is provided on the _Advanced_ tab of the _Plugin Manager_.

The `:config` action idempotently configure the Jenkins HTTP proxy information on the current node. The proxy attribute corresponds to the proxy server name and port number that have to use on the target node. You may also specify a list of no proxy host names with the noproxy attribute. The default is _localhost_ and _127.0.0.1_.

```ruby
# Basic proxy configuration
jenkins_proxy '1.2.3.4:5678'

# Expanded proxy configuration
jenkins_proxy '5.6.7.8:9012' do
  noproxy ['localhost', '127.0.0.1', 'nohost', '*.nodomain']
end
```

The `:remove` action removes the Jenkins HTTP proxy information from the system.

```ruby
jenkins_proxy '1.2.3.4:5678' do
  action :remove
end
```

### jenkins_plugin

This resource manages Jenkins plugins.

#### Actions

- :install
- :uninstall
- :enable
- :disable

This uses the Jenkins CLI to install plugins. By default, it does a cold deploy, meaning the plugin is installed while Jenkins is still running. Some plugins may require you restart the Jenkins instance for their changed to take affect.

- **This resource does not install plugin dependencies from a a given hpi/jpi URL or a specific version - you must specify all plugin dependencies or Jenkins may not startup correctly!**

The `:install` action idempotently installs a Jenkins plugin on the current node. The name attribute corresponds to the name of the plugin on the Jenkins Update Center. You can also specify a particular version of the plugin to install. Finally, you can specify a full source URL or local path (on the node) to a plugin.

```ruby
# Install the latest version of the greenballs plugin and all dependencies
jenkins_plugin 'greenballs'

# Install version 1.3 of the greenballs plugin and no dependencies
jenkins_plugin 'greenballs' do
  version '1.3'
end

# Install a plugin from a given hpi (or jpi) and no dependencies
jenkins_plugin 'greenballs' do
  source 'http://updates.jenkins-ci.org/download/plugins/greenballs/1.10/greenballs.hpi'
end
```

Depending on the plugin, you may need to restart the Jenkins instance for the plugin to take affect:

```ruby
jenkins_plugin 'a_complicated_plugin' do
  notifies :restart, 'service[jenkins]', :immediately
end
```

For advanced users, this resource exposes an `options` attribute that will be passed to the installation command. For more information on the possible values of these options, please consult the documentation for your Jenkins installation.

```ruby
jenkins_plugin 'a_really_complicated_plugin' do
  options '-deploy -cold'
end
```

The `:uninstall` action removes (uninstalls) a Jenkins plugin idempotently on the current node.

```ruby
jenkins_plugin 'greenballs' do
  action :uninstall
end
```

The `:enable` action enables a plugin. If the plugin is not installed, an exception is raised. If the plugin is already enabled, no action is taken.

```ruby
jenkins_plugin 'greenballs' do
  action :enable
end
```

The `:disable` action disables a plugin. If the plugin is not installed, an exception is raised. If the plugin is already disabled, no action is taken.

```ruby
jenkins_plugin 'greenballs' do
  action :disable
end
```

**NOTE** You may need to restart Jenkins after changing a plugin. Because this varies on a case-by-case basis (and because everyone chooses to manage their Jenkins infrastructure differently) this resource does **NOT** restart Jenkins for you.

### jenkins_slave

**NOTE** The use of the Jenkins user resource requires the Jenkins SSH credentials plugin. This plugin is not shipped by default in jenkins 2.x.

This resource manages Jenkins slaves, supporting the following actions:

```
:create, :delete, :connect, :disconnect, :online, :offline
```

The following slave launch methods are supported:

- **JNLP/Java Web Start** - Starts a slave by launching an agent program through JNLP. The launch in this case is initiated by the slave, thus slaves need not be IP reachable from the master (e.g. behind the firewall). This launch method is supported on *nix and Windows platforms.
- **SSH** - Jenkins has a built-in SSH client implementation that it can use to talk to remote `sshd` daemon and start a slave agent. This is the most convenient and preferred method for Unix slaves, which normally has `sshd` out-of-the-box.

The `jenkins_slave` resource is actually the base resource for several resources that map directly back to a launch method:

- `jenkins_jnlp_slave` - As JNLP Slave connections are slave initiated, this resource should be part of a **slave**'s run list.
- `jenkins_ssh_slave` - As SSH Slave connections are master initiated, this resource should be part of a **master**'s run list.

The `:create` action idempotently creates a Jenkins slave on the master. The name attribute corresponds to the name of the slave (which is also used to uniquely identify the slave).

```ruby
# Create a basic JNLP slave
jenkins_jnlp_slave 'builder' do
  description 'A generic slave builder'
  remote_fs   '/home/jenkins'
  labels      ['builder', 'linux']
end

# Create a slave launched via SSH
jenkins_ssh_slave 'executor' do
  description 'Run test suites'
  remote_fs   '/share/executor'
  labels      ['executor', 'freebsd', 'jail']

  # SSH specific attributes
  host        '172.11.12.53' # or 'slave.example.org'
  user        'jenkins'
  credentials 'wcoyote'
  launch_timeout   30
  ssh_retries      5
  ssh_wait_retries 60
end

# A slave's executors, usage mode and availability can also be configured
jenkins_jnlp_slave 'smoke' do
  description     'Runs a series of high-level smoke tests'
  remote_fs       '/home/jenkins'
  executors       5
  usage_mode      'exclusive'
  availability    'demand'
  in_demand_delay 1
  idle_delay      3
  labels          ['runner', 'fast']

  # List of groups to run the slave service under
  service_groups  ['jenkins', 'docker']
end

# Create a slave with a full environment
jenkins_jnlp_slave 'integration' do
  description 'Runs the high-level integration suite'
  remote_fs   '/home/jenkins'
  labels      ['integration', 'rails', 'ruby']
  environment(
    RAILS_ENV: 'test',
    GCC:       '1_000_000_000'
  )
end

# Windows JNLP slave
jenkins_windows_slave 'mywinslave' do
  remote_fs 'C:/jenkins'
  user       '.\Administrator'
  password   'MyPassword'
end
```

The `:delete` action idempotently removes a slave from the cluster. Any services used to manage the underlying slave process will also be disabled.

```ruby
jenkins_jnlp_slave 'builder' do
  action :delete
end

jenkins_ssh_slave 'executor' do
  action :delete
end
```

The `:connect` action idempotently forces the master to reconnect to the specified slave. You can use the base `jenkins_slave` resource or any of its children to perform the connection.

```ruby
jenkins_slave 'builder' do
  action :connect
end

jenkins_ssh_slave 'executor' do
  action :connect
end
```

The `:disconnect` action idempotently forces the master to disconnect the specified slave. You can use the base `jenkins_slave` resource or any of its children to perform the connection.

```ruby
jenkins_slave 'builder' do
  action :disconnect
end

jenkins_ssh_slave 'executor' do
  action :disconnect
end
```

The `:online` action idempotently brings a slave back online. You can use the base `jenkins_slave` resource or any of its children to bring the slave online.

```ruby
jenkins_slave 'builder' do
  action :online
end

jenkins_ssh_slave 'executor' do
  action :online
end
```

The `:offline` action idempotently takes a slave temporarily offline. An optional reason for going offline can be provided with the `offline_reason` attribute. You can use the base `jenkins_slave` resource or any of its children to take a slave offline.

```ruby
jenkins_slave 'builder' do
  action :offline
end

jenkins_ssh_slave 'executor' do
  offline_reason 'ran out of energon'
  action :offline
end
```

**NOTE** It's worth noting the somewhat confusing differences between _disconnecting_ and _off-lining_ a slave:

- **Disconnect** - Instantly closes the channel of communication between the master and slave. Currently executing jobs will be terminated immediately. If a slave is configured with an availability of `always` the master will attempt to reconnect to the slave.
- **Offline** - Keeps the channel of communication between the master and slave open. Currently executing jobs will be allowed to finish, but no new jobs will be scheduled on the slave.

### jenkins_user

**NOTE** The use of the Jenkins user resource requires the Jenkins mailer plugin. This plugin is not shipped by default in jenkins 2.x.

This resource manages Jenkins users, supporting the following actions:

```
:create, :delete
```

This uses the Jenkins groovy API to create users.

The `:create` action idempotently creates a Jenkins user on the current node. The id attribute corresponds to the username of the id of the user on the target node. You may also specify a name, email, and list of SSH keys.

```ruby
# Create a Jenkins user
jenkins_user 'grumpy'

# Create a Jenkins user with specific attributes
jenkins_user 'grumpy' do
  full_name    'Grumpy Dwarf'
  email        'grumpy@example.com'
  public_keys  ['ssh-rsa AAAAB3NzaC1y...']
end
```

The `:delete` action removes a Jenkins user from the system.

```ruby
jenkins_user 'grumpy' do
  action :delete
end
```

## Caveats

### Authentication

If you use or plan to use authentication for your Jenkins cluster (which we highly recommend), you will need to set a special value in the `run_context`:

```ruby
node.run_state[:jenkins_private_key]
```

The underlying executor class (which all HWRPs use) intelligently adds authentication information to the Jenkins CLI commands if this value is set. The method used to generate and populate this key-pair is left to the user:

```ruby
# Using search
master = search(:node, 'fqdn:master.ci.example.com').first
node.run_state[:jenkins_private_key] = master['jenkins']['private_key']

# Using encrypted data bags and chef-sugar
private_key = encrypted_data_bag_item('jenkins', 'keys')['private_key']
node.run_state[:jenkins_private_key] = private_key
```

The associated public key must be set on a Jenkins user. You can use the `jenkins_user` resource to create this pairing. Here's an example that loads a keypair and assigns it appropriately:

```ruby
jenkins_keys = encrypted_data_bag_item('jenkins', 'keys')

require 'openssl'
require 'net/ssh'

key = OpenSSL::PKey::RSA.new(jenkins_keys['private_key'])
private_key = key.to_pem
public_key = "#{key.ssh_type} #{[key.to_blob].pack('m0')}"

# Create the Jenkins user with the public key
jenkins_user 'chef' do
  public_keys [public_key]
end

# Set the private key on the Jenkins executor
node.run_state[:jenkins_private_key] = private_key
```

Please note that older versions of Jenkins (< 1.555) permitted login via CLI for a user defined in Jenkins configuration with an SSH public key but not present in the actual SecurityRealm, and this is no longer permitted. If an operation requires any special permission at all, you must authenticate as a real user. This means that if you have LDAP or GitHub OAuth based authn/authz enabled the user you are using for configuration tasks must have an associated account in the external services. Please see [JENKINS-22346](https://issues.jenkins-ci.org/browse/JENKINS-22346) for more details.

If (and **only if**) you have your Jenkins instance configured to use the PAM (Unix user/group database) security realm you can set the username and password the CLI uses via these two `run_context` values:

```ruby
node.run_state[:jenkins_username]
node.run_state[:jenkins_password]
```

### Jenkins 2

Jenkins 2 enables an install wizard by default. To make sure you can manipulate the jenkins instance, you need to disable the wizard. You can do this by setting an attribute:

```ruby
default['jenkins']['master']['jvm_options'] = '-Djenkins.install.runSetupWizard=false'
```

This is done by default, but must be kept when overriding the jvm_options!

### Proxies

If you need to pass through a proxy to communicate between your masters and slaves, you will need to set a special node attribute:

```ruby
node['jenkins']['executor']['proxy']
```

The underlying executor class (which all HWRPs use) intelligently passes proxy information to the Jenkins CLI commands if this attribute is set. It should be set in the form `HOST:PORT`:

```ruby
node.normal['jenkins']['executor']['proxy'] = '1.2.3.4:5678'
```

## Development

Please see the [Contributing](CONTRIBUTING.md) and [Testing](TESTING.md) Guidelines.

## Contributors

This project exists thanks to all the people who [contribute.](https://opencollective.com/sous-chefs/contributors.svg?width=890&button=false)

### Backers

Thank you to all our backers!

![https://opencollective.com/sous-chefs#backers](https://opencollective.com/sous-chefs/backers.svg?width=600&avatarHeight=40)

### Sponsors

Support this project by becoming a sponsor. Your logo will show up here with a link to your website.

![https://opencollective.com/sous-chefs/sponsor/0/website](https://opencollective.com/sous-chefs/sponsor/0/avatar.svg?avatarHeight=100)
![https://opencollective.com/sous-chefs/sponsor/1/website](https://opencollective.com/sous-chefs/sponsor/1/avatar.svg?avatarHeight=100)
![https://opencollective.com/sous-chefs/sponsor/2/website](https://opencollective.com/sous-chefs/sponsor/2/avatar.svg?avatarHeight=100)
![https://opencollective.com/sous-chefs/sponsor/3/website](https://opencollective.com/sous-chefs/sponsor/3/avatar.svg?avatarHeight=100)
![https://opencollective.com/sous-chefs/sponsor/4/website](https://opencollective.com/sous-chefs/sponsor/4/avatar.svg?avatarHeight=100)
![https://opencollective.com/sous-chefs/sponsor/5/website](https://opencollective.com/sous-chefs/sponsor/5/avatar.svg?avatarHeight=100)
![https://opencollective.com/sous-chefs/sponsor/6/website](https://opencollective.com/sous-chefs/sponsor/6/avatar.svg?avatarHeight=100)
![https://opencollective.com/sous-chefs/sponsor/7/website](https://opencollective.com/sous-chefs/sponsor/7/avatar.svg?avatarHeight=100)
![https://opencollective.com/sous-chefs/sponsor/8/website](https://opencollective.com/sous-chefs/sponsor/8/avatar.svg?avatarHeight=100)
![https://opencollective.com/sous-chefs/sponsor/9/website](https://opencollective.com/sous-chefs/sponsor/9/avatar.svg?avatarHeight=100)
